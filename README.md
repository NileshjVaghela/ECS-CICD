# ECS CI/CD Hands-on Exercise

## Part 1: Base Infrastructure (Pre-deployed by Instructor)

Deploy the base ECS infrastructure using CloudFormation:

```bash
aws cloudformation create-stack \
  --stack-name ecs-base \
  --template-body file://ecs-base-infrastructure.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Wait for completion (~10 minutes):
```bash
aws cloudformation wait stack-create-complete --stack-name ecs-base --region us-east-1
```

This creates:
- VPC with 2 public subnets
- Application Load Balancer
- ECS Fargate Cluster and Service
- ECR Repository
- Security Groups
- IAM Roles

---

## Part 2: CI/CD Pipeline (Student Hands-on)

Students will create the CI/CD pipeline using **AWS Console only**.

Follow the detailed instructions in: **CICD-HANDSON-CONSOLE.md**

---

## Architecture

```
CodeCommit → CodePipeline → CodeBuild → ECR → ECS Service
                                              ↓
                                         Load Balancer
```

---

## Files Provided

1. **ecs-base-infrastructure.yaml** - CloudFormation template for base ECS setup
2. **CICD-HANDSON-CONSOLE.md** - Step-by-step AWS Console instructions for CI/CD
