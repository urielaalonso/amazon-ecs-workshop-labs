# Lab 15: Creating a Podman "Hello World" on Amazon ECS with Amazon ECR

## Objective

In this lab, you will learn how to create a simple "Hello World" container using Podman, push it to Amazon Elastic Container Registry (ECR), and deploy it using Amazon Elastic Container Service (ECS).

## Prerequisites

Ensure you have the following prerequisites in place before starting the lab:

- AWS CLI is configured with the necessary permissions.
- Podman is installed on your Ubuntu system.
- AWS account ready to use.

## Steps

### Step 1: Create a Simple "Hello World" Podman Application

Create a new directory for the project:

```bash
mkdir hello-world-podman
cd hello-world-podman
```

Create a `Dockerfile` in the project directory:

```dockerfile
FROM python:3.8-slim

WORKDIR /app

COPY hello.py /app/hello.py

CMD ["python", "hello.py"]
```

Create a simple Python script named `hello.py`:

```python
print("Hello, World!")
```

### Step 2: Build the Podman Image

Create a variable for your container repository name to minimize conflicts:

```bash
PODMAN_REPO="hello-world-app-<yourname>"
export PODMAN_REPO
echo $PODMAN_REPO
```

Build the Podman image:

```bash
podman build -t $PODMAN_REPO .
```

Verify the Podman image:

```bash
podman images
```

You should see `$PODMAN_REPO` listed among the Podman images.

### Step 3: Push the Podman Image to Amazon ECR

Create an ECR repository:

```bash
ECR_REPO="hello-world-app-<yourname>"
export ECR_REPO
echo $ECR_REPO
```

Create an environment variable for the AWS account:

```bash
AWS_ACCOUNT_ID="<aws-account-id>"
export AWS_ACCOUNT_ID
echo $AWS_ACCOUNT_ID
```

Create the ECR repository:

```bash
aws ecr create-repository --repository-name $ECR_REPO
```

Verify the repository exists:

```bash
aws ecr describe-repositories
```

Log in to ECR:

```bash
AWS_REGION="ap-southeast-1"
export AWS_REGION
echo $AWS_REGION
```

Get the password for ECR and log in:

```bash
aws ecr get-login-password --region $AWS_REGION | podman login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

If successful, you should see:

```
Login Succeeded
```

Tag the Podman image for ECR:

```bash
podman tag $ECR_REPO:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:latest
```

Push the Podman image to ECR:

```bash
podman push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:latest
```

### Step 4: Deploy to Amazon ECS

Create a variable for your ECS cluster:

```bash
ECS_CLUSTER="cluster-<yourname>"
export ECS_CLUSTER
echo $ECS_CLUSTER
```

Create an ECS cluster:

```bash
aws ecs create-cluster --cluster-name $ECS_CLUSTER
```

Register a Task Definition: Create a file named `task-definition.json`:

```json
{
  "family": "hello-world-task-<yourname>",
  "networkMode": "awsvpc",
  "executionRoleArn": "arn:aws:iam::<aws-account-id>:role/ECSExecutionRole",
  "containerDefinitions": [
    {
      "name": "hello-world-container",
      "image": "<aws-account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/hello-world-app-<yourname>:latest",
      "memory": 256,
      "cpu": 256,
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-create-group": "true",
          "awslogs-group": "<yourname>",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecsworkshop"
        }
      }
    }
  ],
  "requiresCompatibilities": [
    "FARGATE"
  ],
  "cpu": "256",
  "memory": "512"
}
```

Register the task definition:

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

Run the Task: Ensure you have a valid subnet and security group, then run:

```bash
aws ecs run-task --cluster $ECS_CLUSTER --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[<your-subnet-id>],securityGroups=[<your-security-group-id>],assignPublicIp=ENABLED}" --task-definition hello-world-task-<yourname>
```

Example:

```bash
aws ecs run-task --cluster $ECS_CLUSTER --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-0ac39594c83295aa3],securityGroups=[sg-0f337d85f089a9cb0],assignPublicIp=ENABLED}" --task-definition hello-world-task-<yourname>
```

### Verification

Go to the Amazon ECS console, navigate to your cluster, and check the Tasks tab. The task should appear with a status of `Running` or `Stopped`. View the logs in the Amazon CloudWatch Logs console under the log group `<yourname>` with the prefix `ecsworkshop` to see the "Hello, World!" output.
