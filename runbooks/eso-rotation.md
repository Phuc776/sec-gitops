# ESO Rotation Runbook

## Goal

Prove a secret can rotate in less than 60 seconds without restarting the pod.

## AWS Secrets Manager Lab Flow

1. Sync `argocd/apps/eso.yaml`.
2. Create the AWS secret and the in-cluster `aws-credentials` Secret using `runbooks/aws-secrets-manager-setup.md`.
3. Sync `argocd/apps/eso-config.yaml`.
4. Confirm the generated Secret:

```powershell
kubectl get secret api-db-secret -n demo -o jsonpath="{.data.DB_PASSWORD}"
```

Decode it:

```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret api-db-secret -n demo -o jsonpath="{.data.DB_PASSWORD}")))
```

5. Confirm the reader pod age:

```powershell
kubectl get pod secret-reader -n demo
```

6. Rotate the AWS source:

```powershell
aws secretsmanager put-secret-value `
  --region us-east-1 `
  --secret-id w10/demo/db-password `
  --secret-string '{\"DB_PASSWORD\":\"demo-password-v2\"}'
```

7. Verify the Kubernetes Secret changes within 30 seconds and the pod age does not reset.

## Optional Fake Provider Fallback

If AWS is unavailable, use `eso/fake-secret-store.example` as a fallback. This proves ESO reconciliation, `ExternalSecret` mapping, Kubernetes Secret update, and pod no-restart behavior, but it does not prove AWS IAM or AWS Secrets Manager integration.
