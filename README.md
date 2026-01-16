# ECS CI/CD Hands-on Exercise - Complete Guide

## Overview

This hands-on exercise teaches students how to create a complete CI/CD pipeline for Amazon ECS using AWS CodePipeline. The exercise is divided into two parts:

- **Part 1:** Base ECS infrastructure (pre-deployed by instructor)
- **Part 2:** CI/CD pipeline creation (student hands-on using AWS Console)

---

## Exercise Structure

### Part 1: Base Infrastructure Setup (Instructor)

The instructor deploys the base ECS infrastructure using CloudFormation. This includes:

- **VPC** with 2 public subnets across availability zones
- **Application Load Balancer** with target group and listener
- **ECS Fargate Cluster** for running containers
- **ECS Service** with task definition
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

---

### Part 2: CI/CD Pipeline (Student Hands-on)

Students create a complete CI/CD pipeline using the **AWS Console only**. No CloudFormation or CLI required for this part.

**What Students Will Build:**

1. **CodeCommit Repository** - AWS-managed Git repository for source control
2. **S3 Artifact Bucket** - Storage for pipeline artifacts
3. **IAM Roles** - Permissions for CodeBuild and CodePipeline
4. **CodeBuild Project** - Builds Docker images and pushes to ECR
5. **CodePipeline** - Orchestrates the entire CI/CD workflow

**Pipeline Flow:**
```
CodeCommit (Source) → CodeBuild (Build Docker Image) → ECS (Deploy)
```

**File:** `CICD-HANDSON-CONSOLE.md`

---

## Learning Objectives

By completing this exercise, students will learn:

✅ How to set up AWS CodeCommit for source control  
✅ How to configure CodeBuild to build Docker images  
✅ How to push Docker images to Amazon ECR  
✅ How to create a CodePipeline with multiple stages  
✅ How to configure IAM roles for CI/CD services  
✅ How to automate ECS deployments  
✅ How to use CloudWatch Events to trigger pipelines  
✅ How to troubleshoot CI/CD pipeline issues  

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

- **Part 1 (Instructor):** 10-15 minutes (CloudFormation deployment)
- **Part 2 (Students):** 60-90 minutes (hands-on exercise)

---

## Architecture Diagram

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
│  │ Deploy │ │ ← Updates ECS Service
│  └────────┘ │
└─────────────┘
       │
       ↓
┌─────────────┐      ┌─────────────┐
│     ECR     │─────→│ ECS Service │
│ (Container  │      │  (Fargate)  │
│   Images)   │      └──────┬──────┘
└─────────────┘             │
                            ↓
                   ┌─────────────────┐
                   │ Load Balancer   │
                   │ (Public Access) │
                   └─────────────────┘
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

### For Students:

1. Get the base stack outputs from your instructor
2. Follow the instructions in `CICD-HANDSON-CONSOLE.md`
3. Create the CI/CD pipeline using AWS Console
4. Test the pipeline by pushing code changes

**For Phase 2 (Production Deployment):**
- Follow `PHASE2-PRODUCTION-DEPLOYMENT.md` to add production environment with manual approval

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

Students successfully complete the exercise when:

✅ CodePipeline is created with Source, Build, and Deploy stages  
✅ Pipeline automatically triggers on code commits  
✅ Docker image is built and pushed to ECR  
✅ ECS service is updated with the new image  
✅ Application is accessible via Load Balancer URL  
✅ Code changes trigger automatic redeployment  

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

### Students:
1. Delete CodePipeline: `ecs-app-pipeline`
2. Delete CodeBuild project: `ecs-app-build`
3. Delete CodeCommit repository: `ecs-app`
4. Empty and delete S3 bucket: `ecs-cicd-artifacts-*`
5. Delete IAM roles: `ecs-codepipeline-role` and `ecs-codebuild-role`

### Instructor:
```bash
aws cloudformation delete-stack --stack-name ecs-base --region us-east-1
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
