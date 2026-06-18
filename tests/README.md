# Gatekeeper Policy Test Guide

This directory contains test Kubernetes manifests to verify that the OPA Gatekeeper policies are functioning correctly on the cluster.

All tests are configured to run in the `demo` namespace.

---

## 1. Test: Disallow Latest Tag (Expected: REJECT)
This pod uses `nginx:latest`, which violates the `disallow-latest-tag` policy.
```bash
kubectl apply -f tests/pod-latest.yaml
```

## 2. Test: Require Resource Limits (Expected: REJECT)
This pod does not define `resources.limits`, which violates the `require-resource-limits` policy.
```bash
kubectl apply -f tests/pod-no-limits.yaml
```

## 3. Test: Disallow Root User (Expected: REJECT)
This pod attempts to run as root (`runAsUser: 0`), which violates the `disallow-root-user` policy.
```bash
kubectl apply -f tests/pod-root-user.yaml
```

## 4. Test: Disallow Host Network (Expected: REJECT)
This pod sets `hostNetwork: true`, which violates the `disallow-host-network` policy.
```bash
kubectl apply -f tests/pod-hostnetwork.yaml
```

## 5. Test: Valid Pod (Expected: PASS)
This pod is configured correctly (pinned image version, has cpu/memory limits, runs as a non-root user, and does not use hostNetwork). It should deploy successfully.
```bash
kubectl apply -f tests/pod-valid.yaml
```

## 6. Test: Require Owner Label - Violating (Expected: REJECT)
This Deployment does not have `metadata.labels.owner` defined, violating the `require-owner-label` custom policy.
```bash
kubectl apply -f tests/deploy-no-owner.yaml
```

## 7. Test: Require Owner Label - Valid (Expected: PASS)
This Deployment has the `owner: platform-team` label defined, passing the custom policy.
```bash
kubectl apply -f tests/deploy-with-owner.yaml
```

---

## Cleanup
To delete the valid test resources after testing:
```bash
kubectl delete -f tests/pod-valid.yaml
kubectl delete -f tests/deploy-with-owner.yaml
```

