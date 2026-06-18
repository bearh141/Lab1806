# Runbook: Secrets Rotation via External Secrets Operator (ESO)

This runbook describes the process for rotating database credentials in AWS Secrets Manager and verifying that the changes propagate to the Kubernetes cluster with Zero Downtime (no pod restarts).

## 1. Secrets Architecture
- **Source of Truth:** AWS Secrets Manager secret `w10/demo/db-password` (in region `ap-southeast-1`).
- **Sync Operator:** External Secrets Operator (ESO) running in `external-secrets` namespace.
- **Connection Resource:** `SecretStore` named `aws-store` authenticating using `aws-creds` secret (containing AWS Access Key ID and Secret Access Key).
- **Mapping Resource:** `ExternalSecret` named `db-creds` mapping `w10/demo/db-password` to Kubernetes Secret `db-secret` in the `demo` namespace.
- **Poll Interval:** `refreshInterval` is configured to `10s` for quick rotation demonstration.
- **Workload Consumption:** Mounted as a read-only volume under `/etc/db-secret/` in `app-api` rollout pods.

---

## 2. Rotation Procedure

### Step 1: Update secret value in AWS Secrets Manager
Use the AWS Console or the AWS CLI to update the secret value:
```bash
aws secretsmanager put-secret-value \
  --secret-id w10/demo/db-password \
  --secret-string "NewRotatedPassword123!" \
  --region ap-southeast-1
```

### Step 2: Monitor ExternalSecret Sync
Within 10 seconds, the status of the `ExternalSecret` should reconcile:
```bash
kubectl get externalsecret db-creds -n demo
```
Expected output shows `STATUS: SecretSynced` and `READY: True`.

---

## 3. Verification & Validation

### Check K8s Secret Updates
Verify that the Kubernetes secret `db-secret` has updated to the base64-encoded representation of the new password:
```bash
# In Linux/macOS:
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 --decode; echo

# In Windows Powershell:
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

### Confirm Zero-Downtime (No Pod Restarts)
Confirm that the API pods have **not** restarted since the secret update. The `AGE` and `RESTARTS` columns must remain unchanged:
```bash
kubectl get pods -n demo -l app=api
```

### Validate Pod Mount Sync
Exec into one of the running API pods to verify that the container sees the new password inside the mounted volume:
```bash
kubectl exec -n demo <pod-name> -- cat /etc/db-secret/password
```
Expected output: `NewRotatedPassword123!`
