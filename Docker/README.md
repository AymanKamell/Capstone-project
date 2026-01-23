# FinTech API – Docker Build & ECR Deployment Guide

This directory contains the source code and Docker configuration for the **FinTech API** service.
The application is a Python Flask API designed to run on **Amazon ECS (Fargate)** and integrate with
AWS managed services such as **RDS (PostgreSQL)**, **ElastiCache Redis**, **DynamoDB**, and **Secrets Manager**.

---

## 📁 Project Structure

```

my-app-v3/
├── app.py              # Flask application
├── requirements.txt    # Python dependencies
└── Dockerfile          # Docker image definition

````

---

## 🧠 Application Overview

The API exposes the following endpoints:

- `GET /health`  
  Basic health check endpoint.

- `GET /test-all`  
  Validates connectivity to:
  - Amazon RDS (PostgreSQL)
  - Amazon ElastiCache (Redis)
  - Amazon DynamoDB
  - AWS Secrets Manager

The application retrieves all sensitive credentials from **AWS Secrets Manager** and does not store
any secrets in code or environment variables.

---

## ✅ Prerequisites

Before building and pushing the image, ensure you have:

- Docker installed and running
- AWS CLI v2 installed
- An AWS account with:
  - Amazon ECR repository created
  - IAM permissions for ECR (`AmazonEC2ContainerRegistryFullAccess`)
- Logged in to the correct AWS account and region

Verify AWS CLI access:

```bash
aws sts get-caller-identity
````

---

## 🐳 Step 1: Build the Docker Image Locally

From inside the `my-app-v3` directory:

```bash
docker build -t fintech-api:latest .
```

Verify the image was built:

```bash
docker images | grep fintech-api
```

---

## 🔐 Step 2: Authenticate Docker to Amazon ECR

Set your AWS region and account ID:

```bash
AWS_REGION=us-east-1
ACCOUNT_ID=296674251987
```

Authenticate Docker to ECR:

```bash
aws ecr get-login-password --region $AWS_REGION \
  | docker login \
    --username AWS \
    --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

You should see:

```
Login Succeeded
```

---

## 🏷️ Step 3: Tag the Docker Image for ECR

Tag the local image with the ECR repository URI:

```bash
docker tag fintech-api:latest \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/fintech-api:latest
```

---

## 🚀 Step 4: Push the Image to Amazon ECR

Push the image to ECR:

```bash
docker push \
  $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/fintech-api:latest
```

Once completed, the image will be available for use by **ECS task definitions**.

---

## 📦 Dockerfile Notes

Key characteristics of the Dockerfile:

* Uses `python:3.11-slim` for a small image footprint
* Runs the application as a **non-root user** (security best practice)
* Installs only required system libraries (`libpq-dev`, `ca-certificates`)
* Exposes port `8080` for ECS / ALB integration

---

## 🔄 Next Steps (After Image Push)

After pushing the image to ECR:

1. Update the **ECS Task Definition** to reference the new image
2. Deploy a new task revision
3. ECS will pull the image automatically from ECR
4. Traffic will be routed via ALB / CloudFront

---

## 🔐 Security Best Practices

* No secrets are stored in the image or repository
* All credentials are fetched dynamically from AWS Secrets Manager
* The container runs as a non-root user
* Network access is restricted via Security Groups

---

## 🧪 Optional Local Testing

You can run the container locally (limited functionality without AWS credentials):

```bash
docker run -p 8080:8080 fintech-api:latest
```

Just tell me 👌
```
