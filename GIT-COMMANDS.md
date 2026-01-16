# Git Commands for ECS-CICD Repository

## Initial Setup

### 1. Create directory and clone repository
```bash
mkdir -p /home/nilesh/ecs-cicd
cd /home/nilesh/ecs-cicd
git clone https://github.com/<YOUR-GITHUB-USERNAME>/ECS-CICD.git .
```

### 2. Copy hands-on files to repository
```bash
cp /home/nilesh/README.md \
   /home/nilesh/ecs-base-infrastructure.yaml \
   /home/nilesh/ecs-cicd-handson.yaml \
   /home/nilesh/ecs-cicd-with-approval.yaml \
   /home/nilesh/CICD-HANDSON-CONSOLE.md \
   /home/nilesh/PHASE2-PRODUCTION-DEPLOYMENT.md \
   /home/nilesh/EXERCISE-OVERVIEW.md \
   /home/nilesh/ecs-cicd/
```

### 3. Add files to git
```bash
cd /home/nilesh/ecs-cicd
git add .
```

### 4. Commit changes
```bash
git commit -m "Add ECS CI/CD hands-on exercise files

- Base infrastructure CloudFormation template with environment parameter
- Phase 1: CI/CD pipeline with staging deployment
- Phase 2: Multi-environment pipeline with manual approval
- Complete console instructions for students
- Documentation and overview guides"
```

### 5. Configure git credentials (if needed)
```bash
# Option 1: Use credential helper
git config --global credential.helper store

# Option 2: Set remote URL with token
git remote set-url origin https://<YOUR-GITHUB-USERNAME>:<YOUR-PERSONAL-ACCESS-TOKEN>@github.com/<YOUR-GITHUB-USERNAME>/ECS-CICD.git
```

### 6. Push to GitHub
```bash
git push origin main
```

---

## Updating Files Later

### 1. Make changes to files
```bash
cd /home/nilesh/ecs-cicd
# Edit files as needed
```

### 2. Check status
```bash
git status
```

### 3. Add modified files
```bash
git add <filename>
# Or add all changes
git add .
```

### 4. Commit changes
```bash
git commit -m "Description of changes"
```

### 5. Push to GitHub
```bash
git push origin main
```

---

## Useful Git Commands

### View commit history
```bash
git log --oneline
```

### View changes
```bash
git diff
```

### Pull latest changes
```bash
git pull origin main
```

### Check remote URL
```bash
git remote -v
```

### Change remote URL
```bash
git remote set-url origin https://github.com/<YOUR-GITHUB-USERNAME>/ECS-CICD.git
```

---

## GitHub Personal Access Token Setup

1. Go to https://github.com/settings/tokens
2. Click **Generate new token** → **Generate new token (classic)**
3. Give it a name: `ECS-CICD-Repo`
4. Select scopes: **repo** (full control of private repositories)
5. Click **Generate token**
6. Copy the token (you won't see it again!)
7. Use the token as password when pushing to GitHub

---

## Notes

- Replace `<YOUR-GITHUB-USERNAME>` with your actual GitHub username
- Replace `<YOUR-PERSONAL-ACCESS-TOKEN>` with your GitHub personal access token
- Never commit tokens or credentials to the repository
- Use `.gitignore` to exclude sensitive files
