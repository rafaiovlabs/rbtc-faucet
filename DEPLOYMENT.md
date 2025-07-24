# GitHub Workflows for ECR and Kubernetes Deployment

This repository contains GitHub Actions workflows that automatically build Docker images, push them to Amazon ECR, and deploy them to Kubernetes clusters.

## Workflows

### 1. `deploy.yml` - Complete inline deployment
This workflow creates Kubernetes manifests inline and deploys them.

### 2. `deploy-with-manifests.yml` - Deployment using separate manifest files
This workflow uses the Kubernetes manifest files in the `k8s/` directory for deployment.

## Required GitHub Secrets

Before using these workflows, you need to set up the following secrets in your GitHub repository:

### AWS Secrets
- `AWS_ACCESS_KEY_ID` - AWS access key with ECR and EKS permissions
- `AWS_SECRET_ACCESS_KEY` - AWS secret access key
- `ECR_REGISTRY` - Your ECR registry URL (e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com`)
- `EKS_CLUSTER_NAME` - Name of your EKS cluster
TODO:
CHANGE FOR SSO IMPLEMENTATION

### Setting up GitHub Secrets

1. Go to your GitHub repository
2. Navigate to Settings → Secrets and variables → Actions
3. Click "New repository secret"
4. Add each secret with the appropriate values

## AWS IAM Permissions

The AWS user/role used by GitHub Actions needs the following permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage"
            ],
            "Resource": "*"
        },
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

## ECR Repository Setup

1. Create an ECR repository named `rbtc-faucet`:
```bash
aws ecr create-repository --repository-name rbtc-faucet --region us-east-1
```

2. Note the repository URI for the `ECR_REGISTRY` secret.

## Kubernetes Cluster Setup

### Prerequisites
- EKS cluster running
- ALB Ingress Controller installed
- cert-manager installed (for SSL certificates)

### Install ALB Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/controller.yaml
```

### Install cert-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

## Workflow Triggers

### Automatic Triggers
- **Push to `main` branch**: Builds and deploys to production
- **Push to `develop` branch**: Builds and deploys to development
- **Pull requests to `main`**: Builds only (no deployment)

### Manual Triggers
- **Workflow dispatch**: Allows manual deployment to a specific environment

## Environment Configuration

The workflows support multiple environments:
- `dev` - Development environment
- `prod` - Production environment

### Domain Configuration

Update the ingress manifest or the inline YAML in the workflow to use your actual domain:

```yaml
spec:
  tls:
  - hosts:
    - rbtc-faucet-dev.yourdomain.com  # Change this
    secretName: rbtc-faucet-dev-tls
  rules:
  - host: rbtc-faucet-dev.yourdomain.com  # Change this
```

## Kubernetes Manifests

The `k8s/` directory contains:

- `deployment.yaml` - Application deployment configuration
- `service.yaml` - Service configuration
- `ingress.yaml` - Ingress configuration for external access
- `configmap.yaml` - ConfigMap for application configuration
- `secret.yaml.example` - Example secret manifest (DO NOT commit actual secrets)

### Environment Variable Substitution

The manifest files use environment variable substitution:
- `${ENVIRONMENT}` - Replaced with the target environment (dev/prod)
- `${IMAGE_URI}` - Replaced with the built Docker image URI

## Customization

### Modifying Resource Limits
Update the `resources` section in `deployment.yaml`:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Adding Environment Variables
Add environment variables to the deployment:

```yaml
env:
- name: NODE_ENV
  value: "production"
- name: CUSTOM_VAR
  value: "custom-value"
```

### Adding Secrets
1. Create secrets in Kubernetes:
```bash
kubectl create secret generic rbtc-faucet-secrets \
  --from-literal=DATABASE_URL="your-database-url" \
  --from-literal=API_KEY="your-api-key" \
  --namespace=rbtc-faucet-dev
```

2. Reference in deployment:
```yaml
envFrom:
- secretRef:
    name: rbtc-faucet-secrets
```

## Monitoring and Debugging

### Check deployment status
```bash
kubectl get pods -n rbtc-faucet-dev
kubectl describe deployment rbtc-faucet -n rbtc-faucet-dev
kubectl logs -f deployment/rbtc-faucet -n rbtc-faucet-dev
```

### Check ingress
```bash
kubectl get ingress -n rbtc-faucet-dev
kubectl describe ingress rbtc-faucet-ingress -n rbtc-faucet-dev
```

### Check events
```bash
kubectl get events -n rbtc-faucet-dev --sort-by='.lastTimestamp'
```

## Security Considerations

1. **Never commit secrets to Git**
2. **Use least privilege IAM policies**
3. **Regularly rotate AWS credentials**
4. **Enable ECR image scanning**
5. **Use network policies in Kubernetes**
6. **Keep kubectl and cluster versions updated**

## Troubleshooting

### Common Issues

1. **ECR Login Failed**
   - Check AWS credentials
   - Verify ECR repository exists
   - Check IAM permissions

2. **Deployment Failed**
   - Check image URI is correct
   - Verify namespace exists
   - Check resource quotas

3. **Ingress Not Working**
   - Verify ALB Ingress Controller is running
   - Check DNS configuration
   - Verify SSL certificate status

### Debug Commands
```bash
# Check workflow logs in GitHub Actions UI
# Check pod logs
kubectl logs -f deployment/rbtc-faucet -n rbtc-faucet-dev

# Check ingress controller logs
kubectl logs -f -n kube-system deployment/aws-load-balancer-controller
```
