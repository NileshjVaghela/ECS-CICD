# ECS CI/CD Hands-on Exercise - Complete Guide

## Overview

This hands-on exercise teaches students how to create a complete CI/CD pipeline for Amazon ECS using AWS CodePipeline. The exercise is divided into two phases:

- **Phase 1:** Staging environment with automated CI/CD pipeline
- **Phase 2:** Production environment with manual approval workflow

---

## Exercise Structure

### Phase 1: Staging Environment (Instructor + Students)

**Instructor deploys base staging infrastructure:**

The instructor deploys the staging ECS infrastructure using CloudFormation. This includes:

- **VPC** with 2 public subnets across availability zones
- **Application Load Balancer** with target group and listener
- **ECS Fargate Cluster** (tagged as staging)
- **ECS Service** with task definition (using placeholder nginx image)
- **ECR Repository** for storing Docker images
- **Security Groups** for ALB and ECS tasks
- **IAM Roles** for ECS task execution
- **CloudWatch Log Group** for container logs

**Deployment Command:**
```bash
aws cloudformation create-stack \
  --stack-name ecs-staging \
  --template-body file://ecs-base-infrastructure.yaml \
  --parameters ParameterKey=Environment,ParameterValue=staging \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**Note:** The `Environment` parameter is set to `staging`. This tags all resources appropriately.

**File:** `ecs-base-infrastructure.yaml`

**Students build CI/CD pipeline:**

Students create a complete CI/CD pipeline using the **AWS Console only**:

1. **CodeCommit Repository** - AWS-managed Git repository for source control
2. **S3 Artifact Bucket** - Storage for pipeline artifacts
3. **IAM Roles** - Permissions for CodeBuild and CodePipeline
4. **CodeBuild Project** - Builds Docker images and pushes to ECR
5. **CodePipeline** - Orchestrates the entire CI/CD workflow

**Pipeline Flow (Phase 1):**
```
CodeCommit (Source) → CodeBuild (Build) → Deploy to Staging
```

**File:** `CICD-HANDSON-CONSOLE.md`

---

### Phase 2: Production Environment with Manual Approval (Advanced)

**Instructor deploys production infrastructure:**

Deploy a second ECS cluster for production using the same template:

```bash
aws cloudformation create-stack \
  --stack-name ecs-production \
  --template-body file://ecs-base-infrastructure.yaml \
  --parameters ParameterKey=Environment,ParameterValue=production \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**Students enhance pipeline with approval:**

Students extend the pipeline to include:
- **Manual Approval Stage** - Review before production deployment
- **SNS Notifications** - Email alerts for approvals
- **Production Deployment** - Deploy to production after approval

**Pipeline Flow (Phase 2):**
```
Source → Build → Deploy Staging → Manual Approval → Deploy Production
                                        ↓
                                  Email Notification
```

**File:** `PHASE2-PRODUCTION-DEPLOYMENT.md`

---

## Learning Objectives

**Phase 1 - Staging Environment:**

✅ How to set up AWS CodeCommit for source control  
✅ How to configure CodeBuild to build Docker images  
✅ How to push Docker images to Amazon ECR  
✅ How to create a CodePipeline with multiple stages  
✅ How to configure IAM roles for CI/CD services  
✅ How to automate ECS deployments  
✅ How to use CloudWatch Events to trigger pipelines  
✅ How to troubleshoot CI/CD pipeline issues  

**Phase 2 - Production Environment:**

✅ How to deploy multi-environment infrastructure  
✅ How to implement manual approval workflows  
✅ How to configure SNS notifications  
✅ How to promote code from staging to production  
✅ How to implement production-grade CI/CD practices  

---

## Prerequisites

**For Instructor:**
- AWS Account with appropriate permissions
- AWS CLI configured
- CloudFormation template: `ecs-base-infrastructure.yaml`

**For Students:**
- Access to AWS Console
- Basic understanding of Docker
- Basic understanding of Git
- AWS CloudShell access or local terminal with AWS CLI

---

## Time Estimate

- **Phase 1 (Instructor):** 10-15 minutes (CloudFormation deployment for staging)
- **Phase 1 (Students):** 60-90 minutes (CI/CD pipeline creation)
- **Phase 2 (Instructor):** 10-15 minutes (CloudFormation deployment for production)
- **Phase 2 (Students):** 30-45 minutes (Add approval and production deployment)

---

## Architecture Diagram

**Phase 1 - Staging Environment:**
```
┌─────────────┐
│ CodeCommit  │
│ Repository  │
└──────┬──────┘
       │ (git push)
       ↓
┌─────────────┐
│ CodePipeline│
│             │
│  ┌────────┐ │
│  │ Source │ │ ← Pulls code from CodeCommit
│  └────┬───┘ │
│       ↓     │
│  ┌────────┐ │
│  │ Build  │ │ ← CodeBuild: docker build & push to ECR
│  └────┬───┘ │
│       ↓     │
│  ┌────────┐ │
│  │ Deploy │ │ ← Updates ECS Staging Service
│  └────────┘ │
└─────────────┘
       │
       ↓
┌─────────────┐      ┌──────────────────┐
│     ECR     │─────→│ ECS Staging      │
│ (Container  │      │ Cluster/Service  │
│   Images)   │      └────────┬─────────┘
└─────────────┘               │
                              ↓
                     ┌─────────────────┐
                     │ Load Balancer   │
                     │ (Staging)       │
                     └─────────────────┘
```

