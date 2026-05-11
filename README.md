# EKS + External Secrets Operator + AWS Secrets Manager

A complete reference implementation for syncing secrets from AWS Secrets Manager into Kubernetes using External Secrets Operator (ESO) with IRSA on Amazon EKS.

## Overview

This repository documents and provides manifests for a production-ready setup where:

1. Secrets are stored in **AWS Secrets Manager**
2. **External Secrets Operator** runs in your EKS cluster
3. ESO authenticates to AWS using **IRSA** (IAM Roles for Service Accounts)
4. ESO automatically syncs secrets into Kubernetes `Secret` objects
5. Your applications consume those secrets as environment variables or mounted files

## Prerequisites

Before starting, ensure you have:

- ✅ AWS CLI installed and configured
- ✅ kubectl installed and configured
- ✅ helm installed
- ✅ An EKS cluster already created
- ✅ kubectl access to the cluster working
- ✅ IAM permissions to create roles, policies, and OIDC providers
- ✅ eksctl installed (for OIDC provider setup)

See [docs/01-prerequisites.md](docs/01-prerequisites.md) for detailed setup.

## Quick start

This is the high-level flow. For detailed steps, see the documentation chapters.

1. Create a secret in AWS Secrets Manager
```bash
aws secretsmanager create-secret \
  --name prod/frontend \
  --secret-string '{"API_KEY":"my-api-key","DB_PASSWORD":"my-password"}' \
  --region us-east-1
```
2. Create IAM policy and role
Create a restrictive IAM policy (see iam/eso-policy.json)
Create an IAM role with that policy
Add trust policy for IRSA (see iam/trust-policy.json)

3. Associate OIDC provider
```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster my-cluster \
  --approve
```

4. Install External Secrets Operator
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<AWS_ACCOUNT_ID>:role/ESORole
```
5. Create ClusterSecretStore
```bash
kubectl apply -f manifests/01-secret-store/cluster-secret-store.yaml
```
6. Create ExternalSecret
```bash
kubectl apply -f manifests/02-external-secret/external-secret.yaml
```
7. Verify the Kubernetes Secret was created
```bash
kubectl get secret frontend-k8s-secret -n default
kubectl get secret frontend-k8s-secret -n default -o jsonpath="{.data.API_KEY}" | base64 -d
```
8. Consume in your Deployment
Choose one:

Option A: Environment variables
```bash
kubectl apply -f manifests/03-deployment/deployment-envvar.yaml
```
Option B: Mounted files
```bash
kubectl apply -f manifests/03-deployment/deployment-volume.yaml
```
##Step-by-step documentation
For detailed explanations and troubleshooting, see:

1. Prerequisites — what you need before starting
2. Create AWS Secret — store secret in AWS Secrets Manager
3. IAM Policy and Role — create restrictive IAM permissions
4. IRSA and OIDC — set up IRSA trust chain
5. Install ESO — install External Secrets Operator
6. ClusterSecretStore — configure ESO provider access
7. ExternalSecret — define what to sync
8. Consume Secret — use secret in Deployment
9. Troubleshooting — common issues and solutions
10. Cleanup — remove resources

## Key concepts
### ClusterSecretStore vs SecretStore
- ClusterSecretStore: cluster-wide, reusable across namespaces
- SecretStore: namespace-scoped, only for that namespace

This repo uses ClusterSecretStore for simplicity and reusability.

## IRSA (IAM Roles for Service Accounts)
IRSA allows Kubernetes ServiceAccounts to assume AWS IAM roles securely:

1. Pod uses a Kubernetes ServiceAccount
2. ServiceAccount is annotated with IAM role ARN
3. AWS OIDC provider trusts the ServiceAccount identity
4. Pod gets temporary AWS credentials
5. Pod can call AWS APIs

No static credentials needed.

### ExternalSecret refresh
By default, ExternalSecret syncs every 1 hour (refreshInterval: 1h).

If the AWS secret changes, the Kubernetes Secret is updated automatically.

## Consuming secrets in your app
### Option A: Environment variables
```bash
env:
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: frontend-k8s-secret
        key: API_KEY
```
Pros: Simple, most apps support it
Cons: Less secure, visible in some contexts, requires restart for updates

### Option B: Mounted files
```bash
volumeMounts:
  - name: secrets
    mountPath: /secrets
    readOnly: true
volumes:
  - name: secrets
    secret:
      secretName: frontend-k8s-secret
```
Pros: More secure, files can update without restart
Cons: App must read from files

##Security notes
- ✅ Secrets are encrypted in AWS Secrets Manager
- ✅ IRSA uses temporary credentials (no long-lived keys)
- ✅ IAM policy follows least privilege principle
- ✅ Kubernetes Secrets are encrypted at rest (if enabled)
- ⚠️ Do not print secrets in logs or terminal
- ⚠️ Do not commit real secrets to git
- ⚠️ Restrict who can read Kubernetes Secrets with RBAC

## Troubleshooting
Common issues and solutions are documented in docs/09-troubleshooting.md.

Quick checklist:
```bash
kubectl get nodes works
kubectl get pods -n external-secrets shows running pods
kubectl describe clustersecretstore aws-secrets-manager shows "Valid"
kubectl describe externalsecret frontend-secret -n default shows "SecretSynced"
kubectl get secret frontend-k8s-secret -n default exists
kubectl exec -it <pod> -- env | grep API_KEY shows the value
```

## Cleanup
To remove all resources:
```bash
kubectl delete deployment frontend -n default
kubectl delete externalsecret frontend-secret -n default
kubectl delete clustersecretstore aws-secrets-manager
helm uninstall external-secrets -n external-secrets
kubectl delete namespace external-secrets
```
See docs/10-cleanup.md for detailed cleanup steps.

## Contributing
This is a reference implementation. Feel free to adapt it to your needs.

## References
External Secrets Operator Documentation
AWS Secrets Manager
IRSA Documentation
EKS Best Practices
