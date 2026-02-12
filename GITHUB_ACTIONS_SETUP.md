# GitHub Actions Deployment Setup Guide

## Overview
This workflow automates the deployment of your Spring Boot application to AWS EKS using:
- GitHub Actions for CI/CD
- AWS IAM OIDC for authentication (no access keys needed)
- Amazon ECR for Docker image storage
- Helm for Kubernetes deployment

## Prerequisites Setup

### 1. Create ECR Repository
```bash
aws ecr create-repository \
  --repository-name packersmovers-app \
  --region us-east-2
```

### 2. Configure GitHub OIDC with AWS IAM

#### Step 2.1: Create OIDC Identity Provider in AWS
1. Go to AWS IAM Console → Identity Providers → Add Provider
2. Provider Type: OpenID Connect
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click "Add provider"

#### Step 2.2: Create IAM Role for GitHub Actions
Create a trust policy file `github-trust-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/YOUR_REPO_NAME:*"
        }
      }
    }
  ]
}
```

Create the role:
```bash
aws iam create-role \
  --role-name GitHubActionsEKSRole \
  --assume-role-policy-document file://github-trust-policy.json
```

#### Step 2.3: Attach Required Policies to the Role
```bash
# ECR permissions
aws iam attach-role-policy \
  --role-name GitHubActionsEKSRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

# EKS permissions
aws iam attach-role-policy \
  --role-name GitHubActionsEKSRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

Create a custom policy for EKS access `eks-access-policy.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

```bash
aws iam create-policy \
  --policy-name EKSAccessPolicy \
  --policy-document file://eks-access-policy.json

aws iam attach-role-policy \
  --role-name GitHubActionsEKSRole \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/EKSAccessPolicy
```

### 3. Update EKS ConfigMap for kubectl Access

Add the IAM role to your EKS cluster's aws-auth ConfigMap:

```bash
kubectl edit configmap aws-auth -n kube-system
```

Add this under `mapRoles`:
```yaml
- rolearn: arn:aws:iam::YOUR_ACCOUNT_ID:role/GitHubActionsEKSRole
  username: github-actions
  groups:
    - system:masters
```

Or use this command:
```bash
eksctl create iamidentitymapping \
  --cluster your-eks-cluster-name \
  --region us-east-2 \
  --arn arn:aws:iam::YOUR_ACCOUNT_ID:role/GitHubActionsEKSRole \
  --username github-actions \
  --group system:masters
```

### 4. Update GitHub Actions Workflow

Edit `.github/workflows/deploy.yml` and replace:
- `YOUR_ACCOUNT_ID` with your AWS account ID
- `YOUR_GITHUB_ACTIONS_ROLE` with `GitHubActionsEKSRole` (or your role name)
- `your-eks-cluster-name` with your actual EKS cluster name

Example:
```yaml
role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsEKSRole
```

```yaml
env:
  AWS_REGION: us-east-2
  EKS_CLUSTER_NAME: my-production-cluster
  ECR_REPOSITORY: packersmovers-app
```

## Workflow Explanation

### What the workflow does:

1. **Checkout code**: Gets your repository code
2. **Set up JDK 17**: Installs Java 17 (Amazon Corretto)
3. **Build with Maven**: Compiles your Spring Boot app and creates the JAR file
4. **Configure AWS credentials**: Uses OIDC to assume the IAM role (no secrets needed!)
5. **Login to ECR**: Authenticates with your ECR registry
6. **Build Docker image**: Creates a Docker image using your multi-stage Dockerfile
7. **Push to ECR**: Pushes the image with both commit SHA tag and 'latest' tag
8. **Install kubectl & Helm**: Sets up Kubernetes tools
9. **Update kubeconfig**: Configures kubectl to connect to your EKS cluster
10. **Deploy with Helm**: Upgrades or installs your app using Helm chart
11. **Verify deployment**: Shows the deployed pods and services

### Key Features:

- **No AWS credentials in GitHub**: Uses OIDC federation for secure authentication
- **Automatic versioning**: Each deployment is tagged with the git commit SHA
- **Rolling updates**: Helm manages zero-downtime deployments
- **Idempotent**: Running multiple times is safe (uses `helm upgrade --install`)

## Changes Made to Your Repository

### 1. `helm-chart/values.yaml`
- Removed Harness-specific placeholders (`<+pipeline...>`)
- Set default values:
  - `replicaCount: 2`
  - `containerPort: 8080`
  - `containerName: packersmovers-app`
- Image repository will be overridden during deployment

### 2. `helm-chart/templates/deployment.yaml`
- Added image tag support: `{{ .Values.image.tag | default "latest" }}`
- This allows the workflow to specify which image version to deploy

### 3. `.github/workflows/deploy.yml` (NEW)
- Complete CI/CD pipeline
- Builds, tests, containerizes, and deploys your app

## Testing the Deployment

1. Push your code to the `main` branch
2. Go to GitHub → Actions tab
3. Watch the workflow run
4. Once complete, verify:
```bash
kubectl get pods -n default
kubectl get svc -n default
kubectl logs -l app=packersmovers-app -n default
```

## Accessing Your Application

Since the service is ClusterIP (internal only), you have options:

### Option 1: Port Forward (for testing)
```bash
kubectl port-forward svc/packersmovers-app-service 8080:8080 -n default
```
Then access: http://localhost:8080

### Option 2: Change to LoadBalancer (for production)
Edit `helm-chart/templates/service.yaml`:
```yaml
spec:
  type: LoadBalancer  # Changed from ClusterIP
```

After deployment, get the external URL:
```bash
kubectl get svc packersmovers-app-service -n default
```

### Option 3: Use Ingress (recommended for production)
Set up an Ingress controller and configure ingress rules in your Helm chart.

## Troubleshooting

### Issue: "error: You must be logged in to the server (Unauthorized)"
- Check that the IAM role is added to aws-auth ConfigMap
- Verify the role ARN in the workflow matches your IAM role

### Issue: "Error: INSTALLATION FAILED: Kubernetes cluster unreachable"
- Ensure EKS cluster name is correct
- Check that the IAM role has EKS describe permissions

### Issue: Docker build fails
- Check that the JAR file name in Dockerfile matches pom.xml version
- Current: `customer-1.0.1.jar`

### Issue: Pods in CrashLoopBackOff
- Check database connectivity (RDS endpoint in application.properties)
- View logs: `kubectl logs -l app=packersmovers-app -n default`

## Security Notes

- Database credentials are currently in `application.properties` (not secure!)
- Consider using AWS Secrets Manager or Kubernetes Secrets
- The workflow uses IAM roles, which is more secure than access keys
- Image tags use commit SHA for traceability

## Next Steps

1. Set up proper secrets management for database credentials
2. Configure health checks in deployment.yaml
3. Add resource limits and requests
4. Set up monitoring and logging
5. Configure autoscaling if needed
