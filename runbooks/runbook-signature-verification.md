# Runbook: Image Signature Verification via Sigstore Policy Controller

This runbook explains how the Sigstore Policy Controller enforces container image signature verification on the Kubernetes cluster, blocking unsigned images and allowing only verified images.

## 1. Supply Chain Verification Architecture
- **Signing Tool:** Cosign (Sigstore).
- **Verify Controller:** Sigstore `policy-controller` installed in the `cosign-system` namespace via Helm.
- **Enforcement Scope:** Policies are active on namespaces labeled with `policy.sigstore.dev/include: "true"`.
- **Policy Resource:** `ClusterImagePolicy` named `image-signature-policy` matching glob `ghcr.io/bearh141/*`.
- **Authority:** Configured with our public key (`cosign.pub`) to authenticate signatures.

---

## 2. Key Management & Manual Signing

### Generate Cosign Key Pair
To generate a new Cosign cryptographic key pair:
```bash
cosign generate-key-pair
```
This generates:
- `cosign.pub`: Public key (committed to the repo under `signing/cosign.pub` and added to `ClusterImagePolicy`).
- `cosign.key`: Encrypted private key (stored in GitHub Secret `COSIGN_PRIVATE_KEY`).

### Manually Sign an Image
If you need to sign a container image manually:
```bash
export COSIGN_PASSWORD="<your_private_key_password>"
cosign sign --key signing/cosign.key ghcr.io/bearh141/w10-api-141:0.0.2
```
This uploads the signature as an OCI artifact to the registry alongside the image tag.

### Manually Verify an Image
To check if an image signature is valid:
```bash
cosign verify --key signing/cosign.pub ghcr.io/bearh141/w10-api-141:0.0.2
```

---

## 3. Verification & Validation in Cluster

### Test 1: Deploying Unsigned Image (Expected: REJECT)
Attempt to deploy an unsigned image (e.g. standard `nginx:latest` or an unsigned build) in the labeled `payments` namespace:
```bash
kubectl run test-unsigned --image=nginx:latest -n payments
```
Expected output:
```
Error from server (Forbidden): admission webhook "policy.sigstore.dev" denied the request: image nginx:latest is not signed/verified.
```

### Test 2: Deploying Signed Image (Expected: PASS)
Deploy the API pod using the signed image built from the GitHub Actions pipeline:
```bash
kubectl run test-signed --image=ghcr.io/bearh141/w10-api-141:0.0.2 -n payments
```
Expected output:
```
pod/test-signed created
```
The pod is successfully admitted because its signature matches the public key in `ClusterImagePolicy`.
