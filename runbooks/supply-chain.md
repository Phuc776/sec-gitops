# Supply Chain Runbook

## Goal

Lab 2.2 proves two controls:

- CI blocks images with HIGH or CRITICAL CVEs before they are pushed.
- Admission blocks unsigned images before they can run in Kubernetes.

## Files

```text
.github/workflows/build-push.yml
argocd/apps/policy-controller.yaml
argocd/apps/policies.yaml
policies/cluster-image-policy.yaml
signing/cosign.pub
test/supply-chain-signed-pod.yaml
test/supply-chain-unsigned-pod.yaml
```

`test/` is local-only and ignored by Git.

## GitHub Secrets

Create these GitHub Actions secrets before relying on CI signing:

```text
COSIGN_PRIVATE_KEY
COSIGN_PASSWORD
```

Use the full local `cosign.key` content for `COSIGN_PRIVATE_KEY`. Do not commit the private key.

## Install Admission Controller

Push/sync these ArgoCD apps:

```powershell
kubectl apply -f argocd/apps/policy-controller.yaml
kubectl apply -f argocd/apps/policies.yaml
```

Check:

```powershell
kubectl get application policy-controller image-policies -n argocd
kubectl get pods -n cosign-system
kubectl get crd clusterimagepolicies.policy.sigstore.dev
kubectl get clusterimagepolicy
```

Expected:

```text
policy-controller   Synced   Healthy
image-policies      Synced   Healthy
ClusterImagePolicy  require-phuc776-signature
```

## Verify Image Signature

Verify the image used by the lab:

```powershell
cosign verify --key signing/cosign.pub ghcr.io/phuc776/w10-api:0.0.1
```

If the image is not signed yet, sign it locally once:

```powershell
cosign sign --yes --key cosign.key ghcr.io/phuc776/w10-api:0.0.1
```

## Enable Admission For Demo Namespace

Only label the namespace after the current app image has a valid signature:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite
```

## Test

Unsigned image must be rejected:

```powershell
kubectl apply -f test/supply-chain-unsigned-pod.yaml
```

Expected result:

```text
admission webhook ... denied the request
signature check failed
```

Signed image must be accepted:

```powershell
kubectl apply -f test/supply-chain-signed-pod.yaml
kubectl get pod signed-w10-api -n demo
```

Clean up:

```powershell
kubectl delete -f test/supply-chain-signed-pod.yaml --ignore-not-found
kubectl delete -f test/supply-chain-unsigned-pod.yaml --ignore-not-found
```

To disable admission temporarily:

```powershell
kubectl label namespace demo policy.sigstore.dev/include-
```
