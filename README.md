Deploy a Docker container image on AWS Fargate

## Cloud Service Provider

- Amazon Web Services

# Scalable Container Deployment using AWS Fargate

A reference project and deployment guide for running containerized applications on AWS Fargate with best-practice configuration for scalability, observability, and cost control.

This repository contains examples and instructions to:
- Build and push container images to Amazon ECR
- Deploy containers to Amazon ECS on Fargate (task definition, service, ALB)
- Configure autoscaling and health checks
- Enable logging, monitoring, and alerting
- Integrate with CI/CD pipelines (GitHub Actions example)

---

## Table of contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Quickstart (manual)](#quickstart-manual)
- [Recommended deployment options](#recommended-deployment-options)
  - [CloudFormation / AWS CLI](#cloudformation--aws-cli)
  - [Terraform](#terraform)
  - [ecs-cli / Copilot (optional)](#ecs-cli--copilot-optional)
- [CI/CD (GitHub Actions) example](#cicd-github-actions-example)
- [Configuration & secrets management](#configuration--secrets-management)
- [Logging, metrics & alerts](#logging-metrics--alerts)
- [Cost & security considerations](#cost--security-considerations)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Overview
This project demonstrates a practical, production-ready pattern for deploying containerized services on AWS Fargate. Fargate removes the need to manage EC2 instances and simplifies scaling. The README provides the end-to-end workflow from building a Docker image to running and autoscaling a service behind an Application Load Balancer (ALB).

---

## Architecture
High-level components:
- Source code -> Docker image
- Amazon ECR (private) stores the image
- Amazon ECS cluster (Fargate launch type)
- Task Definition with container definition, CPU/memory, port mappings
- ECS Service fronted by an Application Load Balancer (ALB) with target group health checks
- Autoscaling (ECS Service Auto Scaling) based on CPU, memory, or custom CloudWatch metrics
- CloudWatch Logs for container logs, CloudWatch Metrics and Alarms for monitoring

ASCII diagram:
App Repo -> Docker build -> ECR
          ECR -> ECS Task (Fargate) -> ALB -> Internet
                       |-> CloudWatch Logs / Metrics
                       |-> Auto Scaling

---

## Features
- Container image build and publish workflow
- ECS Task Definition and Service configured for Fargate
- ALB for routing and health checks
- Service autoscaling example
- Centralized logging to CloudWatch Logs
- Example GitHub Actions pipeline for CI/CD
- Guidance for secrets with Secrets Manager or SSM Parameter Store

---

## Prerequisites
- AWS account with permissions to ECR, ECS, IAM, CloudFormation (or Terraform), ALB/ELB, CloudWatch
- AWS CLI installed and configured (aws configure)
- Docker installed (for local image build)
- Optional: Terraform or AWS CloudFormation knowledge
- Optional: GitHub repository and Actions enabled for CI

---

## Quickstart (manual)
These are minimal example steps to get a container running on Fargate. Replace placeholders (REGION, ACCOUNT_ID, REPO_NAME, CLUSTER_NAME, SERVICE_NAME, IMAGE_TAG) with your values.

1. Create an ECR repository:
   aws ecr create-repository --repository-name my-app --region us-east-1

2. Authenticate Docker to ECR:
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

3. Build and push the image:
   docker build -t my-app:latest .
   docker tag my-app:latest <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
   docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

4. Create an ECS cluster (Fargate):
   aws ecs create-cluster --cluster-name my-fargate-cluster --region us-east-1

5. Register a task definition (example JSON snippet shown below) and create a service using the AWS Console, CloudFormation, Terraform, or aws CLI. The task should reference the ECR image and set required CPU/memory and port mappings.

Minimal task definition (example container portion):
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
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

6. Create an ALB and target group, then create the ECS service attached to the ALB listener so your tasks can receive traffic.

7. Configure autoscaling for the ECS Service (target tracking policy, e.g., average CPU utilization = 50%).

---

## Recommended deployment options

### CloudFormation / AWS CLI
- Provide a CloudFormation template that creates:
  - IAM roles (task execution role)
  - ECR repository
  - ECS Cluster
  - Task Definition
  - ALB, target group, listener
  - ECS Service
  - CloudWatch log groups and alarms
- Deploy with:
  aws cloudformation deploy --template-file template.yaml --stack-name my-fargate-stack --capabilities CAPABILITY_NAMED_IAM

### Terraform
- Use AWS provider with resources:
  - aws_ecr_repository
  - aws_ecs_cluster
  - aws_ecs_task_definition
  - aws_lb, aws_lb_target_group, aws_lb_listener
  - aws_ecs_service (with launch_type = FARGATE and network_configuration)
  - aws_appautoscaling_target & aws_appautoscaling_policy
- Keep state secure (remote backend).

### ecs-cli / Copilot (optional)
- AWS Copilot simplifies building and deploying microservices to ECS (Fargate) including pipelines:
  - copilot init
  - copilot svc deploy
- Good for quick prototypes.

---

## CI/CD (GitHub Actions) example
A simple pipeline:
- Build Docker image
- Log in to ECR
- Push image to ECR
- Deploy or update ECS task definition (via AWS CLI or CloudFormation)

Key steps in GitHub Actions:
- actions/checkout
- aws-actions/configure-aws-credentials
- docker/build-push-action or run docker build/push manually
- Use an automated task-definition update action or a script that registers a new task definition and updates the service

Note: Keep AWS credentials in GitHub Secrets with least privilege (e.g., deploy role).

---

## Configuration & secrets management
- Use environment variables in task definition for non-sensitive config.
- Store secrets (DB passwords, API keys) in AWS Secrets Manager or Parameter Store (SSM) and reference them in the task definition (via secrets parameter).
- Use task execution IAM role with minimal permissions:
  - ecr:GetAuthorizationToken
  - ecr:BatchGetImage, ecr:GetDownloadUrlForLayer
  - logs:CreateLogStream, logs:PutLogEvents
  - secretsmanager:GetSecretValue / ssm:GetParameters (only if using secrets)

---

## Logging, metrics & alerts
- Configure awslogs driver so containers send logs to CloudWatch Logs.
- Create CloudWatch Alarms for:
  - Task/Service unhealthy host count
  - High CPU / Memory usage
  - ALB target unhealthy host counts
- Optionally integrate with SNS for notification or PagerDuty/Slack via Lambda.