**Phase 2 - Multi-Environment with Approval:**
```
┌─────────────┐
│ CodeCommit  │
└──────┬──────┘
       │
       ↓
┌──────────────────────────────────────────────────────┐
│                   CodePipeline                       │
│                                                      │
│  Source → Build → Deploy Staging → Manual Approval  │
│                                          ↓           │
│                                    Deploy Production │
└──────────────────────────────────────────────────────┘
       │                                      │
       ↓                                      ↓
┌──────────────┐                      ┌──────────────┐
│   Staging    │                      │ Production   │
│ ECS Cluster  │                      │ ECS Cluster  │
│      +       │                      │      +       │
│     ALB      │                      │     ALB      │
└──────────────┘                      └──────────────┘
```

---

## Files Included

| File | Purpose | Used By |
|------|---------|---------|
| `README.md` | This overview document | Everyone |
| `ecs-base-infrastructure.yaml` | CloudFormation template for base ECS setup | Instructor |
| `CICD-HANDSON-CONSOLE.md` | Step-by-step AWS Console instructions | Students |
| `EXERCISE-OVERVIEW.md` | Quick reference guide | Everyone |

---

## Getting Started

### For Instructors:

**Phase 1 - Deploy Staging:**
1. Review `ecs-base-infrastructure.yaml`
2. Deploy the CloudFormation stack with staging environment:
   ```bash
   aws cloudformation create-stack \
     --stack-name ecs-staging \
     --template-body file://ecs-base-infrastructure.yaml \
     --parameters ParameterKey=Environment,ParameterValue=staging \
     --capabilities CAPABILITY_NAMED_IAM \
     --region us-east-1
   ```
3. Share the stack outputs with students
4. Provide students with `CICD-HANDSON-CONSOLE.md`

**Phase 2 - Deploy Production (Optional):**
1. Deploy production infrastructure:
   ```bash
   aws cloudformation create-stack \
     --stack-name ecs-production \
     --template-body file://ecs-base-infrastructure.yaml \
     --parameters ParameterKey=Environment,ParameterValue=production \
     --capabilities CAPABILITY_NAMED_IAM \
     --region us-east-1
   ```
2. Provide students with `PHASE2-PRODUCTION-DEPLOYMENT.md`

### For Students:

**Phase 1:**
1. Get the staging stack outputs from your instructor
2. Follow the instructions in `CICD-HANDSON-CONSOLE.md`
3. Create the CI/CD pipeline using AWS Console
4. Test the pipeline by pushing code changes

**Phase 2 (Optional Advanced):**
1. Follow `PHASE2-PRODUCTION-DEPLOYMENT.md` to add production environment
2. Implement manual approval workflow
3. Test staging-to-production promotion

---

## Sample Application

Students will deploy a simple Nginx web application:

**Dockerfile:**
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

**index.html:**
```html
<!DOCTYPE html>
<html>
<head>
    <title>ECS CI/CD Demo</title>
</head>
<body>
    <h1>Hello from ECS CI/CD Pipeline!</h1>
    <p>Version 1.0</p>
</body>
</html>
```

---

## Success Criteria

**Phase 1 - Staging:**

✅ CodePipeline is created with Source, Build, and Deploy stages  
✅ Pipeline automatically triggers on code commits  
✅ Docker image is built and pushed to ECR  
✅ ECS staging service is updated with the new image  
✅ Application is accessible via staging Load Balancer URL  
✅ Code changes trigger automatic redeployment to staging  

**Phase 2 - Production:**

✅ Production ECS cluster is deployed  
✅ Pipeline includes manual approval stage  
✅ SNS email notifications are received for approvals  
✅ Staging deployment completes before approval  
✅ Production deployment occurs only after approval  
✅ Both staging and production environments are accessible  

---

## Troubleshooting Common Issues

### Pipeline Fails at Build Stage
- Verify Privileged mode is enabled in CodeBuild
- Check IAM role has ECR permissions
- Review CodeBuild logs for errors

### Pipeline Fails at Deploy Stage
- Verify ECS cluster and service names are correct
- Check that container name in task definition is `app`
- Ensure `imagedefinitions.json` is generated correctly

### Cannot Access Application
- Wait 2-3 minutes for ECS service to stabilize
- Check ECS tasks are in RUNNING state
- Verify security groups allow traffic on port 80

---

## Cleanup Instructions

### Phase 1 - Students:
1. Delete CodePipeline: `ecs-app-pipeline`
2. Delete CodeBuild project: `ecs-app-build`
3. Delete CodeCommit repository: `ecs-app`
4. Empty and delete S3 bucket: `ecs-cicd-artifacts-*`
5. Delete IAM roles: `ecs-codepipeline-role` and `ecs-codebuild-role`

### Phase 1 - Instructor:
```bash
aws cloudformation delete-stack --stack-name ecs-staging --region us-east-1
```

### Phase 2 - Students:
1. Delete enhanced CodePipeline: `ecs-cicd-with-approval-pipeline`
2. Delete SNS topic subscription (check email)
3. Delete CodeBuild project
4. Delete CodeCommit repository
5. Empty and delete S3 bucket
6. Delete IAM roles

### Phase 2 - Instructor:
```bash
# Delete production infrastructure
aws cloudformation delete-stack --stack-name ecs-production --region us-east-1

# Delete staging infrastructure
aws cloudformation delete-stack --stack-name ecs-staging --region us-east-1
```

---

## Additional Resources

- [AWS CodePipeline Documentation](https://docs.aws.amazon.com/codepipeline/)
- [AWS CodeBuild Documentation](https://docs.aws.amazon.com/codebuild/)
- [Amazon ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [AWS CodeCommit Documentation](https://docs.aws.amazon.com/codecommit/)

---

## Support

For questions or issues during the exercise:
1. Check the Troubleshooting section in `CICD-HANDSON-CONSOLE.md`
2. Review AWS service logs (CodeBuild, ECS)
3. Consult with the instructor

---

**Happy Learning! 🚀**
