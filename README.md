# Containerized Microservices with Amazon ECS Fargate and Service Discovery

**Project 6 | Containers | AWS CDK (TypeScript)**

This project demonstrates how to decompose a Node.js monolith into three independently packaged services—**Auth**, **Orders**, and **Notifications**—and run them as containers on Amazon ECS with AWS Fargate.

> **Implementation note — blue/green deployment compatibility:** This repository uses one shared Application Load Balancer (ALB) with three path-based listener rules. CodeDeploy's ECS blue/green integration manages traffic through listener-level target-group pairs; it is not a drop-in independent controller for three unrelated path rules on one shared listener. The core CDK deployment therefore provisions ECS services with the standard ECS rolling deployment controller and implements the shared ALB routing. The repository includes CodeDeploy AppSpec/build artifacts and a deployment design guide describing the supported deployment choices. For strict per-service blue/green, use a deployment topology with independently controllable listener rules and target-group pairs, or evaluate the native ECS blue/green strategy with `productionListenerRule` per service. Do not enable CodeDeploy against the shared listener without adapting the routing model and validating it in a sandbox.
>
> This is an intentional architecture constraint, not a hidden implementation assumption.

## Table of Contents

- [Solution Overview](#solution-overview)
- [Architecture Diagram](#architecture-diagram)
- [Architecture Components](#architecture-components)
- [Request and Service Communication Flow](#request-and-service-communication-flow)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Customize the Solution](#customize-the-solution)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Configure the Application](#2-configure-the-application)
  - [3. Install Dependencies and Test](#3-install-dependencies-and-test)
  - [4. Bootstrap and Deploy](#4-bootstrap-and-deploy)
  - [5. Build and Push Container Images](#5-build-and-push-container-images)
- [Service Discovery](#service-discovery)
- [Secrets Management](#secrets-management)
- [Redis Session Store](#redis-session-store)
- [CI/CD and Deployment Strategy](#cicd-and-deployment-strategy)
- [Observability and Distributed Tracing](#observability-and-distributed-tracing)
- [Testing and Validation](#testing-and-validation)
- [Security Considerations](#security-considerations)
- [Cost Considerations](#cost-considerations)
- [Cleanup](#cleanup)
- [Known Limitations and Production Hardening](#known-limitations-and-production-hardening)
- [References](#references)
- [License](#license)

## Solution Overview

The solution separates identity, order processing, and notification responsibilities into independent Node.js services. Each service is packaged as a Docker image, stored in a private Amazon Elastic Container Registry (ECR) repository, and deployed as an ECS Fargate service.

The public ALB accepts HTTPS traffic and routes requests based on URL path:

| Path | Destination | Cloud Map name |
|---|---|---|
| `/api/auth/*` | Auth ECS service | `auth.service.local` |
| `/api/orders/*` | Orders ECS service | `orders.service.local` |
| `/api/notifications/*` | Notifications ECS service | `notifications.service.local` |

The services run in private subnets across two Availability Zones. AWS Cloud Map provides private DNS-based discovery. AWS Secrets Manager supplies database/API credentials to task containers at runtime. ElastiCache for Redis stores shared session state so that any task can serve a request without relying on local memory.

## Architecture Diagram

![Solution architecture](architecture/architecture-diagram.png)

Editable diagram source: [`architecture/architecture-diagram.mmd`](architecture/architecture-diagram.mmd).

## Architecture Components

| Component | Purpose | Implementation |
|---|---|---|
| Amazon VPC | Network isolation | Public and private subnets across two AZs |
| Application Load Balancer | Internet entry and path routing | HTTPS listener and three path rules |
| Amazon ECS | Container orchestration | One cluster and three Fargate services |
| AWS Fargate | Serverless container compute | `awsvpc` networking; no EC2 hosts |
| Amazon ECR | Private image registry | One repository per microservice; scan on push |
| AWS Cloud Map | Service discovery | Private DNS namespace and service registrations |
| AWS Secrets Manager | Runtime credentials | ECS container `secrets` injection |
| Amazon ElastiCache for Redis | Shared session cache | Private subnet group, TLS, auth token |
| AWS X-Ray | Distributed tracing | AWS X-Ray SDK instrumentation in Node.js |
| Amazon CloudWatch | Logs and alarms | Per-service log groups and metrics |
| AWS CDK | Infrastructure as Code | TypeScript CDK v2 application |

## Request and Service Communication Flow

1. A client calls `https://<ALB-DNS>/api/orders/...`.
2. The ALB listener rule forwards the request to the Orders target group.
3. The Orders container validates the session using Redis.
4. If it needs identity information, Orders resolves `auth.service.local` through Cloud Map and calls the Auth service over the private VPC network.
5. The Auth and Orders services use the shared Redis cache for session state.
6. Notifications can be called through `notifications.service.local` for internal event delivery.
7. Each service writes structured logs to CloudWatch and emits X-Ray trace segments.

The sample application uses in-memory demo data for orders and a Redis-backed session example. Orders validates a supplied session by calling Auth through its Cloud Map DNS name. It is a learning workload, not a production commerce platform.

## Repository Structure

```text
ecs-fargate-microservices/
├── README.md
├── architecture/
│   ├── architecture-diagram.png
│   └── architecture-diagram.mmd
├── bin/
│   └── app.ts
├── lib/
│   └── microservices-stack.ts
├── services/
│   ├── auth/
│   ├── orders/
│   └── notifications/
├── cicd/
│   ├── buildspec.yml
│   ├── appspec.yaml
│   └── deployment-notes.md
├── test/
│   └── microservices-stack.test.ts
├── cdk.json
├── package.json
├── tsconfig.json
└── .gitignore
```

## Prerequisites

- An AWS account with permissions to create VPC, ALB, ECS, ECR, Cloud Map, Secrets Manager, ElastiCache, IAM, CloudWatch, and X-Ray resources.
- AWS CLI v2 configured with a deployment profile.
- Node.js 20 or later and npm.
- AWS CDK v2 CLI (`npm install -g aws-cdk`).
- Docker Desktop or Docker Engine.
- Git.
- A supported AWS Region with Fargate, ElastiCache, and the required services available.

Use a dedicated sandbox account. The deployment creates billable resources, including NAT Gateway, ALB, Fargate tasks, and Redis.

## Customize the Solution

### 1. Clone the Repository

```bash
git clone https://github.com/<your-user>/ecs-fargate-microservices.git
cd ecs-fargate-microservices
```

### 2. Configure the Application

Set the deployment environment:

```bash
export AWS_PROFILE=your-sandbox-profile
export CDK_DEFAULT_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
export CDK_DEFAULT_REGION=us-east-1
```

The CDK app accepts `projectName` and `allowedCidr` context values. Restrict `allowedCidr` to your trusted public IP/CIDR before exposing the ALB. The default `0.0.0.0/0` is convenient for a short-lived lab only.

Create the application secrets before deploying:

```bash
aws secretsmanager create-secret \
  --name microservices/auth \
  --secret-string '{"JWT_SECRET":"replace-with-a-long-random-value","API_KEY":"replace-me"}'

aws secretsmanager create-secret \
  --name microservices/orders \
  --secret-string '{"DATABASE_URL":"postgres://user:password@db.example:5432/orders"}'

aws secretsmanager create-secret \
  --name microservices/notifications \
  --secret-string '{"NOTIFICATION_API_KEY":"replace-me"}'
```

For a production system, use a managed database, private endpoints where appropriate, secret rotation, and separate least-privilege secrets for each service.

### 3. Install Dependencies and Test

```bash
npm ci
npm test
npm run build
```

### 4. Bootstrap and Deploy

```bash
cdk bootstrap aws://$CDK_DEFAULT_ACCOUNT/$CDK_DEFAULT_REGION
cdk synth
cdk deploy --require-approval broadening
```

The stack outputs the ALB DNS name, Redis primary endpoint, and ECR repository URIs.

### 5. Build and Push Container Images

Authenticate Docker to ECR:

```bash
aws ecr get-login-password --region "$CDK_DEFAULT_REGION" \
  | docker login --username AWS --password-stdin \
    "$CDK_DEFAULT_ACCOUNT.dkr.ecr.$CDK_DEFAULT_REGION.amazonaws.com"
```

Build and push each service (replace the repository names with the stack outputs):

```bash
docker build -t auth ./services/auth
docker tag auth:latest <AUTH_ECR_URI>:bootstrap
docker push <AUTH_ECR_URI>:bootstrap

docker build -t orders ./services/orders
docker tag orders:latest <ORDERS_ECR_URI>:bootstrap
docker push <ORDERS_ECR_URI>:bootstrap

docker build -t notifications ./services/notifications
docker tag notifications:latest <NOTIFICATIONS_ECR_URI>:bootstrap
docker push <NOTIFICATIONS_ECR_URI>:bootstrap
```

The initial stack references the immutable `bootstrap` tag. Push these first images, then force a new deployment for each ECS service so tasks can pull them. For later releases, push a new immutable Git commit tag and redeploy the CDK stack with `-c imageTag=<commit-sha>` so task definitions reference that image tag. Never reuse an immutable tag.

## Service Discovery

The CDK stack creates a Cloud Map private DNS namespace named `service.local` and registers each ECS service. The service names are:

- `auth.service.local`
- `orders.service.local`
- `notifications.service.local`

Containers resolve these names using the VPC-provided DNS resolver. Service-to-service calls remain inside the VPC and do not traverse the public ALB.

The sample Orders service demonstrates a call to Auth using `AUTH_SERVICE_URL=http://auth.service.local:3000`. Ensure the ECS task security group allows service-to-service traffic on the application port.

## Secrets Management

Secrets are not stored in Dockerfiles, source code, or plaintext task-definition environment variables. ECS task definitions reference Secrets Manager values through the container `secrets` property. The ECS task execution role requires permission to retrieve the referenced secrets. ECS injects values when a task starts; a running container does not automatically receive a rotated value. Force a new deployment after rotation or use application-side retrieval if immediate refresh is required.

## Redis Session Store

The Redis replication group is deployed in private subnets with encryption in transit and an authentication token. The token is generated and stored in Secrets Manager. The ECS task security group is allowed to connect to Redis on TCP 6379.

Redis is a cache/session store, not the durable system of record. Configure TTLs, eviction policy, backups, Multi-AZ/failover, and capacity based on workload requirements. The lab defaults are deliberately small and should be sized before production use.

## CI/CD and Deployment Strategy

The `cicd/buildspec.yml` file illustrates image build, test, and ECR push stages. ECR performs the configured scan-on-push behavior. The `cicd/appspec.yaml` is a CodeDeploy ECS AppSpec template for a **single service deployment group**. These files are CI/CD building blocks; a CodePipeline source connection, CodeBuild project, CodeDeploy application, deployment groups, and deploy actions must be provisioned and wired for the selected deployment topology.

### Important deployment topology decision

CodeDeploy ECS blue/green uses a production listener (and optional test listener) with a target-group pair for the service being deployed. The project requirement also uses one shared ALB listener with three path rules. CodeDeploy's listener-level traffic control should not be assumed to independently swap one path rule while leaving the other two untouched.

Choose one of these deployment models before enabling blue/green:

1. **Keep one shared ALB and three independent path rules:** use ECS rolling deployments (the deployed baseline in this repository), or evaluate ECS-native blue/green using a `productionListenerRule` ARN per service. This gives each service an independently controlled rule.
2. **Use CodeDeploy ECS blue/green:** create an independently controllable listener/rule topology for each service and configure the production/test traffic routes and target-group pair for that service. Validate that deployment of one service cannot alter the other services' traffic.
3. **Coordinated application release:** deploy the three services as one release unit with an explicitly designed traffic-switching mechanism and rollback strategy.

Do not copy the AppSpec file into production and assume it completes blue/green configuration. The CodeDeploy application, deployment group, ECS service deployment controller, listeners, target groups, IAM service role, and pipeline deploy action must all agree.

## Observability and Distributed Tracing

Each service writes JSON application logs to its own CloudWatch log group. X-Ray instrumentation is included as a dependency and a middleware integration point. For production, instrument outbound HTTP calls and propagate trace context across service boundaries. Add an X-Ray daemon/collector sidecar or use a supported OpenTelemetry Collector approach, grant the task role the required trace-write permissions, and validate the end-to-end service map.

## Testing and Validation

After deployment:

```bash
curl -i "http://<ALB_DNS>/api/auth/health"
curl -i "http://<ALB_DNS>/api/orders/health"
curl -i "http://<ALB_DNS>/api/notifications/health"
```

Expected result: HTTP 200 with a JSON health response from the selected service.

Additional validation:

- Confirm three ECS services reach `RUNNING` desired count.
- Confirm each ECR repository has an image with image scanning results.
- Confirm Cloud Map records resolve from an ECS task.
- Confirm the ECS task role and execution role are distinct and scoped.
- Confirm no secrets appear in task-definition plaintext environment variables or logs.
- Confirm Redis connectivity and TLS from the application subnets.
- Confirm ALB target groups show healthy IP targets.
- Confirm CloudWatch logs contain service-specific events.
- Confirm trace IDs propagate between Orders and Auth when X-Ray collection is enabled.
- Exercise task replacement and verify that session state remains available through Redis.

## Security Considerations

- Place ECS tasks and Redis in private subnets.
- Permit inbound application traffic to tasks only from the ALB security group; permit service-to-service traffic only from the ECS service security group.
- Use HTTPS at the ALB with an ACM certificate for production. The lab stack can be deployed with an HTTP listener until a certificate is supplied.
- Use separate task execution and task roles.
- Grant `secretsmanager:GetSecretValue` only for the exact service secrets.
- Enable ECR image scanning on push and use immutable image tags.
- Avoid public IP assignment on Fargate tasks.
- Use TLS for Redis and rotate the authentication token.
- Enable CloudTrail, GuardDuty, Security Hub, and centralized logging as appropriate.
- Pin dependencies and scan container base images.

## Cost Considerations

The main cost drivers are:

- NAT Gateway hourly and data-processing charges.
- Application Load Balancer hourly and LCU charges.
- Fargate vCPU and memory consumption for all running tasks.
- ElastiCache node-hours and data transfer.
- ECR image storage and scanning.
- CloudWatch log ingestion/storage and X-Ray traces.
- Secrets Manager secret storage and API calls.

The lab defaults use one NAT Gateway and one small Redis node to limit cost, but still incur ongoing charges. Stop or delete the stack when not in use. For production, use Multi-AZ NAT design and appropriately sized highly available Redis.

## Cleanup

Delete the CDK stack:

```bash
cdk destroy
```

Delete the manually created secrets after confirming they are no longer needed:

```bash
aws secretsmanager delete-secret --secret-id microservices/auth --force-delete-without-recovery
aws secretsmanager delete-secret --secret-id microservices/orders --force-delete-without-recovery
aws secretsmanager delete-secret --secret-id microservices/notifications --force-delete-without-recovery
```

ECR repositories may retain images if configured for retention. Verify and delete them if required.

## Known Limitations and Production Hardening

- This is a learning reference implementation. It does not include a production database, durable messaging broker, WAF, custom domain, certificate provisioning, or a full release approval workflow.
- The sample services use demo data. Replace it with durable data stores and idempotent APIs.
- The HTTP listener is for lab validation only. Use HTTPS and restrict ingress for production.
- Redis credentials and application secrets must be rotated and monitored.
- For production, use multi-AZ resilience, autoscaling, health checks, deployment alarms, and tested rollback procedures.
- The CodeDeploy blue/green and shared path-based ALB interaction is an explicit design constraint documented above.

## References

- [Amazon ECS blue/green deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)
- [ALB resources for ECS blue/green deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/alb-resources-for-blue-green.html)
- [CodeDeploy ECS deployments](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-ecs.html)
- [Troubleshooting ECS blue/green deployments](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/troubleshooting-blue-green.html)
- [ECS service discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)
- [Inject Secrets Manager secrets into ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/secrets-envvar-secrets-manager.html)
- [AWS CDK v2 documentation](https://docs.aws.amazon.com/cdk/v2/guide/home.html)
- [Amazon ElastiCache for Redis OSS](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html)
- [AWS X-Ray SDK for Node.js](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-nodejs.html)

## License

This project is provided for educational use. Add an explicit license file before publishing the repository.
