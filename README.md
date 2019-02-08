# Google Cloud Native Spring Boot PetClinic

Example Petclinic deployment on Google Cloud Platform into Google Kubernetes Engine with Istio.
This is based on [Spring PetClinic Microservices](https://github.com/spring-petclinic/spring-petclinic-microservices)

This example has:
 - Observability and Monitoring
   - Stackdriver Trace
   - Stackdriver Monitorning
   - Stackdriver Logging
   - Stackdriver Debugging
   - Stackdriver Profiling
 - Spring Boot Petclinic Example with Google Cloud Native configuration
   - Spring Cloud GCP
   - Removed Eureka, Hystrix, Ribbon, Config Server, Gateway, and many other components, because they are provided by Kubernetes and Istio.
     - Eureka -> Kubernetes Service
     - Config Server -> Kubernetes Config Map
     - Gateway -> Kubernetes Ingress
     - Hystrix -> Istio
     - Ribbon -> Istio
 - Build
   - Spotify's dockerfile-maven-plugin
 - DevOps
   - Travis CI

## Google Cloud Platform Project
Create a new Project if you haven't done so already.
```
$ export PROJECT_ID=...
$ gcloud project create $PROJECT_ID
```

Set the default Project ID:
```
$ gcloud config set core/project $PROJECT_ID
```

## Kubernetes Engine Cluster
Use `gcloud` to provision a multi-zone Kubernetes Engine cluster.

```
$ gcloud services enable compute.googleapis.com container.googleapis.com
$ CLOUDSDK_CONTAINER_USE_V1_API_CLIENT=false
$ gcloud container clusters create petclinic-cluster \
    --cluster-version=1.9.6 \
    --region=us-central1 \
    --num-nodes=2 \
    --machine-type=n1-standard-2 \
    --enable-autorepair \
    --no-enable-cloud-logging \
    --no-enable-cloud-monitoring
```

## Istio
Install the basics:
```
$ ISTIO_VERSION=0.7.1
$ curl -L https://git.io/getLatestIstio | sh -
$ cd istio-$ISTIO_VERSION
$ kubectl apply -f install/kubernetes/istio.yaml --as=admin --as-group=system:masters
```

Update Sidecar Injector to limit Istio to 10.0.0.0/8 network:
1. Open `install/kubernetes/istio-sidecar-injector-configmap-release.yaml`
1. Update the `initContainers` `args` block:
```
...
      initContainers:
      - name: istio-init
        image: docker.io/istio/proxy_init:0.7.1
        args:
        - "-p"
        - {{ .MeshConfig.ProxyListenPort }}
        - "-u"
        - 1337
        # ADD THE FOLLOWING LINES
        - -i
        - 10.0.0.0/8
        # ADD THE ABOVE LINES
...
```

Install Sidecar Injector:
```
$ install/kubernetes/webhook-create-signed-cert.sh \
    --service istio-sidecar-injector \
    --namespace istio-system \
    --secret sidecar-injector-certs
$ kubectl apply -f install/kubernetes/istio-sidecar-injector-configmap-release.yaml
$ cat install/kubernetes/istio-sidecar-injector.yaml | \
     ./install/kubernetes/webhook-patch-ca-bundle.sh > \
     install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
$ kubectl apply -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

Enable Sidecar Injector on `default` namespace:
```
$ kubectl label namespace default istio-injection=enabled
```

## Spanner
```
$ gcloud spanner instances create petclinic --config=regional-us-central1 --nodes=1 --description="PetClinic Spanner Instance"
$ gcloud spanner databases create petclinic --instance=petclinic
$ gcloud spanner databases ddl update petclinic --instance=petclinic --ddl="$(<petclinic/db/spanner.ddl)"
```

## Debugging and Profiling
```
$ gcloud services enable cloudprofiler.googleapis.com clouddebugger.googleapis.com
```

## Generate Service Account
Create a new Service Account for the microservices:
```
$ gcloud iam service-accounts create petclinic --display-name "PetClinic Service Account"
```

Grant IAM Roles to the Service Account:
```
$ gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member serviceAccount:petclinic@$PROJECT_ID.iam.gserviceaccount.com \
     --role roles/cloudprofiler.agent
$ gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member serviceAccount:petclinic@$PROJECT_ID.iam.gserviceaccount.com \
     --role roles/clouddebugger.agent
$ gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member serviceAccount:petclinic@$PROJECT_ID.iam.gserviceaccount.com \
     --role roles/cloudtrace.agent
$ gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member serviceAccount:petclinic@$PROJECT_ID.iam.gserviceaccount.com \
     --role roles/spanner.databaseUser
```

Create a new JSON Service Account Key. Keep it secure!
```
$ gcloud iam service-accounts keys create ~/petclinic-service-account.json \
    --iam-account petclinic@$PROJECT_ID.iam.gserviceaccount.com
```

## Build
### Compile and Install to Maven
```
$ mvn install
```

### Build Docker Images
Build all images:
```
$ mvn package install -PbuildDocker
```

Build just one image:
```
$ mvn package install -PbuildDocker -pl spring-petclinic-customers-service
```

## Run
### Docker Compose
Update `docker-compose.yml` file so that `secrets.petclinic-credentials.file`
points to the JSON file.

Run everything:
```
$ echo "PROJECT_ID=$PROJECT_ID" > .env
$ docker-compose up
```

### Kubernetes
Store Service Account as a Kubenetes Secret:
```
$ kubectl create secret generic petclinic-credentials --from-file=$HOME/petclinic-service-account.json
```

Deploy Application:
```
$ kubectl apply -f kubernetes/
```

Deploy Route Rules:
```
$ kubectl apply -f istio/
```

### Travis CI/CD
Install the Travis CLI:
```
$ brew install travis
```
Or, follow the [Travis CLI Installation instruction](https://github.com/travis-ci/travis.rb#installation)

Login to Travis
```
$ travis login
```
Or, optionally login with `travis login --github-token=...` to avoid typing password, etc.

Configure Docker credentials:
```
$ travis env set DOCKER_USERNAME your_username
$ travis env set DOCKER_PASSWORD your_password
```

Create a CI/CD Service Account, assign roles, and create a JSON file:
```
$ gcloud iam service-accounts create travis-ci --display-name "Travis CI/CD"
$ gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member serviceAccount:travis-ci@$PROJECT_ID.iam.gserviceaccount.com \
     --role roles/container.developer
$ gcloud iam service-accounts keys create ~/travis-ci-petclinic.json \
    --iam-account travis-ci@$PROJECT_ID.iam.gserviceaccount.com
```

Encrypt and Store the Travis CI/CD Service Account:
```
$ travis encrypt-file ~/travis-ci-petclinic.json
```
Travis asks you to add a line to `before_install` section. Make sure it's updated.

Set the Google Cloud Platform Project ID for reference in the build:
```
$ travis env set PROJECT_ID $PROJECT_ID
```

Commit `.travis.yml`

## Generating/updating the separate microservices

Use the following command from the CLI or in your CI/CD pipeline:

    jx step split monorepo -o petclinic-gcp  --glob "spring-*"