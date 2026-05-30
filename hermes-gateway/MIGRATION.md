# Hermes Gateway HA Migration Guide

Follow these steps to migrate your Hermes gateway from the local systemd-controlled instance to the highly available K3s-managed deployment.

## Prerequisites
1.  Ensure you have merged the `feature/hermes-ha` pull request into `main`.
2.  ArgoCD must be synced for the `hermes-system` namespace to be created.

## Steps

### 1. Stop the Local Service
Run these commands on the node where Hermes is currently running (usually `ubuntu-dev`):

```bash
systemctl --user stop hermes-gateway
systemctl --user disable hermes-gateway
```

### 2. Prepare the PVC
The K3s deployment uses a Longhorn `ReadWriteMany` (RWX) volume. We need to copy your local data into it:

1.  Identify the Pod name created by ArgoCD:
    `kubectl get pods -n hermes-system`
2.  Copy your data from the local host to the Persistent Volume:
    `kubectl cp /home/fcurrie/.hermes/ <pod-name>:/workspace/.hermes/ -n hermes-system`

*(Note: If you need to perform this copy in chunks, you can rsync them manually.)*

### 3. Verify
Once the data is synchronized, the pod should automatically stabilize and connect to your messaging platforms. You can view the logs with:
`kubectl logs -f -n hermes-system -l app=hermes-gateway`
