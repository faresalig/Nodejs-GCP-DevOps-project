<img src="https://img.shields.io/github/forks/tush-tr/DevOps-Projects"> <img src="https://img.shields.io/github/license/tush-tr/DevOps-Projects"> <img src="https://img.shields.io/github/stars/tush-tr/DevOps-Projects"> <a href="https://twitter.com/tush_tr604" target="blank"><img src="https://img.shields.io/twitter/follow/tush_tr604?logo=twitter&style=flat" alt="tush_tr604" /></a>

# Node.js DevOps Project with GCP, Kubernetes & Terraform

A complete CI/CD pipeline demonstrating modern DevOps practices using Node.js application deployment to Google Kubernetes Engine (GKE) with Infrastructure as Code.

## 🚀 Features

- **Node.js Express Application** - Simple web application containerized with Docker
- **Infrastructure as Code** - GKE cluster provisioning using Terraform
- **CI/CD Pipeline** - Automated deployment with GitHub Actions
- **Container Registry** - Docker images stored in Google Artifact Registry
- **Kubernetes Orchestration** - Application deployment with LoadBalancer service
- **OIDC Authentication** - Secure GitHub-to-GCP authentication without service account keys

## 🛠️ Tech Stack

- **Runtime**: Node.js, Express
- **Containerization**: Docker
- **Orchestration**: Kubernetes (GKE)
- **Infrastructure**: Terraform
- **CI/CD**: GitHub Actions
- **Cloud Provider**: Google Cloud Platform
- **Container Registry**: Google Artifact Registry

## 📁 Project Structure

```
├── nodeapp/           # Node.js application source code
├── terraform/         # Terraform infrastructure files
├── .github/workflows/ # GitHub Actions CI/CD pipeline
└── README.md         # Project documentation
```

## 🔧 Key Components

- **GKE Cluster**: Managed Kubernetes cluster on Google Cloud
- **Terraform Modules**: Infrastructure provisioning and management
- **GitHub OIDC**: Keyless authentication for secure deployments
- **LoadBalancer Service**: External access to the application
- **Automated Pipeline**: Build, test, and deploy on every push

This project demonstrates enterprise-grade DevOps practices with cloud-native technologies and GitOps workflows.

## Setup Instructions

- [x] Setup Github OIDC Authentication with GCP
  - Create a new workload Identity pool
    ```sh
    gcloud iam workload-identity-pools create "k8s-pool" \
    --project="${PROJECT_ID}" \
    --location="global" \
    --display-name="k8s Pool"
    ```
  - Create a oidc identity provider for authenticating with Github
    ```sh
    gcloud iam workload-identity-pools providers create-oidc k8s-provider \
  --project= ${GCP_PROJECT_ID} \
  --location=global \
  --workload-identity-pool=k8s-pool \
  --display-name="GitHub Actions Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='faresalig/Nodejs-GCP-DevOps-project'"
    ```
  - Create a service account with these permissions
    ```sh
    roles/compute.admin
    roles/container.admin
    roles/container.clusterAdmin
    roles/iam.serviceAccountTokenCreator
    roles/iam.serviceAccountUser
    roles/storage.admin
    roles/artifactregistry.writer
    ```
  - Add IAM Policy bindings with Github repo, Identity provider and service account.
    ```sh
    gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT_EMAIL}" \
    --project="${GCP_PROJECT_ID}" \
    --role="roles/iam.workloadIdentityUser" \
    --member="principalSet://iam.googleapis.com/projects/${GCP_PROJECT_NUMBER}/locations/global/workloadIdentityPools/k8s-pool/attribute.repository/${GITHUB_REPO}"
    ```

Enable Authenticate to Google Cloud

gcloud services enable iamcredentials.googleapis.com \
  --project="${GCP_PROJECT_ID}"

Enabel Authenticate to Google Artifactregistry

gcloud services enable artifactregistry.googleapis.com \
  --project="${GCP_PROJECT_ID}"

- [x] Create a bucket in GCS for storing terraform state file.
- [x] Get your GCP Project number for reference.
  ```sh
  gcloud projects describe ${PROJECT_ID}
  ``` 
- [x] Add secrets to Github Repo
  - GCP_PROJECT_ID
  - GCP_TF_STATE_BUCKET
- [x] write GH Actions workflow for deploying our app to GKE using terraform

```yml
name: Deploy to kubernetes
on:
  push:
    branches:
      - "Complete-CI/CD-with-Terraform-GKE"

env:
  GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  TF_STATE_BUCKET_NAME: ${{ secrets.GCP_TF_STATE_BUCKET }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      IMAGE_TAG: ${{ github.sha }}

    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
    - uses: 'actions/checkout@v3'
    - id: 'auth'
      name: 'Authenticate to Google Cloud'
      uses: 'google-github-actions/auth@v1'
      with:
        token_format: 'access_token'
        workload_identity_provider: 'projects/886257991781/locations/global/workloadIdentityPools/k8s-pool/providers/k8s-provider'
        service_account: 'tf-gke-test@$GCP_PROJECT_ID.iam.gserviceaccount.com'
    - name: 'Set up Cloud SDK'
      uses: 'google-github-actions/setup-gcloud@v1'
    - name: docker auth
      run: gcloud auth configure-docker
    - run: gcloud auth list
    - name: Build and push docker image
      run: |
        docker build -t us.gcr.io/$GCP_PROJECT_ID/nodeappimage:$IMAGE_TAG .
        docker push us.gcr.io/$GCP_PROJECT_ID/nodeappimage:$IMAGE_TAG
      working-directory: ./nodeapp
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
    - name: Terraform init
      run: terraform init -backend-config="bucket=$TF_STATE_BUCKET_NAME" -backend-config="prefix=test"
      working-directory: ./terraform
    - name: Terraform Plan
      run: |
        terraform plan \
        -var="region=us-central1" \
        -var="project_id=$GCP_PROJECT_ID" \
        -var="container_image=us.gcr.io/$GCP_PROJECT_ID/nodeappimage:$IMAGE_TAG" \
        -out=PLAN
      working-directory: ./terraform
    - name: Terraform Apply
      run: terraform apply PLAN
      working-directory: ./terraform
```