# AWS Secrets Manager Setup

## Goal

Prepare AWS Secrets Manager and a Kubernetes credential Secret for ESO without committing real credentials to Git.

## Assumptions

- AWS CLI or AWS SAM credentials already work on this machine.
- The lab region must match `eso/secret-store.yaml`.
- Current configured region: `us-east-1`.
- Kubernetes context points to the `w10` minikube cluster.

## Create The AWS Secret

Create the initial secret value:

```powershell
aws secretsmanager create-secret `
  --region us-east-1 `
  --name w10/demo/db-password `
  --secret-string '{\"DB_PASSWORD\":\"demo-password-v1\"}'
```

If the secret already exists, update it instead:

```powershell
aws secretsmanager put-secret-value `
  --region us-east-1 `
  --secret-id w10/demo/db-password `
  --secret-string '{\"DB_PASSWORD\":\"demo-password-v1\"}'
```

## Create Kubernetes AWS Credentials Secret

Do not commit AWS credentials. Create this Secret manually from the local AWS profile:

```powershell
$accessKey = aws configure get aws_access_key_id
$secretKey = aws configure get aws_secret_access_key

kubectl create secret generic aws-credentials -n demo `
  --from-literal=access-key-id=$accessKey `
  --from-literal=secret-access-key=$secretKey
```

If you use a non-default profile:

```powershell
$profile = "YOUR_PROFILE"
$accessKey = aws configure get aws_access_key_id --profile $profile
$secretKey = aws configure get aws_secret_access_key --profile $profile

kubectl create secret generic aws-credentials -n demo `
  --from-literal=access-key-id=$accessKey `
  --from-literal=secret-access-key=$secretKey
```

## Rotate Secret

```powershell
aws secretsmanager put-secret-value `
  --region us-east-1 `
  --secret-id w10/demo/db-password `
  --secret-string '{\"DB_PASSWORD\":\"demo-password-v2\"}'
```

ESO should update `api-db-secret` within `refreshInterval`.
