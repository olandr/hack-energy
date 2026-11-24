# KeplerDown

## Summary

Prometheus cannot reach a Kepler pod. Node-level energy metrics from that pod
are missing until it comes back.

## Impact

Dashboards and alerts that use Kepler data lose the affected node for the
duration of the outage.

## Checks

1. Find the pod:

   ```sh
   kubectl -n <namespace> get pods -l app.kubernetes.io/name=kepler -o wide
   ```

2. Check pod state and recent restarts. Look for `CrashLoopBackOff`,
   `ImagePullBackOff`, or `Error`.
3. Check the logs of the failing pod:

   ```sh
   kubectl -n <namespace> logs <pod> --previous
   ```

4. Check the node. The pod runs as a DaemonSet, so a NotReady node drops the
   pod with it.

## Common causes

- The node is NotReady or tainted so the DaemonSet pod cannot run.
- The container cannot read `/proc` or `/sys` from the host. Confirm
  `hostPID: true` and the `securityContext` match the values in `values.yaml`.
- The RAPL or hwmon backend is not available on this hardware. Set
  `config.cpu.preferredMeters` to a backend that works, for example `fake`
  for test clusters.

## Recovery

- Restart the pod:

  ```sh
  kubectl -n <namespace> delete pod <pod>
  ```

- If the node is the problem, drain or repair the node and the DaemonSet pod
  comes back on its own.
