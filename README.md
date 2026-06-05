# Telecom Churn MLOps GitOps Demo

This project demonstrates a minimal end-to-end MLOps workflow for a telecom customer churn prediction model.

It starts from a simple machine learning model and extends it into a containerized, Kubernetes-based, GitOps-driven inference deployment.

---

## What the Model Does

The model predicts whether a telecom customer is likely to churn based on customer attributes such as:

- Age
- Tenure in months
- Monthly charges
- Total charges
- Number of support calls

Example input:

```json
{
  "age": 45,
  "tenure_months": 24,
  "monthly_charges": 79.99,
  "total_charges": 1920.00,
  "num_support_calls": 3
}
```

Example output:

```json
{
  "churn": 1,
  "churn_probability": 0.52
}
```

---

## Architecture

```text
Developer
  ↓ git push
GitHub Repository
  ↓ triggers
GitHub Actions
  ↓ builds and pushes image
Docker Hub
  ↓ pulled by Kubernetes
kind Kubernetes Cluster
  ↑ synced by Argo CD
FastAPI Churn Inference API
```

KServe extension:

```text
KServe InferenceService
  ↓
KServe-created Predictor Deployment
  ↓
FastAPI Churn Model Container
  ↓
Prediction Response
```

---

## Tech Stack

- Python
- pandas
- scikit-learn
- FastAPI
- Uvicorn
- Docker
- Docker Hub
- Kubernetes
- kind
- Argo CD
- GitHub Actions
- Kustomize
- KServe

---

## Repository Structure

```text
.
├── api.py
├── generate_data.py
├── train.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
├── .gitignore
├── README.md
├── k8s/
│   ├── base/
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── kserve/
│       ├── inferenceservice.yaml
│       └── kustomization.yaml
├── argocd/
│   └── application.yaml
└── .github/
    └── workflows/
        └── mlops-ci.yaml
```

---

## Local Development

Create a virtual environment:

```bash
python -m venv .venv
source .venv/Scripts/activate
```

Install dependencies:

```bash
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r requirements.txt
```

Create runtime directories:

```bash
mkdir -p data models
```

Generate synthetic data:

```bash
python generate_data.py
```

Train the model:

```bash
python train.py
```

Run the API locally:

```bash
python -m uvicorn api:app --host 0.0.0.0 --port 8000 --reload
```

Test prediction:

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

---

## Docker

Build the container image:

```bash
docker build -t telecom-churn-mlops-gitops:local .
```

Run the container:

```bash
docker run --rm -p 8000:8000 telecom-churn-mlops-gitops:local
```

Test:

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

---

## CI/CD with GitHub Actions

GitHub Actions performs the following steps:

1. Checks out the repository.
2. Installs Python dependencies.
3. Creates runtime directories.
4. Generates the synthetic churn dataset.
5. Trains the churn model.
6. Validates the generated model artifact.
7. Builds the Docker image.
8. Pushes the image to Docker Hub.

Docker image:

```text
tarek910/telecom-churn-mlops-gitops:latest
```

---

## Kubernetes Deployment

The base Kubernetes manifests are located in:

```text
k8s/base
```

Apply manually:

```bash
kubectl apply -k k8s/base
```

Check resources:

```bash
kubectl get deploy,rs,pod,svc -n churn
```

Expected resources:

```text
deployment.apps/churn-api
service/churn-api
pod/churn-api-xxxxx
```

Test through NodePort:

```bash
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

---

## GitOps with Argo CD

The Argo CD application manifest is located in:

```text
argocd/application.yaml
```

Apply the Argo CD application:

```bash
kubectl apply -f argocd/application.yaml
```

Check status:

```bash
kubectl get application churn-api -n argocd
```

Expected:

```text
NAME        SYNC STATUS   HEALTH STATUS
churn-api   Synced        Healthy
```

The Argo CD application watches:

```text
k8s/base
```

and continuously reconciles the Kubernetes cluster with the desired state stored in Git.

---

## KServe Custom Predictor

This project includes a KServe `InferenceService` that deploys the FastAPI churn model container as a custom predictor.

KServe manifest:

```text
k8s/kserve/inferenceservice.yaml
```

Apply manually:

```bash
kubectl apply -k k8s/kserve
```

Check status:

```bash
kubectl get inferenceservice -n churn
kubectl get deploy,svc,pod -n churn
```

Expected:

```text
churn-predictor   True
```

KServe creates a predictor service similar to:

```text
churn-predictor-predictor
```

Port-forward the KServe-created service:

```bash
kubectl port-forward -n churn svc/churn-predictor-predictor 8090:80
```

Test prediction through KServe:

```bash
curl -X POST http://localhost:8090/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

Example response:

```json
{
  "churn": 1,
  "churn_probability": 0.52
}
```

---

## End-to-End Workflow

```text
Developer pushes code
  ↓
GitHub Actions runs CI/CD
  ↓
Model is trained
  ↓
Docker image is built
  ↓
Image is pushed to Docker Hub
  ↓
Argo CD syncs Kubernetes manifests
  ↓
Kubernetes pulls the Docker image
  ↓
FastAPI inference API runs in the cluster
  ↓
KServe can serve the same image as a custom predictor
```

---

## Validation Commands

Check Argo CD:

```bash
kubectl get application churn-api -n argocd
```

Check Kubernetes resources:

```bash
kubectl get deploy,rs,pod,svc -n churn -o wide
```

Check deployed image:

```bash
kubectl get deployment churn-api -n churn \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Check KServe:

```bash
kubectl get pods -n kserve
kubectl get inferenceservice -n churn
```

Test base Kubernetes service:

```bash
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

Test KServe service:

```bash
curl -X POST http://localhost:8090/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

---

## Current Status

Implemented:

- Synthetic churn data generation
- Churn model training
- FastAPI inference API
- Dockerized model serving
- Docker Hub image publishing
- GitHub Actions CI/CD
- Kubernetes deployment on kind
- Argo CD GitOps deployment
- KServe custom predictor
- Working real-time inference endpoint

Planned improvements:

- DVC for model/data versioning
- Prometheus and Grafana monitoring
- Request/response logging
- Model performance tracking
- Canary or blue-green rollout strategy
- Better production-grade security hardening

---
