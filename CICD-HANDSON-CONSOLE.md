# ECS CI/CD Hands-on - AWS Console Instructions

## Overview
In this hands-on exercise, you will create a complete CI/CD pipeline for ECS using the AWS Console. The base ECS infrastructure is already deployed.

## What You'll Build
- **CodeCommit Repository** for source control
- **CodeBuild Project** to build Docker images
- **CodePipeline** to orchestrate the CI/CD workflow
- Automatic deployment to ECS on code commits

---

## Prerequisites (Already Deployed)
The instructor has deployed the `ecs-staging` CloudFormation stack with:
- VPC and Subnets
- Application Load Balancer
- ECS Fargate Cluster and Service (tagged as **staging**)
- ECR Repository

**Deployment command used by instructor:**
```bash
aws cloudformation create-stack \
  --stack-name ecs-staging \
  --template-body file://ecs-base-infrastructure.yaml \
  --parameters ParameterKey=Environment,ParameterValue=staging \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

### Get Base Stack Information
1. Go to **CloudFormation** console
2. Click on stack `ecs-staging`
3. Click **Outputs** tab
4. **Note down these values** (you'll need them):
   - `Environment` → `staging`
   - `ECSClusterName` (e.g., `ecs-staging-cluster`)
   - `ECSServiceName` (e.g., `ecs-staging-service`)
   - `ECRRepositoryName` (e.g., `ecs-staging-app`)
   - `ECRRepositoryUri` (e.g., `123456789.dkr.ecr.us-east-1.amazonaws.com/ecs-staging-app`)
   - `LoadBalancerUrl` (e.g., `http://ecs-staging-alb-xxx.us-east-1.elb.amazonaws.com`)

---

## Part 0: Push Initial Image to ECR (Required First Step)

**Important:** The base CloudFormation stack creates an ECS service with a placeholder nginx image. You need to push your application image to ECR and update the service.

### Using CloudShell or Local Terminal

```bash
# Set variables (replace with your values from CloudFormation outputs)
export AWS_ACCOUNT_ID=<YOUR-ACCOUNT-ID>
export AWS_REGION=us-east-1
export ECR_REPO_NAME=ecs-staging-app  # Use the ECRRepositoryName from CloudFormation outputs

# Create a simple Dockerfile
cat > /tmp/Dockerfile << 'EOF'
FROM nginx:alpine
RUN echo '<html><body><h1>ECS Demo App</h1><p>Initial Version</p></body></html>' > /usr/share/nginx/html/index.html
EXPOSE 80
EOF

# Login to ECR
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Build the image
cd /tmp
docker build -t $ECR_REPO_NAME:latest .

# Tag the image
docker tag $ECR_REPO_NAME:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest

# Push to ECR
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest
```

### Update Task Definition to Use ECR Image

Create a new task definition revision that uses your ECR image:

```bash
# Get the current task definition
aws ecs describe-task-definition \
  --task-definition ecs-staging-task \
  --region us-east-1 \
  --query 'taskDefinition' > /tmp/task-def.json

# Update the image in the task definition
cat /tmp/task-def.json | jq --arg IMAGE "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO_NAME:latest" \
  '.containerDefinitions[0].image = $IMAGE | del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)' \
  > /tmp/new-task-def.json

# Register the new task definition
aws ecs register-task-definition \
  --cli-input-json file:///tmp/new-task-def.json \
  --region us-east-1
```

### Update ECS Service to Use New Task Definition

```bash
aws ecs update-service \
  --cluster ecs-staging-cluster \
  --service ecs-staging-service \
  --force-new-deployment \
  --region us-east-1
```

**Wait 2-3 minutes** for tasks to start running. Verify:

```bash
aws ecs describe-services \
  --cluster ecs-staging-cluster \
  --services ecs-staging-service \
  --region us-east-1 \
  --query 'services[0].[runningCount,desiredCount]' \
  --output text
```

You should see `2  2` (2 running, 2 desired).

---

## Part 1: Create CodeCommit Repository

1. Go to **CodeCommit** console
2. Click **Create repository**
3. **Repository name:** `ecs-app`
4. Click **Create**
5. **Note the Clone URL (HTTPS)** from the repository page

---

## Part 2: Create S3 Bucket for Artifacts

1. Go to **S3** console
2. Click **Create bucket**
3. **Bucket name:** `ecs-cicd-artifacts-<YOUR-ACCOUNT-ID>`
   - Replace `<YOUR-ACCOUNT-ID>` with your AWS account ID
