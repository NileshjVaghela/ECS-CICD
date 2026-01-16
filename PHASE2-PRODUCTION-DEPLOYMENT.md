# Phase 2: Production Deployment with Manual Approval

## Overview

In Phase 2, you'll extend the CI/CD pipeline to deploy to both **Staging** and **Production** environments with manual approval between them.

**Pipeline Flow:**
```
Source → Build → Deploy to Staging → Manual Approval → Deploy to Production
```

---

## Prerequisites

Complete Phase 1 with staging environment deployed.

---

## Step 1: Deploy Production Infrastructure

Deploy a second ECS cluster for production using the same template with different environment parameter:

```bash
aws cloudformation create-stack \
  --stack-name ecs-production \
  --template-body file://ecs-base-infrastructure.yaml \
  --parameters ParameterKey=Environment,ParameterValue=production \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**Note:** The `Environment` parameter tags resources as `production`. The staging stack was deployed with `Environment=staging` (default value).

Wait for completion:
```bash
aws cloudformation wait stack-create-complete --stack-name ecs-production --region us-east-1
```

**Verify both stacks:**
```bash
# List both stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --query 'StackSummaries[?contains(StackName, `ecs-`)].{Name:StackName,Status:StackStatus}' \
  --output table
```

You should see:
- `ecs-staging` (deployed in Phase 1)
- `ecs-production` (just deployed)

---

## Step 2: Push Initial Image to Production ECR

```bash
export AWS_ACCOUNT_ID=<YOUR-ACCOUNT-ID>
export AWS_REGION=us-east-1
export ECR_REPO_NAME=ecs-production-app

# Login to ECR
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Tag and push the same image to production ECR
docker tag ecs-staging-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
```

Update production task definition and service (same as staging).

---

## Step 3: Delete Existing CI/CD Stack

Delete the Phase 1 CI/CD stack:

```bash
aws cloudformation delete-stack --stack-name ecs-cicd --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name ecs-cicd --region us-east-1
```

---

## Step 4: Deploy New CI/CD Pipeline with Approval

Deploy the enhanced pipeline:

```bash
aws cloudformation create-stack \
  --stack-name ecs-cicd-with-approval \
  --template-body file://ecs-cicd-with-approval.yaml \
  --parameters \
    ParameterKey=StagingStackName,ParameterValue=ecs-staging \
    ParameterKey=ProductionStackName,ParameterValue=ecs-production \
    ParameterKey=RepositoryName,ParameterValue=ecs-app \
    ParameterKey=BranchName,ParameterValue=main \
    ParameterKey=ApprovalEmail,ParameterValue=<YOUR-EMAIL> \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

**Important:** Check your email and confirm the SNS subscription!

---

## Step 5: Test the Pipeline

### Make a Code Change

```bash
cd ecs-app
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>ECS CI/CD Demo</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 50px; }
        h1 { color: #FF9900; }
    </style>
</head>
<body>
    <h1>Multi-Environment Deployment!</h1>
    <p>Version 3.0</p>
    <p>Staging → Approval → Production</p>
</body>
</html>
EOF

git add index.html
git commit -m "Test multi-environment deployment"
git push origin main
```

### Monitor the Pipeline

1. Go to **CodePipeline** console
2. Watch the pipeline execute:
   - ✅ **Source** - Pulls code
   - ✅ **Build** - Builds Docker image
   - ✅ **Deploy-Staging** - Deploys to staging
   - ⏸️ **Approval** - Waits for manual approval
   - ⏳ **Deploy-Production** - Pending approval

### Verify Staging Deployment

Get staging URL:
```bash
aws cloudformation describe-stacks \
  --stack-name ecs-staging \
  --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerUrl`].OutputValue' \
  --output text
```

Open in browser and verify the changes.

### Approve Production Deployment

**Option 1: Via Email**
- Check your email for approval notification
- Click the approval link
- Review and approve

**Option 2: Via Console**
1. Go to **CodePipeline** console
2. Click on the pipeline
3. In the **Approval** stage, click **Review**
4. Add comments (optional)
5. Click **Approve**

### Verify Production Deployment

Get production URL:
```bash
aws cloudformation describe-stacks \
  --stack-name ecs-production \
  --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerUrl`].OutputValue' \
  --output text
```

Open in browser and verify the changes are deployed.

---

## Architecture

```
┌─────────────┐
│ CodeCommit  │
└──────┬──────┘
       │
       ↓
┌─────────────────────────────────────────────────┐
│              CodePipeline                       │
│                                                 │
│  Source → Build → Deploy Staging → Approval    │
│                                        ↓        │
│                              Deploy Production  │
└─────────────────────────────────────────────────┘
       │                                  │
       ↓                                  ↓
┌──────────────┐                  ┌──────────────┐
│   Staging    │                  │ Production   │
│ ECS Cluster  │                  │ ECS Cluster  │
└──────────────┘                  └──────────────┘
```

---

## Key Differences from Phase 1

| Feature | Phase 1 | Phase 2 |
|---------|---------|---------|
| Environments | Staging only | Staging + Production |
| Approval | None | Manual approval required |
| Notifications | None | SNS email notifications |
| Deployment | Automatic | Staging auto, Production manual |

---

## Best Practices

✅ **Always test in staging first** before approving production  
✅ **Review CloudWatch logs** for any errors in staging  
✅ **Use meaningful commit messages** for audit trail  
✅ **Set up monitoring** for both environments  
✅ **Document approval decisions** in the approval comments  

---

## Troubleshooting

### Approval Email Not Received
- Check spam folder
- Verify email address in CloudFormation parameters
- Check SNS subscription status in SNS console

### Pipeline Stuck at Approval
- Approval expires after 7 days by default
- Reject and restart pipeline if needed

### Production Deployment Fails
- Verify production ECS service is healthy
- Check production ECR has the image
- Review CodePipeline execution logs

---

## Cleanup

```bash
# Delete CI/CD pipeline
aws cloudformation delete-stack --stack-name ecs-cicd-with-approval --region us-east-1

# Delete production infrastructure
aws cloudformation delete-stack --stack-name ecs-production --region us-east-1

# Delete staging infrastructure
aws cloudformation delete-stack --stack-name ecs-staging --region us-east-1
```

---

**Congratulations! You've implemented a production-grade CI/CD pipeline with manual approval! 🎉**
