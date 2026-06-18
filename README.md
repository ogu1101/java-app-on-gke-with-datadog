# Example Application with Datadog

![Architecture](doc/architecture-gke.drawio.png)

## Table of Contents

- [Overview](#overview)
- [About the Application](#about-the-application)
- [Enabled Datadog Features](#enabled-datadog-features)
- [Build and Run (Google Cloud)](#build-and-run-google-cloud)
  - [Prerequisites](#prerequisites-google-cloud)
  - [Pre-work](#pre-work-google-cloud)
  - [Edit terraform/terraform.tfvars](#edit-terraformterraformtfvars)
  - [Enable Google Cloud APIs](#enable-google-cloud-apis)
  - [Create Google Cloud Resources](#create-google-cloud-resources)
  - [Update Commands and Files](#update-commands-and-files)
  - [Build and Push the Application Container Image](#build-and-push-the-application-container-image)
  - [Edit k8s/manifests.yaml](#edit-k8smanifestsyaml)
  - [Deploy Kubernetes Resources](#deploy-kubernetes-resources)
  - [Send an HTTP Request](#send-an-http-request-google-cloud)
  - [Delete Kubernetes Resources](#delete-kubernetes-resources)
  - [Delete Google Cloud Resources](#delete-google-cloud-resources)
- [Build and Run (Local)](#build-and-run-local)
  - [Prerequisites](#prerequisites-local)
  - [Pre-work](#pre-work-local)
  - [Start Containers](#start-containers)
  - [Send an HTTP Request](#send-an-http-request-local)
  - [Jenkins](#jenkins)
  - [Stop Containers](#stop-containers)
- [References](#references)

---

## Overview

Clone this repository and run the `terraform` and `kubectl` commands described below to provision the Google Cloud resources shown in the architecture diagram above, then deploy the Java application container and Datadog Agent container to GKE.

For the full command sequence, see [Build and Run (Google Cloud)](#build-and-run-google-cloud).

You can also run the following containers locally using `docker compose`:

- Java application (web service) container
- Datadog Agent container
- PostgreSQL container
- Jenkins container

For the full command sequence, see [Build and Run (Local)](#build-and-run-local).

---

## About the Application

- Uses **Spring Boot** as the web framework.
- Persists the contents of incoming HTTP requests to PostgreSQL.
- Outputs logs in JSON format so they can be parsed by Datadog.
- Unit tests can be run with `mvn test`.

---

## Enabled Datadog Features

All features below are enabled automatically with no manual intervention, **except CI Visibility**.

To enable CI Visibility, you must manually [install the Datadog plugin for Jenkins](https://docs.datadoghq.com/ja/continuous_integration/pipelines/jenkins/?tab=linux#datadog-jenkins-%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB) and create a Jenkins job.

| Feature | Notes |
|---|---|
| Live Processes | |
| Application Performance Monitoring | |
| Continuous Profiler | |
| Log Management | Connected to traces |
| Application Security Management | |
| CI Visibility | Local only |
| Database Monitoring | Connected to traces; local only |
| Network Performance Monitoring | Google Cloud only |
| Universal Service Monitoring | Google Cloud only |

---

## Build and Run (Google Cloud)

### Prerequisites (Google Cloud)

- Install **Docker** by following the [official documentation](https://docs.docker.com/engine/install/).
- To build multi-platform container images, enable **`Use containerd for pulling and storing images`** in Docker Desktop by following [this guide](https://docs.docker.com/desktop/containerd/#turn-on-the-containerd-image-store-feature).
- Install **Helm** by following the [official documentation](https://helm.sh/).
- Install **Terraform** by following the [official documentation](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli).
- Install **`gke-gcloud-auth-plugin`** by following [this guide](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl?hl=ja#install_plugin).
- For all other prerequisites, refer to [this Terraform GKE tutorial](https://developer.hashicorp.com/terraform/tutorials/kubernetes/gke?utm_medium=WEB_IO&in=terraform%2Fkubernetes&utm_content=DOCS&utm_source=WEBSITE&utm_offer=ARTICLE_PAGE#prerequisites).

### Pre-work (Google Cloud)

Set your Datadog API key as the value of `DD_API_KEY` in the `.env` file.

### Edit `terraform/terraform.tfvars`

Update `terraform/terraform.tfvars` as follows:

- Set `project_id` to your Google Cloud project ID.
- Set `env` to an arbitrary value to avoid naming conflicts with existing Google Cloud resources.
- Look up your global IP address (e.g., via [this site](https://www.cman.jp/network/support/go_access.cgi)) and set `your_global_ip_address` accordingly.

### Enable Google Cloud APIs

Run the following commands from any directory:

```bash
gcloud services enable artifactregistry.googleapis.com

gcloud services enable compute.googleapis.com

gcloud services enable container.googleapis.com

gcloud services enable sqladmin.googleapis.com
```

### Create Google Cloud Resources

Run the following commands from the `terraform` directory:

```bash
terraform init

terraform apply
```

### Update Commands and Files

Replace the placeholder variables in the commands and files below:

- Replace `${PROJECT_ID}` with the value of `project_id` in `terraform/terraform.tfvars`.
- Replace `${REGION}` with the value of `region` in `terraform/terraform.tfvars`.
- Replace `${ENV}` with the value of `env` in `terraform/terraform.tfvars`.

For example commands with these values substituted, see `example-command.sh`.

### Build and Push the Application Container Image

Run the following commands from the directory containing the `Dockerfile`:

```bash
gcloud auth configure-docker ${REGION}-docker.pkg.dev

docker buildx build . -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${ENV}-repository/java-app-on-gke-with-datadog:latest --platform linux/amd64,linux/arm64 --build-arg DD_GIT_REPOSITORY_URL=$(git config --get remote.origin.url) --build-arg DD_GIT_COMMIT_SHA=$(git rev-parse HEAD)

docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${ENV}-repository/java-app-on-gke-with-datadog:latest
```

### Edit `k8s/manifests.yaml`

Update `k8s/manifests.yaml` as follows. Lines that require changes are marked with a `# REPLACE ME` comment.

- Replace `image: us-central1-docker.pkg.dev/tribal-iridium-308123/shuhei-repository/java-app-on-gke-with-datadog:latest` with `image: ${REGION}-docker.pkg.dev/${PROJECT_ID}/${ENV}-repository/java-app-on-gke-with-datadog:latest`.
- Replace - `tribal-iridium-308123:us-central1:shuhei-cloud-sql` with - `${PROJECT_ID}:${REGION}:${ENV}-cloud-sql`.
- Replace `kubernetes.io/ingress.global-static-ip-name: shuhei-ip-address` with `kubernetes.io/ingress.global-static-ip-name: ${ENV}-ip-address`.

### Deploy Kubernetes Resources

Replace `${API-KEY}` in the commands below with your Datadog API key.

Run the following commands from the `k8s` directory:

```bash
gcloud container clusters get-credentials --zone ${REGION} ${ENV}-gke

helm repo add datadog https://helm.datadoghq.com

helm install datadog-operator datadog/datadog-operator

kubectl create secret generic datadog-secret --from-literal api-key=${API-KEY}

kubectl apply -f datadog-agent.yaml -f service-account.yaml

kubectl annotate serviceaccount ksa-cloud-sql iam.gke.io/gcp-service-account=${ENV}-service-account-id@${PROJECT_ID}.iam.gserviceaccount.com

kubectl apply -f manifests.yaml
```

### Send an HTTP Request (Google Cloud)

Run kubectl get service app to find the external IP address:

```bash
shuhei.ogura@COMP-R7QQCTJ177 k8s % kubectl get service app
NAME   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
app    LoadBalancer   10.187.247.149   35.238.101.70   8080:31303/TCP   38h
```

Replace `${EXTERNAL-IP}` with the `EXTERNAL-IP` value shown above, then run:

```bash
curl -v -X POST -H 'Content-Type:application/json' -d '{"message":"Hello", "target":"Kagetaka"}' ${EXTERNAL-IP}:8080/greeting
```

### Delete Kubernetes Resources

Run the following commands from the `k8s` directory:

```bash
kubectl delete -f manifests.yaml -f service-account.yaml

kubectl delete datadogagent datadog

helm delete datadog-operator
```

### Delete Google Cloud Resources

Run the following command from the `terraform` directory:

```bash
terraform destroy
```

---
## Build and Run (Local)

### Prerequisites (Local)

- Install Docker by following the official documentation (https://docs.docker.com/engine/install/).
- The local Datadog Agent container is designed to run correctly on macOS only.

### Pre-work (Local)

Set your Datadog API key as the value of `DD_API_KEY` in the `.env` file.

### Start Containers

Run the following command from the directory containing `compose.yaml`:

```bash
docker compose up -d --build
```

### Send an HTTP Request (Local)

To send an HTTP request to the application container:

```bash
curl -v -X POST -H 'Content-Type:application/json' -d '{"message":"Hello", "target":"Kagetaka"}' 127.0.0.1:8080/greeting
```

### Jenkins

Jenkins is accessible at http://localhost:8888.

The username and password are printed to the Jenkins container logs on first startup.

### Stop Containers

Run the following command from the directory containing `compose.yaml`:

```bash
docker compose down
```

---
## References

- [Spring Initializr](https://start.spring.io/)
- [Provision a GKE Cluster (Google Cloud) — Terraform Tutorial](https://developer.hashicorp.com/terraform/tutorials/kubernetes/gke)
- [Connecting to Cloud SQL from Google Kubernetes Engine](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine?hl=ja)
- [Configure Networking for a Basic Production Cluster](https://cloud.google.com/kubernetes-engine/docs/tutorials/configure-networking?hl=ja)
- [Terraform Registry — Google Cloud Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Enable or Disable High Availability for Cloud SQL](https://cloud.google.com/sql/docs/postgres/configure-ha?hl=ja#terraform)
