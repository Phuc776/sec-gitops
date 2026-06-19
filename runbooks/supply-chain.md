# Supply Chain Runbook

## Goal

Lab 2.2 proves three controls:

- Trivy blocks HIGH or CRITICAL CVEs in CI.
- Cosign signs the image after CI builds and pushes it.
- Admission rejects unsigned images in Kubernetes.

## Risk Map

```text
Trivy in CI       -> F-05
Cosign signing    -> F-06
Admission verify  -> F-06
```

## Key Rules

`cosign.key` is private. It is only used by GitHub Actions through repository secrets:

```text
COSIGN_PRIVATE_KEY
COSIGN_PASSWORD
```

`signing/cosign.pub` is public. It is committed and embedded in:

```text
policies/cluster-image-policy.yaml
```

Do not sign the lab image locally for the main proof. The image must be signed by CI after Trivy passes.

## CI Flow

```text
push src/api or workflow change
-> build image for scan
-> Trivy scan HIGH/CRITICAL with exit-code 1
-> build and push image to GHCR
-> sign pushed image digest with COSIGN_PRIVATE_KEY
-> update app-api/rollout.yaml to the signed digest
```

The rollout must use the signed digest, not just `latest`.

## Cluster Setup

ArgoCD apps:

```text
argocd/apps/policy-controller.yaml
argocd/apps/policies.yaml
```

Checks:

```powershell
kubectl get application policy-controller image-policies -n argocd
kubectl get pods -n cosign-system
kubectl get clusterimagepolicy
```

Expected:

```text
policy-controller webhook Running
image-policies Healthy
ClusterImagePolicy require-phuc776-signature exists
```

## Test

Use the local ignored `test/` folder. It creates a dedicated namespace with admission enabled, so `demo` is not disrupted before the CI-signed rollout is ready.

```powershell
.\test\supply-chain-test-ci.ps1
```

Expected:

```text
cosign verify succeeds for the rollout image digest
unsigned image is rejected by admission
signed rollout image is accepted
```

Manual commands:

```powershell
kubectl apply -f test/supply-chain-namespace.yaml
cosign verify --key signing/cosign.pub <image-from-app-api-rollout.yaml>
kubectl apply -f test/supply-chain-unsigned-pod.yaml
kubectl apply -f test/supply-chain-signed-pod.yaml
```

Clean up:

```powershell
kubectl delete -f test/supply-chain-signed-pod.yaml --ignore-not-found
kubectl delete -f test/supply-chain-unsigned-pod.yaml --ignore-not-found
kubectl delete -f test/supply-chain-namespace.yaml --ignore-not-found
```

## Evidence

The lab is complete when:

```text
Push image with HIGH CVE -> CI fails
Deploy unsigned image    -> admission rejects
Deploy CI-signed image   -> admission allows
```

If a vendor CVE has no fix yet, document a time-bounded exception in `runbooks/cve-exception-adr.md`.
