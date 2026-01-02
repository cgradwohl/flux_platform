## CSI + AWS Secret Store Provider Auth Chain

- Pod -> ServiceAccount -> IAM Role (via IRSA) -> AWS STS -> Secrets Manager

When a pod starts:

1. Pod uses a ServiceAccount
2. ServiceAccount is annotated with IAM role
3. EKS injects a projected token
4. AWS provider uses STS AssumeRoleWithWebIdentity
5. Secrets are fetched on-demand
6. Secrets are mounted into the pod filesystem
