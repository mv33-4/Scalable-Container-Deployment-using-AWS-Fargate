# Scalable Container Deployment on AWS Fargate

A practical, opinionated reference for building, publishing, and running containerized services on AWS Fargate with production-ready defaults for scalability, observability, and least-privilege security.

This repository contains examples and deployment templates that demonstrate how to:
- Build and publish Docker images to Amazon ECR
- Run containerized workloads on Amazon ECS (Fargate launch type)
- Front services with an Application Load Balancer (ALB)
- Configure service autoscaling and health checks
- Send container logs to CloudWatch Logs and configure basic alerts
- Integrate image build & deployment into GitHub Actions CI/CD

Why this repo
- Short, focused patterns you can copy into an application repo.
- Minimal, cloud-native defaults: awsvpc networking, awslogs, tasks using an execution role with least privilege.
- Deployment options: CLI/CloudFormation, Terraform, and AWS Copilot examples.

---

## Quick summary
- Primary goal: show an end-to-end flow from source code -> ECR image -> ECS Task (Fargate) -> ALB
- Intended audience: engineers who need a reproducible, maintainable Fargate deployment pattern
- Not a full product: this is a reference/guide with example templates and commands you can adapt.

---

## Contents

```text
2. Deploy a Docker container image on AWS Fargate/    Example deployment material and walkthrough
README.md                                          This file (replaced)
```

---

## Stack
- Language(s): agnostic (deployment examples are Docker + AWS CLI / CloudFormation / Terraform)
- Runtime / tools: Docker, AWS CLI, Amazon ECR, Amazon ECS (Fargate), Application Load Balancer
- Notable docs / examples: GitHub Actions workflow (build & push), CloudFormation/Terraform snippets

---

## Architecture (short)
Source code -> Docker build -> push to ECR -> ECS task (Fargate) running the container -> ALB routes external traffic to tasks. CloudWatch collects logs and metrics; ECS Service Auto Scaling adjusts desired count.

Key runtime pieces you will reuse from this repo:
- ECR repository for image storage
- Task Definition (awsvpc, container definition, cpu/memory, awslogs driver)
- ECS Service attached to an ALB target group
- Autoscaling configuration (target tracking policies or CloudWatch alarms)

---

## Quickstart (local -> Fargate)
Replace the placeholders (REGION, ACCOUNT_ID, REPO_NAME, CLUSTER_NAME, SERVICE_NAME, IMAGE_TAG) before running.

1) Build the image locally

```bash
docker build -t my-app:latest .
```

2) Create (or reuse) an ECR repository and authenticate Docker

```bash
aws ecr create-repository --repository-name my-app --region us-east-1 || true
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

3) Tag and push the image

```bash
docker tag my-app:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

4) Create an ECS cluster (Fargate)

```bash
aws ecs create-cluster --cluster-name my-fargate-cluster --region us-east-1
```

5) Register a task definition (replace image and log group)

Save the task JSON to task-def.json (example below), then register:

```bash
aws ecs register-task-definition --cli-input-json file://task-def.json
```

Container snippet (example):

```json
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "essential": true
    }
  ]
}
```

6) Create an ALB, target group, and ECS service that attaches to the target group. You can use CloudFormation/Terraform or the Console. Example CLI to create a service (assumes you have VPC/subnets/security groups configured):

```bash
aws ecs create-service \
  --cluster my-fargate-cluster \
  --service-name my-app-service \
  --task-definition my-app-task:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-...],securityGroups=[sg-...],assignPublicIp=ENABLED}"
```

7) Configure Service Auto Scaling (example: target tracking on CPU at 50%).

---

## Deployment options
- CloudFormation / AWS CLI: CloudFormation templates are recommended when you want repeatable stacks and IAM roles to be tracked.
- Terraform: good for multi-account/multi-environment workflows. Keep state in a secure remote backend.
- AWS Copilot: fastest path for application-centric deployments and built-in pipelines; tradeoff is less control over low-level resources.

---

## GitHub Actions (example)
A typical pipeline includes:
- checkout
- setup AWS credentials (least-privilege deploy role)
- build and push Docker image to ECR
- register new task definition and update the ECS service

Example steps (conceptual):

```yaml
- uses: actions/checkout@v4
- uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-region: us-east-1
    role-to-assume: arn:aws:iam::111111111111:role/github-actions-deploy
- name: Build and push
  uses: docker/build-push-action@v4
  with:
    push: true
    tags: ${{ env.ECR_REGISTRY }}/my-app:${{ github.sha }}
- name: Deploy to ECS
  run: |
    aws ecs register-task-definition --cli-input-json file://task-def.json
    aws ecs update-service --cluster my-fargate-cluster --service my-app-service --force-new-deployment
```

Notes: store credentials in GitHub Secrets and prefer role assumption over long-lived keys.

---

## Configuration & secrets
- Non-sensitive configuration: environment variables in the container definition.
- Secrets: store in AWS Secrets Manager or SSM Parameter Store and reference them in the task definition using the `secrets` field.
- Provide an execution role with minimal required permissions (e.g., ecr:GetAuthorizationToken, ecr:BatchGetImage, logs:CreateLogStream, logs:PutLogEvents, secretsmanager:GetSecretValue if used).

---

## Observability & alerts
- Logging: awslogs driver to CloudWatch Logs. Create a log group `/ecs/<service-name>` and set a retention policy.
- Metrics: use CloudWatch metrics (CPU, memory, ALB target health). Create CloudWatch Alarms for unhealthy host count and sustained high CPU/memory.
- Notifications: connect alarms to SNS and then to Slack/PagerDuty as required.

---

## Security & cost guidance
- Use least-privilege IAM roles and avoid embedding credentials in images.
- Use private ECR repositories; restrict who can push images.
- Use appropriate task sizes (cpu/memory) and autoscaling policies to reduce cost.
- If using public subnets and assignPublicIp, ensure security groups are restrictive.

---

## Troubleshooting
- Container failing to start: check CloudWatch Logs for the container. Inspect task events in the ECS Console for reasons like "CannotPullContainerError" or missing IAM permissions.
- ALB 5xx or unhealthy targets: verify health check path and container listening port mapping.
- Image not found: double-check the image URI and registry permissions.

---
