# 🚀 Deploying a Machine Learning FastAPI App on Amazon SageMaker Using Terraform: Design and Architecture Explained

Machine learning applications are evolving rapidly — and so is the need for scalable, reproducible, and automated deployment workflows. If you’ve ever wanted to serve a custom inference API built with FastAPI on AWS SageMaker, Terraform provides the perfect infrastructure-as-code layer to make it all repeatable and cloud-agnostic.

In this post, we’ll walk through the **design architecture** and **deployment pipeline** for a FastAPI-based inference app hosted on a **SageMaker endpoint**, all managed using **Terraform**.

---

## 🧩 High-Level Architecture

At its core, the architecture consists of **three major components**:

1. **FastAPI Inference App**  
   The custom inference logic — wrapped inside a FastAPI application (`app.py`) — is responsible for handling prediction requests, performing lightweight pre/post-processing, and returning results in real-time.

2. **Amazon SageMaker Model & Endpoint**  
   SageMaker hosts the model and inference container as a scalable endpoint. It abstracts the complexities of provisioning compute, load balancing, and scaling the service under traffic.

3. **Terraform Infrastructure-as-Code**  
   Terraform provisions and configures all necessary AWS resources — including IAM roles, ECR repositories, SageMaker models, and endpoints — ensuring consistent and repeatable deployments across environments.

---


## ⚙️ The FastAPI Inference Application

At the heart of this deployment lies the **FastAPI app**, which defines a RESTful API interface for serving predictions. The app can be containerized and pushed to **Amazon ECR (Elastic Container Registry)** for SageMaker to use.

A simplified version of the `app.py` file looks like this:

```python
from fastapi import FastAPI, Request
import joblib

app = FastAPI(title="SageMaker FastAPI Endpoint")

# Load your trained model (bundled with the container)
model = joblib.load("/opt/ml/model/model.joblib")

@app.get("/")
def root():
    return {"message": "SageMaker FastAPI is live!"}

@app.post("/predict")
async def predict(request: Request):
    payload = await request.json()
    features = payload.get("features")
    prediction = model.predict([features]).tolist()
    return {"prediction": prediction}
```


This app defines:
1. A GET endpoint for health checks.
2. A POST endpoint (/predict) that handles real-time inference requests.

Once containerized with a simple Dockerfile, it’s uploaded to ECR, where Terraform will reference it in the SageMaker model configuration.

---

## 🧱 Infrastructure Deployment with Terraform

The main.tf file defines the infrastructure resources that bring everything to life.
At a high level, it does the following:
1. Create IAM Roles for SageMaker execution (permissions to pull the container and run inference).
2. Define the SageMaker Model, referencing the ECR image URL and model artifacts.
3. Deploy the Endpoint Configuration, specifying instance type, scaling, and environment variables.
4. Launch the SageMaker Endpoint that serves the FastAPI app in real time.

A simplified main.tf structure looks like this:

```terraform 
provider "aws" {
  region = "us-east-1"
}

resource "aws_iam_role" "sagemaker_role" {
  name = "sagemaker_execution_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Principal = {
          Service = "sagemaker.amazonaws.com"
        },
        Effect = "Allow"
      }
    ]
  })
}

resource "aws_sagemaker_model" "fastapi_model" {
  name              = "fastapi-inference-model"
  execution_role_arn = aws_iam_role.sagemaker_role.arn

  primary_container {
    image = "<your-ecr-repo-url>:latest"
    mode  = "SingleModel"
  }
}

resource "aws_sagemaker_endpoint_configuration" "fastapi_config" {
  name = "fastapi-endpoint-config"

  production_variants {
    variant_name           = "AllTraffic"
    model_name             = aws_sagemaker_model.fastapi_model.name
    initial_instance_count = 1
    instance_type          = "ml.m5.large"
  }
}

resource "aws_sagemaker_endpoint" "fastapi_endpoint" {
  name                 = "fastapi-endpoint"
  endpoint_config_name = aws_sagemaker_endpoint_configuration.fastapi_config.name
}

```
Terraform will create a fully managed SageMaker endpoint running your FastAPI inference service.

## 🧩 Putting It All Together

Build and push your FastAPI Docker image to Amazon ECR:
```bash
docker build -t fastapi-sagemaker .
docker tag fastapi-sagemaker:latest <your-ecr-repo-url>:latest
docker push <your-ecr-repo-url>:latest
```
Run Terraform to provision AWS resources and deploy the endpoint.
Test the endpoint using a simple curl or requests call:
```python
import requests

payload = {"features": [1.2, 3.4, 5.6]}
response = requests.post("<SAGEMAKER-ENDPOINT-URL>/invocations", json=payload)
print(response.json())
```


## 🔒 Security and Scaling Considerations
1. IAM Roles & Policies — Keep least privilege principles when granting SageMaker permissions.
2. VPC Integration — For sensitive workloads, run the endpoint inside a private VPC.
3. Auto-scaling — Configure dynamic endpoint scaling policies for production environments.
4. Monitoring — Use CloudWatch for logs and performance metrics.


## 🧠 Key Takeaways
1. Terraform gives you declarative, repeatable deployment of SageMaker resources.
2. FastAPI provides a flexible inference API that integrates easily with containerized workflows.
3. SageMaker manages scaling, availability, and hosting — so you can focus on the model and API logic.

## 📚 Conclusion

By combining Terraform, FastAPI, and SageMaker, you get a modern, infrastructure-as-code deployment stack that’s scalable, reproducible, and production-ready. Whether you’re serving a deep learning model or a lightweight ML classifier, this approach provides a clean, automated pathway from development to deployment.

[Medium: Deploying a Machine Learning FastAPI app on Amazon Sagemaker Endpoint](https://medium.com/@echraibi.amine/deploying-a-fastapi-app-on-amazon-sagemaker-using-terraform-design-and-architecture-explained-13be5b3ed15b)

[Youtube: Deploying a Machine Learning FastAPI app on Amazon Sagemaker Endpoint](https://www.youtube.com/watch?v=KFqehCAMaLQ)