# Lab 2.1 ESO Next Steps

This repo now contains the GitOps manifests for AWS Secrets Manager + External Secrets Operator, but they are not pushed or applied yet.

## Files That Should Be Committed Later

- `argocd/apps/eso.yaml`
- `argocd/apps/eso-config.yaml`
- `eso/secret-store.yaml`
- `eso/external-secret.yaml`
- `eso/secret-reader-pod.yaml`
- `runbooks/aws-secrets-manager-setup.md`
- `runbooks/eso-rotation.md`

## Local-Only File

Do not commit this file:

- `eso/aws-credentials.secret.yaml`

It is ignored by `.gitignore` through `*.secret.yaml`. Fill in the real values locally, then apply it manually:

```powershell
kubectl apply -f eso/aws-credentials.secret.yaml
```

Alternatively, create the same Secret directly from the AWS CLI profile:

```powershell
$accessKey = aws configure get aws_access_key_id
$secretKey = aws configure get aws_secret_access_key

kubectl create secret generic aws-credentials -n demo `
  --from-literal=access-key-id=$accessKey `
  --from-literal=secret-access-key=$secretKey
```

## Region Check

`eso/secret-store.yaml` currently points to:

```yaml
region: us-east-1
```

Before pushing, make sure the AWS secret exists in the same region, or change the region in `eso/secret-store.yaml`.

## AWS Secret Required

Create or update this AWS Secrets Manager secret in the configured region:

```text
w10/demo/db-password
```

The secret string must be JSON with a `DB_PASSWORD` field:

```json
{"DB_PASSWORD":"demo-password-v1"}
```

Create:

```powershell
aws secretsmanager create-secret `
  --region us-east-1 `
  --name w10/demo/db-password `
  --secret-string '{\"DB_PASSWORD\":\"demo-password-v1\"}'
```

Update if it already exists:

```powershell
aws secretsmanager put-secret-value `
  --region us-east-1 `
  --secret-id w10/demo/db-password `
  --secret-string '{\"DB_PASSWORD\":\"demo-password-v1\"}'
```

## Push And Sync Order

When ready:

1. Push the manifests.
2. Let Argo CD sync `external-secrets`.
3. Manually create/apply `aws-credentials` in namespace `demo`.
4. Let Argo CD sync `eso-config`.
5. Verify ESO created `api-db-secret`.

## Verification

```powershell
kubectl get application external-secrets eso-config -n argocd
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n demo
kubectl get secret api-db-secret -n demo
```

Decode the synced password:

```powershell
[System.Text.Encoding]::UTF8.GetString(
  [System.Convert]::FromBase64String(
    (kubectl get secret api-db-secret -n demo -o jsonpath="{.data.DB_PASSWORD}")
  )
)
```

Rotate:

```powershell
aws secretsmanager put-secret-value `
  --region us-east-1 `
  --secret-id w10/demo/db-password `
  --secret-string '{\"DB_PASSWORD\":\"demo-password-v2\"}'
```

Expected result: `api-db-secret` updates within `refreshInterval: 30s`, and `secret-reader` pod age does not reset.
