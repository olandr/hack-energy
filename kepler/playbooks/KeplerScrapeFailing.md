# KeplerScrapeFailing

## Summary

The Kepler pod is up, but Prometheus scraped zero samples from it for the
last 15 minutes.

## Impact

Energy metrics from the affected pod stop updating even though the pod is
running. Alerts that depend on recent samples will not fire for that pod.

## Checks

1. Port-forward the pod and hit `/metrics`:

   ```sh
   kubectl -n <namespace> port-forward <pod> 9102
   curl -s localhost:9102/metrics | head
   ```

   If this returns no `kepler_*` metrics, the exporter inside the pod is the
   problem. If it returns metrics, the problem is between Prometheus and the
   pod.

2. Check that the Service selector matches the pod labels:

   ```sh
   kubectl -n <namespace> describe svc <service>
   ```

3. Check the ServiceMonitor or PodMonitor is picked up by Prometheus. In the
   Prometheus UI, open **Status → Targets** and look for the Kepler job.

## Common causes

- The Prometheus `serviceMonitorSelector` does not match the labels on the
  Kepler ServiceMonitor. Set `prometheus.additionalLabels` so the selector
  can pick it up.
- The `/metrics` endpoint is disabled. Check
  `config.exporter.prometheus.enabled: true`.
- A NetworkPolicy blocks Prometheus. Check `networkPolicy.ingress` allows
  the Prometheus pod.

## Recovery

- Fix the label or NetworkPolicy and wait one scrape interval.
- If the exporter itself is silent, restart the pod and check the logs for
  init errors.
