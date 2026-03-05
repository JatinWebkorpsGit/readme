# Mobile BFF – Production Infrastructure

## Overview

The **Mobile BFF (Backend for Frontend)** service is deployed on AWS using **ECS Fargate**.
Container images are stored in **Amazon ECR**, and the service runs inside an ECS cluster with multiple supporting containers for monitoring and security.

This document provides an overview of the production infrastructure, services, and health endpoints.

---

# Environment Details

| Property       | Value      |
| -------------- | ---------- |
| Environment    | Production |
| Application    | Mobile BFF |
| Cloud Provider | AWS        |
| Region         | us-west-2 (Oregon)  |

---

# Infrastructure Creation Flow

The production infrastructure was created in the following sequence:


1. VPC, Subnets, and Transit Gateway configurations were provisioned by the Network Team.

2. Application secrets were created and stored in HashiCorp Vault.

3. ECR production repository was provisioned to store Mobile BFF container images.

   Terraform Stack:
   https://github.com/bwhhg/bwh-mobile-bff/tree/main/stacks/mobile_bff_ecr_prod

4. Kinesis Firehose production pipeline was created for log and data streaming.

   Terraform Stack:
   https://github.com/bwhhg/bwh-mobile-bff/tree/main/stacks/mobile_bff_firehose_prod

5. json2tf was used to convert JSON-based infrastructure definitions into Terraform configuration, primarily for generating AWS Security Group resources.

6. ECS production infrastructure was provisioned including cluster, services, task definitions, load balancers, IAM roles, Route53 records, and integrations.

   Terraform Stack:  
   https://github.com/bwhhg/bwh-mobile-bff/tree/main/stacks/mobile_bff_ecs_prod

   The ECS stack was applied in two stages:

   • During the first Terraform apply, the Lambda-based SSL management configuration was temporarily commented out because it required the `mobile_bff_cert_renewal` resources to exist.

   • After the required resources were created, the SSL management configuration was uncommented and Terraform was applied again.

7. ECS VPC Link (CTLZ) was required to allow API Gateway to communicate with services running inside the VPC.

   Terraform Stack:
   https://github.com/bwhhg/bwh-mobile-bff/tree/main/stacks/mobile_bff_ecs_prod

   Note:
   The Mobile team did not have sufficient permissions to create the VPC Link.
   Therefore, a request was raised with the Infrastructure/Platform Engineering team,
   and they executed `tfv apply` to provision the VPC Link.

8. Lambda-based SSL management infrastructure was deployed for certificate management and renewal.

   Terraform Stack:  
   https://github.com/bwhhg/bwh-mobile-bff/tree/main/stacks/mobile_bff_ssl_mgmt_prod

   The SSL management stack creates a Lambda function responsible for certificate renewal and an EventBridge rule that triggers the Lambda.

   Post Deployment Validation:

   a. Navigate to the EventBridge rule created by the SSL management stack.

   b. Open the **Trigger** configuration.

   c. Go to **Targets** and select the Lambda function target.

   d. View the **Constant Input** section and copy the JSON payload.

   e. Navigate to the created Lambda function.

   f. Open the **Test** tab.

   g. Paste the copied JSON payload into the test event.

   h. Run the test to trigger the Lambda and verify certificate renewal.

   This step ensures the certificate renewal workflow is functioning correctly.

---

# Infrastructure Architecture

The Mobile BFF service is deployed using the following AWS components:

* Container images stored in Amazon ECR
* Containers run inside Amazon ECS
* Compute powered by AWS Fargate

### Deployment Flow

```
User Request
      │
      ▼
Load Balancer
      │
      ▼
ECS Cluster (mobile-prod-ecs-us-west-2)
      │
      ▼
ECS Service (mobilebff-prod-ecs-fargate-us-west-2)
      │
      ▼
Task Definition
      │
      ├── mobile-bff
      ├── ecs-service-connect
      ├── datadog
      └── aquasec
```

---

# Container Images

The ECS task runs the following containers:

| Container           | Purpose                                    |
| ------------------- | ------------------------------------------ |
| mobile-bff          | Main backend service handling API requests |
| ecs-service-connect | Enables service discovery within ECS       |
| datadog             | Monitoring and observability agent         |
| aquasec             | Container security scanning                |

---

# ECS Configuration

### ECS Cluster

```
mobile-prod-ecs-us-west-2
```

### ECS Service

```
mobilebff-prod-ecs-fargate-us-west-2
```

### Task Definition

The service runs tasks containing four containers responsible for application logic, monitoring, networking, and security.

---

# Health Check Endpoints

These endpoints are used internally to verify the health of the infrastructure.

| Endpoint                                                      | Description                     |
| ------------------------------------------------------------- | ------------------------------- |
| https://bff-api.mobile.bwhhg.io/health/live                   | Infrastructure health check     |
| https://bff.bff-api.mobile.bwhhotelgroup.com/prod/health/live | Production service health check |

---

# Secrets Management

Secrets used by the Mobile BFF production services are securely managed using **HashiCorp Vault** and **AWS Secrets Manager**.

### Secret Flow

```
HashiCorp Vault
      ↓
Terraform retrieves secrets
      ↓
Secrets stored in AWS Secrets Manager
      ↓
ECS services consume secrets at runtime
```

HashiCorp Vault acts as the **primary secret storage**, while AWS Secrets Manager is used to make those secrets available to AWS services securely.

### Benefits of This Approach

* Centralized secret management in Vault
* Secure integration with AWS infrastructure
* Secrets are never stored in Terraform code or Git repositories
* ECS services retrieve secrets securely during runtime

---

# Monitoring & Observability

Monitoring and logging are handled through:

* Datadog agent container
* Amazon CloudWatch logs
* Application health endpoints

Metrics monitored include:

* CPU usage
* Memory usage
* Container health
* API availability

---

# Security

Security controls include:

* Container scanning with Aqua Security
* IAM role-based access control
* Secrets stored in secure secret management services

Sensitive credentials and tokens are **not stored in this repository**.

---

# Terraform Deployment

Infrastructure is managed using **Terraform**.

Typical deployment workflow:

1. Initialize Terraform

```
tfv init
```

2. Review infrastructure changes

```
tfv plan
```

3. Apply infrastructure

```
tfv apply
```

Terraform state is stored remotely according to the organization’s infrastructure standards.

---

# Repository Structure

```
bwh-mobile-bff
│
└── stacks
    ├── mobile_bff_ecs_dev
    ├── mobile_bff_ecs_qa
    └── mobile_bff_ecs_prod
        ├── acm.tf
        ├── alb.tf
        ├── apigw.tf
        ├── aws.tf
        ├── container_definition.tf
        ├── data.tf
        ├── ecs.tf
        ├── iam.tf
        ├── kms.tf
        ├── locals.tf
        ├── metadata.tf
        ├── nlb.tf
        ├── route53.tf
        ├── secrets.tf
        ├── terraform.tf
        ├── terraform.tfvars
        ├── variables.tf
        ├── vault.tf
        └── versions.tf
```

---

# Maintainers

Infrastructure maintained by the **Mobile Team**.

For infrastructure changes or production issues, contact the platform engineering team.
