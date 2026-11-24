# KeplerHighCPU

## Summary

A Kepler pod uses more than half a CPU on average over 15 minutes. Kepler is
designed to be cheap, so this is worth a look.

## Impact

Kepler eats CPU that workloads could use. On large clusters this adds up.

## Checks

1. Compare pods. If one pod is high and the rest are normal, the node under
   it probably has far more containers, processes, or cgroups than the
   others.
2. Check the scrape interval on Prometheus and in
   `config.monitor.interval`. Very short intervals cause a power computation
   on every scrape.
3. Check `config.exporter.prometheus.metricsLevel`. Dropping `process` cuts
   the biggest cost.

## Common causes

- Scrape interval is shorter than `config.monitor.staleness`, so every
  scrape recomputes the power table instead of serving the cache.
- The node runs thousands of short-lived processes. The process-level
  exporter has to keep up.
- Debug collectors are on. Check `config.exporter.prometheus.debugCollectors`.

## Recovery

- Raise `config.monitor.interval` and `config.monitor.staleness` so the
  cache is used between scrapes.
- Drop the `process` level from `metricsLevel` if you do not need
  per-process attribution.
- Raise `daemonset.resources.limits.cpu` if the extra cost is expected.
