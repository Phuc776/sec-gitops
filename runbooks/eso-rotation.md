# Lab 2.1 Runbook: External Secrets Operator

## Goal

Prove that application secrets are not stored in Git. The source of truth is AWS Secrets Manager, and External Secrets Operator syncs the value into Kubernetes.

```text
AWS Secrets Manager
  w10/demo/db-password
  field: DB_PASSWORD
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret demo/api-db-secret
        |
        v
Pod demo/secret-reader reads the mounted Secret
```

## Files

```text
argocd/apps/eso.yaml
argocd/apps/eso-config.yaml
eso/secret-store.yaml
eso/external-secret.yaml
eso/secret-reader-pod.yaml
runbooks/eso-rotation.md
```

Do not commit AWS credentials. The local credential file is ignored:

```text
eso/aws-credentials.secret.yaml
```

## Constants

```powershell
$Region = "us-east-1"
$SecretId = "w10/demo/db-password"
$Namespace = "demo"
$K8sSecret = "api-db-secret"
```

These must match:

- `eso/secret-store.yaml`: AWS region
- `eso/external-secret.yaml`: remote secret key and `DB_PASSWORD` property

## Prepare AWS Secret

Check AWS identity:

```powershell
aws sts get-caller-identity
```

Create or update the AWS secret without printing the value:

```powershell
'{"DB_PASSWORD":"demo-password-v1"}' | Set-Content -NoNewline .secret-value.json

aws secretsmanager describe-secret `
  --region $Region `
  --secret-id $SecretId

if ($LASTEXITCODE -eq 0) {
  aws secretsmanager put-secret-value `
    --region $Region `
    --secret-id $SecretId `
    --secret-string file://.secret-value.json
} else {
  aws secretsmanager create-secret `
    --region $Region `
    --name $SecretId `
    --secret-string file://.secret-value.json
}

Remove-Item .secret-value.json
```

Verify the JSON shape:

```powershell
$encoded = aws secretsmanager get-secret-value `
  --region $Region `
  --secret-id $SecretId `
  --query SecretString `
  --output json

$secretString = $encoded | ConvertFrom-Json
$secretObject = $secretString | ConvertFrom-Json

if ($secretObject.PSObject.Properties.Name -contains "DB_PASSWORD") {
  "DB_PASSWORD field: present"
} else {
  "DB_PASSWORD field: missing"
}
```

## Prepare Kubernetes Credential

Create the Kubernetes credential Secret before or immediately after `eso-config` syncs:

```powershell
$accessKey = aws configure get aws_access_key_id
$secretKey = aws configure get aws_secret_access_key

kubectl create secret generic aws-credentials -n $Namespace `
  --from-literal=access-key-id=$accessKey `
  --from-literal=secret-access-key=$secretKey `
  --dry-run=client -o yaml | kubectl apply -f -
```

Expected:

```powershell
kubectl get secret aws-credentials -n $Namespace
```

## Check ESO Sync

```powershell
kubectl get application external-secrets eso-config -n argocd
kubectl get pods -n external-secrets
kubectl get secretstore,externalsecret -n $Namespace
kubectl get secret $K8sSecret -n $Namespace
kubectl get pod secret-reader -n $Namespace
```

Expected:

```text
external-secrets                  Synced/Healthy
SecretStore/aws-store             Valid, Ready=True
ExternalSecret/api-db-password    SecretSynced, Ready=True
Secret demo/api-db-secret         exists
Pod demo/secret-reader            Running
```

Optional decode check:

```powershell
[System.Text.Encoding]::UTF8.GetString(
  [System.Convert]::FromBase64String(
    (kubectl get secret $K8sSecret -n $Namespace -o jsonpath="{.data.DB_PASSWORD}")
  )
)
```

## Rotation Test

Record pod age:

```powershell
kubectl get pod secret-reader -n $Namespace
```

Rotate the AWS secret:

```powershell
'{"DB_PASSWORD":"demo-password-v2"}' | Set-Content -NoNewline .secret-value.json

aws secretsmanager put-secret-value `
  --region $Region `
  --secret-id $SecretId `
  --secret-string file://.secret-value.json

Remove-Item .secret-value.json
```

Wait 30-60 seconds. `eso/external-secret.yaml` uses:

```yaml
refreshInterval: 30s
```

Check that the workload sees the new value:

```powershell
kubectl logs secret-reader -n $Namespace --tail=20
```

Expected: logs show `demo-password-v2`.

Check the pod age again:

```powershell
kubectl get pod secret-reader -n $Namespace
```

## Pass Criteria

```text
AWS Secrets Manager value changes.
ESO syncs the new value into demo/api-db-secret.
secret-reader sees the new value.
secret-reader pod is not restarted for the rotation.
No AWS credential file is committed to Git.
```
