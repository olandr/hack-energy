# KeplerNodeJoulesStale

## Summary

`kepler_node_platform_joules_total` stopped increasing on a node, but it was
increasing an hour ago. Kepler is scraping fine. The power backend behind the
counter has gone quiet.

## Impact

Node-level energy numbers freeze. Container and process attribution on that
node is still computed, but it is divided across a stale total, so values
drift.

## Checks

1. Look at the Kepler pod logs for errors from the backend:

   ```sh
   kubectl -n <namespace> logs <pod> | grep -iE "rapl|hwmon|redfish|power"
   ```

2. Check the backend on the host:

   - RAPL: `ls /sys/class/powercap/intel-rapl*`
   - hwmon: `ls /sys/class/hwmon`
   - Redfish: check `config.experimental.platform.redfish.enabled` and the
     BMC endpoint.

3. If GPU is enabled, check the NVML init container and that
   `/run/nvidia/driver` exists on the node.

## Common causes

- A kernel update removed read permissions on `/sys/class/powercap`. Kepler
  needs `CAP_DAC_READ_SEARCH` or privileged mode.
- The hwmon chip the first iteration picked no longer reports. Tune
  `config.cpu.preferredMeters` or `config.experimental.hwmon.chipRules`.
- The Redfish BMC is unreachable. Check network path and credentials.

## Recovery

- Restart the pod so Kepler re-initialises the backend list.
- If the hardware really stopped reporting, switch `preferredMeters` to the
  next backend in the list.
