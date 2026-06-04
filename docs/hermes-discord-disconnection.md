# Hermes Discord Connection and Disconnection Research

## 1. Executive Summary
The Hermes gateway continues to receive and send updates via Discord despite the Discord configuration being commented out in the host's `/home/fcurrie/.hermes/.env` file. This occurs because the gateway runs in the K3s cluster (`hermes-system` namespace) and reads its active configuration from the Persistent Volume (PV) mounted at `/workspace/.hermes`. 

During the HA migration, the local `.env` configuration containing active Discord credentials was copied into the PV. Commenting out credentials on the host does not modify the active PV.

## 2. Technical Findings
- **Active Configuration Path:** `/workspace/.hermes/.env` inside the `hermes-gateway` container (backed by `hermes-home-pvc` on Longhorn).
- **Active Discord Settings Found:**
  - `DISCORD_BOT_TOKEN`
  - `DISCORD_HOME_CHANNEL`
  - `DISCORD_ALLOW_ALL_USERS`
  - `DISCORD_IGNORE_NO_MENTION`
  - `DISCORD_REQUIRE_MENTION`
  - `DISCORD_FREE_RESPONSE_CHANNELS`
- **Current State:** These keys are commented out on the host (`/home/fcurrie/.hermes/.env`) but remain uncommented and active on the K3s Persistent Volume.

## 3. Complete Disconnection Procedure

To completely disconnect the active Hermes deployment from Discord, run the following steps on the controller/host node:

### Step 1: Comment Out Discord Variables on the PV
Execute a search-and-replace via `kubectl exec` to comment out any active `DISCORD_` configurations inside the Persistent Volume's `.env`:
```bash
kubectl exec -n hermes-system deployment/hermes-gateway -c hermes -- sed -i 's/^DISCORD_/# DISCORD_/' /workspace/.hermes/.env
```

### Step 2: Restart the Hermes Gateway Deployment
Trigger a rolling restart to force the gateway instances to re-read the updated `.env` file and terminate active Discord listeners:
```bash
kubectl rollout restart deployment/hermes-gateway -n hermes-system
```

### Step 3: Verify Disconnection
Verify that the gateway starts up without establishing a Discord connection:
```bash
kubectl logs -n hermes-system deployment/hermes-gateway -c hermes --tail=50
```
Expect to see logs confirming that the platform initialization skipped Discord and only connected to remaining active platforms.