4. **Region:** Same as your ECS cluster
5. Scroll down to **Bucket Versioning** → Enable
6. Click **Create bucket**

---

## Part 3: Create IAM Role for CodeBuild

### Create the Role
1. Go to **IAM** console → **Roles**
2. Click **Create role**
3. **Trusted entity type:** AWS service
4. **Use case:** CodeBuild
5. Click **Next**
6. Search and select: `AmazonEC2ContainerRegistryPowerUser`
7. Click **Next**
8. **Role name:** `ecs-codebuild-role`
9. Click **Create role**

### Add Inline Policy
1. Click on the newly created `ecs-codebuild-role`
2. Click **Add permissions** → **Create inline policy**
3. Click **JSON** tab
4. Paste this policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:/aws/codebuild/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::ecs-cicd-artifacts-*/*"
    }
  ]
}
```
5. Click **Next**
6. **Policy name:** `CodeBuildAdditionalPolicy`
7. Click **Create policy**

---

## Part 4: Create CodeBuild Project

1. Go to **CodeBuild** console
2. Click **Create build project**

### Project Configuration
- **Project name:** `ecs-app-build`

### Source
- **Source provider:** AWS CodeCommit
- **Repository:** `ecs-app`
- **Branch:** `main`

### Environment
- **Environment image:** Managed image
- **Operating system:** Amazon Linux
- **Runtime(s):** Standard
- **Image:** `aws/codebuild/standard:7.0` (or latest)
- **Image version:** Always use the latest
- **Privileged:** ✅ **Enable** (required for Docker builds)
- **Service role:** Existing service role
- **Role ARN:** Select `ecs-codebuild-role`

### Environment Variables
Add these variables (click **Add environment variable** for each):

| Name | Value | Type |
|------|-------|------|
| `AWS_ACCOUNT_ID` | Your AWS Account ID | Plaintext |
| `AWS_DEFAULT_REGION` | Your region (e.g., us-east-1) | Plaintext |
| `IMAGE_REPO_NAME` | Value from CloudFormation Outputs | Plaintext |
| `IMAGE_TAG` | `latest` | Plaintext |

### Buildspec
- **Build specifications:** Insert build commands
- Click **Switch to editor**
- Paste this buildspec:
```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
      - printf '[{"name":"app","imageUri":"%s"}]' $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
```

### Artifacts
- **Type:** No artifacts (we'll configure this in CodePipeline)

### Logs
- Keep defaults

Click **Create build project**

---

## Part 5: Create IAM Role for CodePipeline

### Create the Role
1. Go to **IAM** console → **Roles**
2. Click **Create role**
3. **Trusted entity type:** AWS service
4. **Use case:** CodePipeline
5. Click **Next**
6. Click **Next** (default policy is attached)
7. **Role name:** `ecs-codepipeline-role`
8. Click **Create role**

### Add Inline Policy
1. Click on `ecs-codepipeline-role`
2. Click **Add permissions** → **Create inline policy**
3. Click **JSON** tab
4. Paste this policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::ecs-cicd-artifacts-*",
        "arn:aws:s3:::ecs-cicd-artifacts-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:*",
        "iam:PassRole"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codecommit:GetBranch",
        "codecommit:GetCommit",
        "codecommit:UploadArchive",
        "codecommit:GetUploadArchiveStatus",
        "codecommit:CancelUploadArchive"
      ],
      "Resource": "*"
    }
  ]
}
```
5. Click **Next**
6. **Policy name:** `CodePipelineAdditionalPolicy`
7. Click **Create policy**

---

## Part 6: Create CodePipeline

1. Go to **CodePipeline** console
2. Click **Create pipeline**

### Pipeline Settings
- **Pipeline name:** `ecs-app-pipeline`
- **Service role:** Existing service role
- **Role name:** `ecs-codepipeline-role`
- **Artifact store:** Custom location
- **Bucket:** `ecs-cicd-artifacts-<YOUR-ACCOUNT-ID>`
- Click **Next**

### Source Stage
- **Source provider:** AWS CodeCommit
- **Repository name:** `ecs-app`
- **Branch name:** `main`
- **Change detection options:** Amazon CloudWatch Events (recommended)
- Click **Next**

### Build Stage
- **Build provider:** AWS CodeBuild
- **Region:** Your region
- **Project name:** `ecs-app-build`
- Click **Next**

### Deploy Stage
- **Deploy provider:** Amazon ECS
- **Region:** Your region
- **Cluster name:** Enter the value from CloudFormation Outputs (e.g., `ecs-staging-cluster`)
- **Service name:** Enter the value from CloudFormation Outputs (e.g., `ecs-staging-service`)
- **Image definitions file:** `imagedefinitions.json`
- Click **Next**

### Review
- Review all settings
- Click **Create pipeline**

The pipeline will be created but will fail initially (no code in repository yet).

---

## Part 7: Add Application Code to CodeCommit

### Configure Git Credentials (One-time setup)
Open **CloudShell** or your local terminal:

```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

### Clone Repository
```bash
git clone https://git-codecommit.<YOUR-REGION>.amazonaws.com/v1/repos/ecs-app
cd ecs-app
```

### Create Application Files

**Create `Dockerfile`:**
```bash
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
EOF
```

**Create `index.html`:**
```bash
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
    <h1>Hello from ECS CI/CD Pipeline!</h1>
    <p>Version 1.0</p>
    <p>Deployed via AWS CodePipeline</p>
</body>
</html>
EOF
```

### Commit and Push
```bash
git add .
git commit -m "Initial commit - ECS application"
git push origin main
```

---

## Part 8: Monitor Pipeline Execution

1. Go to **CodePipeline** console
2. Click on `ecs-app-pipeline`
3. Watch the pipeline execute through three stages:
   - **Source:** Pulls code from CodeCommit ✅
   - **Build:** Builds Docker image and pushes to ECR ✅
   - **Deploy:** Updates ECS service ✅

**This will take 5-10 minutes.**

### View Build Logs (Optional)
1. Click on **Details** in the Build stage
2. Click **Link to execution details**
3. View the build logs in CodeBuild

---

## Part 9: Verify Deployment

1. Go to **CloudFormation** console
2. Click on `ecs-base` stack → **Outputs** tab
3. Copy the `LoadBalancerUrl`
4. Open the URL in your browser
5. You should see: **"Hello from ECS CI/CD Pipeline!"**

---

## Part 10: Test CI/CD Workflow

### Make a Code Change
```bash
cd ecs-app
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>ECS CI/CD Demo</title>
    <style>
        body { font-family: Arial; text-align: center; padding: 50px; background: #f0f0f0; }
        h1 { color: #232F3E; }
    </style>
</head>
<body>
    <h1>Hello from ECS CI/CD Pipeline!</h1>
    <p>Version 2.0 - Updated!</p>
    <p>CI/CD is working! 🚀</p>
</body>
</html>
EOF
```

### Commit and Push
```bash
git add index.html
git commit -m "Update to version 2.0"
git push origin main
```

### Watch Automatic Deployment
1. Go to **CodePipeline** console
2. Watch `ecs-app-pipeline` automatically trigger
3. Wait for all stages to complete
4. Refresh the Load Balancer URL
5. See the updated content!

---

## Cleanup

### Delete CI/CD Resources
1. **CodePipeline:** Delete `ecs-app-pipeline`
2. **CodeBuild:** Delete `ecs-app-build` project
3. **CodeCommit:** Delete `ecs-app` repository
4. **S3:** Empty and delete `ecs-cicd-artifacts-*` bucket
5. **IAM Roles:** Delete `ecs-codepipeline-role` and `ecs-codebuild-role`

### Delete Base Infrastructure (Instructor)
```bash
aws cloudformation delete-stack --stack-name ecs-staging --region us-east-1
```

---

## Troubleshooting

### Pipeline Fails at Build Stage
- **Check:** CodeBuild logs (click Details in Build stage)
- **Verify:** Privileged mode is enabled in CodeBuild
- **Verify:** IAM role has ECR permissions
- **Check:** Environment variables are set correctly

### Pipeline Fails at Deploy Stage
- **Verify:** ECS cluster and service names match CloudFormation outputs
- **Check:** `imagedefinitions.json` exists in build artifacts
- **Verify:** Container name in task definition is `app`

### Cannot Clone CodeCommit Repository
- **Verify:** Git credential helper is configured
- **Check:** IAM user/role has CodeCommit permissions
- **Try:** Using HTTPS URL (not SSH)

### Application Not Accessible
- **Wait:** ECS service takes 2-3 minutes to update
- **Check:** ECS service tasks are running (ECS console)
- **Verify:** Security groups allow traffic on port 80

---

## What You Learned

✅ Creating a CodeCommit repository  
✅ Building Docker images with CodeBuild  
✅ Creating a complete CI/CD pipeline with CodePipeline  
✅ Automating ECS deployments  
✅ Configuring IAM roles for CI/CD services  
✅ Using CloudWatch Events to trigger pipelines  

**Congratulations! You've successfully created an ECS CI/CD pipeline!** 🎉
