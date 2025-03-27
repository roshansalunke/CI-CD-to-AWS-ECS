# Deploying an Application to AWS ECS using Terraform and GitHub Actions

This guide provides step-by-step instructions for deploying a containerized application to AWS ECS using **Terraform** and **GitHub Actions**.

---

## **1. Prerequisites**

Ensure you have the following installed:
- [Terraform](https://www.terraform.io/downloads.html)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Docker](https://www.docker.com/get-started)
- GitHub Repository with GitHub Actions enabled

You'll also need an **AWS Account** with permissions to manage ECS, ECR, and IAM.

---

## **2. Infrastructure Setup using Terraform**

Terraform is used to provision the required AWS resources, including:
- ECS Cluster
- ECS Service
- ECS Task Definition
- ECR Repository
- IAM Roles
- Security Groups
- Load Balancer (optional)

### **2.1 Terraform Configuration**

Create a `main.tf` file with the following content:

```hcl
provider "aws" {
  region = "ap-southeast-1"
}

resource "aws_ecr_repository" "demo_app" {
  name = "demo-app"
}

resource "aws_ecs_cluster" "demo_cluster" {
  name = "demo-cluster"
}

resource "aws_ecs_task_definition" "demo_task" {
  family                   = "demo-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = "arn:aws:iam::325961785170:role/ecsTaskExecutionRole"  # Replace with your actual AWS Account ID

  container_definitions = jsonencode([
    {
      name      = "demo-app",
      image     = "325961785170.dkr.ecr.ap-southeast-1.amazonaws.com/demo-app:latest",
      essential = true,
      portMappings = [{
        containerPort = 4000
      }]
    }
  ])
}

resource "aws_ecs_service" "demo_service" {
  name            = "demo-service"
  cluster         = aws_ecs_cluster.demo_cluster.id
  task_definition = aws_ecs_task_definition.demo_task.arn
  desired_count   = 1
  launch_type     = "FARGATE"
}
```

### **2.2 Initialize Terraform and Apply**

Run the following commands to set up the infrastructure:
```sh
terraform init
terraform apply -auto-approve
```

---

## **3. GitHub Actions CI/CD Pipeline**

### **3.1 Create `.github/workflows/deploy.yml`**

```yaml
name: Deploy to AWS ECS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      - name: Login to AWS ECR
        run: |
          aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com

      - name: Delete Existing Image (if exists)
        run: |
          IMAGE_TAG=latest
          REPO_NAME=demo-app
          aws ecr batch-delete-image --repository-name $REPO_NAME --image-ids imageTag=$IMAGE_TAG || true

      - name: Build & Push Docker Image
        run: |
          docker build -t demo-app .
          docker tag demo-app:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com/demo-app:latest
          docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com/demo-app:latest

      - name: Update ECS Service
        run: |
          aws ecs update-service --cluster demo-cluster --service demo-service --desired-count 1 --force-new-deployment
```

---

## **4. Deployment Process**

### **4.1 Push Code to Main Branch**
Whenever you push changes to the `main` branch, GitHub Actions will:
1. Authenticate with AWS
2. Delete the existing ECR image (if it exists)
3. Build and push a new Docker image
4. Update the ECS service to use the new image

### **4.2 Check Deployment Status**
To check running tasks:
```sh
aws ecs list-tasks --cluster demo-cluster --region ap-southeast-1
```

To check logs:
```sh
aws logs describe-log-groups
aws logs tail /aws/ecs/demo-service --follow
```

### **4.3 Access the Application**
Once deployed, you can access your application using:
```sh
http://<ECS_PUBLIC_IP>:4000
```
If using a Load Balancer, use the ALB URL instead.

---

## **5. Troubleshooting**

### **Issue: Container is still running on port 3000**
- Ensure the `containerPort` in Terraform and the app's `Dockerfile` expose **port 4000**
- Restart the ECS service: `aws ecs update-service --force-new-deployment`

### **Issue: No running tasks after deployment**
- Check the ECS service logs: `aws ecs describe-services --cluster demo-cluster --services demo-service`
- Increase the `desired_count` value in Terraform and redeploy

### **Issue: CannotPullContainerError**
- Ensure the image exists in ECR: `aws ecr describe-images --repository-name demo-app`
- If not, manually push an image: `docker push <ECR-URL>:latest`

---

## **6. Conclusion**
This guide provides a full CI/CD pipeline using **Terraform** and **GitHub Actions** to deploy applications on **AWS ECS**. Now, every push to `main` will trigger an automated deployment!

Let me know if you need any modifications! 🚀

