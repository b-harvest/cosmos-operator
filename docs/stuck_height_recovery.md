## StuckHeightRecovery

Status: v1

StuckHeightRecovery automatically recovers Cosmos blockchain nodes when they get stuck at a specific height.

The controller monitors blockchain height and detects when a node is stuck for a configurable duration. When stuck height is detected,
it can optionally create a VolumeSnapshot for safety, then executes a user-provided recovery script in a temporary pod. The script has
access to the node's PVC and can perform operations like deleting lock files, resetting the addrbook, or other recovery tasks.

**Warning: Recovery scripts run with full access to the node's data directory.** Test your recovery scripts thoroughly before production use.

### Key features

- Automatic blockchain height monitoring
- Configurable stuck detection time (e.g., "10m", "1h")
- Optional VolumeSnapshot creation before recovery
- Custom recovery script execution
- Rate limiting to prevent excessive recovery attempts (default: 3 attempts per 5 minutes)
- Flexible pod template for custom images and resources

### Basic example

```yaml
apiVersion: cosmos.bharvest.io/v1
kind: StuckHeightRecovery
metadata:
  name: cosmoshub-recovery
  namespace: default
spec:
  fullNodeRef:
    name: cosmoshub
  stuckDuration: "10m"
  recoveryScript: |
    #!/bin/sh
    # Delete addrbook to reconnect peers
    rm -f $CHAIN_HOME/config/addrbook.json
```

### Recovery with snapshot

```yaml
apiVersion: cosmos.bharvest.io/v1
kind: StuckHeightRecovery
metadata:
  name: cosmoshub-recovery-with-snapshot
  namespace: default
spec:
  fullNodeRef:
    name: cosmoshub
  stuckDuration: "15m"
  createVolumeSnapshot: true
  volumeSnapshotClassName: "csi-snapshot-class"
  recoveryScript: |
    #!/bin/sh
    set -e
    find $CHAIN_HOME/data -name "LOCK" -delete
    rm -rf $CHAIN_HOME/data/cs.wal
    rm -f $CHAIN_HOME/config/addrbook.json
```

### Custom pod template

```yaml
apiVersion: cosmos.bharvest.io/v1
kind: StuckHeightRecovery
metadata:
  name: cosmoshub-recovery-custom
  namespace: default
spec:
  fullNodeRef:
    name: cosmoshub
  stuckDuration: "10m"
  podTemplate:
    image: "alpine:3.18"
    resources:
      requests:
        cpu: "200m"
        memory: "256Mi"
  recoveryScript: |
    #!/bin/sh
    rm -f $CHAIN_HOME/config/addrbook.json
```

### Rate limiting

```yaml
apiVersion: cosmos.bharvest.io/v1
kind: StuckHeightRecovery
metadata:
  name: cosmoshub-recovery
  namespace: default
spec:
  fullNodeRef:
    name: cosmoshub
  stuckDuration: "10m"
  maxRetries: 5
  rateLimitWindow: "10m"
  recoveryScript: |
    #!/bin/sh
    rm -f $CHAIN_HOME/config/addrbook.json
```

### Environment variables

The recovery script has access to these environment variables:

- `POD_NAME`: Name of the stuck pod
- `POD_NAMESPACE`: Namespace of the stuck pod
- `CURRENT_HEIGHT`: Current height of the stuck pod
- `PVC_NAME`: Name of the PVC
- `CHAIN_HOME`: Home directory of the chain (/chain-home)

### Status checking

```bash
# Check phase
kubectl get shr cosmoshub-recovery -o jsonpath='{.status.phase}'

# Check current height
kubectl get shr cosmoshub-recovery -o jsonpath='{.status.lastObservedHeight}'

# Check recovery attempts
kubectl get shr cosmoshub-recovery -o jsonpath='{.status.recoveryAttempts}'
```

### Recovery pod logs

```bash
# Find recovery pod
kubectl get pods -l cosmos.bharvest.io/type=recovery-pod

# Check logs
kubectl logs <recovery-pod-name>
```

Limitations:
- The CosmosFullNode and StuckHeightRecovery must be in the same namespace.
- Recovery scripts are executed with the permissions of the recovery pod.
- VolumeSnapshot support requires CSI driver with snapshot capabilities.

[Example yaml](../config/samples/cosmos_v1_stuckheightrecovery.yaml)
