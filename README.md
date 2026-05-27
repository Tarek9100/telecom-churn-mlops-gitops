# Telecom Churn MLOps GitOps Demo

This project demonstrates a minimal end-to-end MLOps workflow for a telecom customer churn prediction model.

It starts from a simple machine learning model and extends it into a containerized, Kubernetes-based, GitOps-driven inference deployment.

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

## Architecture

```text
Developer
  ↓ git push
GitHub Repository
  ↓ triggers
GitHub Actions
  ↓ builds, trains, and pushes image
Docker Hub
  ↓ pulled by Kubernetes
kind Kubernetes Cluster
  ↑ synced by Argo CD
FastAPI Churn Inference API
```

## Current Implemented Workflow

The current workflow is:

1. Code is pushed to GitHub.
2. GitHub Actions installs dependencies.
3. Synthetic churn data is generated.
4. The churn model is trained.
5. The model artifact is validated.
6. A Docker image is built.
7. The image is pushed to Docker Hub.
8. Argo CD syncs the Kubernetes manifests from Git.
9. Kubernetes pulls the image from Docker Hub.
10. The FastAPI inference API becomes available through a NodePort service.

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
│   └── base/
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
├── argocd/
│   └── application.yaml
└── .github/
    └── workflows/
        └── mlops-ci.yaml
```

## Local Development

Create and activate a Python virtual environment:

```bash
python -m venv .venv
source .venv/Scripts/activate
```

Install dependencies:

```bash
python -m pip install --upgrade pip setuptools wheel
python -m pip install -r requirements.txt
```

Generate the synthetic dataset and train the model:

```bash
mkdir -p data models
python generate_data.py
python train.py
```

Run the API locally:

```bash
python -m uvicorn api:app --host 0.0.0.0 --port 8000 --reload
```

Test the prediction endpoint:

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

Expected response format:

```json
{
  "churn": 1,
  "churn_probability": 0.52
}
```

The exact probability may vary because the model is trained from generated synthetic data.

## Docker

Build the Docker image locally:

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

## Kubernetes Deployment

The Kubernetes manifests are located in:

```text
k8s/base
```

Apply manually:

```bash
kubectl apply -k k8s/base
```

Check the workload:

```bash
kubectl get deploy,rs,pod,svc -n churn
```

Test through the kind NodePort mapping:

```bash
curl -X POST http://localhost:8080/predict \
  -H "Content-Type: application/json" \
  -d '{"age":45,"tenure_months":24,"monthly_charges":79.99,"total_charges":1920,"num_support_calls":3}'
```

## GitOps with Argo CD

The Argo CD application manifest is located in:

```text
argocd/application.yaml
```

Apply the Argo CD application:

```bash
kubectl apply -f argocd/application.yaml
```

Check application status:

```bash
kubectl get application churn-api -n argocd
```

Expected state:

```text
SYNC STATUS: Synced
HEALTH STATUS: Healthy
```

## CI/CD Pipeline

GitHub Actions performs the following steps:

1. Checkout source code.
2. Set up Python 3.11.
3. Install Python dependencies.
4. Create runtime directories.
5. Generate synthetic churn data.
6. Train the model.
7. Validate the model artifact.
8. Log in to Docker Hub.
9. Build the Docker image.
10. Push the Docker image to Docker Hub.

Docker image:

```text
tarek910/telecom-churn-mlops-gitops:latest
```

## Docker Hub

The Kubernetes Deployment pulls the inference image from Docker Hub:

```text
tarek910/telecom-churn-mlops-gitops:latest
```

## API Usage

Endpoint:

```text
POST /predict
```

Request:

```json
{
  "age": 45,
  "tenure_months": 24,
  "monthly_charges": 79.99,
  "total_charges": 1920.00,
  "num_support_calls": 3
}
```

Response:

```json
{
  "churn": 1,
  "churn_probability": 0.52
}
```

## Current Status

Implemented:

- Synthetic churn data generation
- Model training with scikit-learn
- FastAPI inference service
- Dockerized model server
- Kubernetes Deployment and NodePort Service
- GitHub Actions CI/CD pipeline
- Docker Hub image publishing
- Argo CD GitOps deployment
- Real-time prediction API exposed from a kind Kubernetes cluster

Planned improvements:

- KServe custom predictor deployment
- DVC-based model/data versioning
- Model metrics tracking
- Prometheus and Grafana monitoring
- Canary or blue-green rollout strategy
- Better API health endpoints
- Automated image tag updates instead of relying only on `latest`

## Notes

This project started from a simple churn prediction demo and was extended into a local MLOps platform using Docker, Kubernetes, GitHub Actions, Docker Hub, and Argo CD.

The purpose of this project is to demonstrate practical MLOps and platform engineering concepts in a reproducible local environment.
