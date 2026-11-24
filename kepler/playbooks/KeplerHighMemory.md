# KeplerHighMemory

## Summary

A Kepler pod working set is above 500Mi for 30 minutes.

## Impact

Kepler is competing with workloads for memory. If it hits the cgroup limit,
the kubelet kills it and the node loses energy metrics for a while.

## Checks

1. Check the Go runtime metrics that Kepler exposes:

   ```sh
   kubectl -n <namespace> port-forward <pod> 9102
   curl -s localhost:9102/metrics | grep go_memstats_
   ```

   A steady rise in `go_memstats_alloc_bytes` points to a leak. A high but
   flat value usually points to too many tracked workloads.

2. Check `config.monitor.maxTerminated`. Kepler keeps terminated workloads
   in memory until the next scrape. A very high value on a busy node grows
   the heap.

3. Check the pod restart count and `kubectl -n <namespace> describe pod
   <pod>` for `OOMKilled`.

## Common causes

- `maxTerminated` is set high on a node with a lot of churn (Jobs,
  short-lived pods).
- The scrape interval is long, so terminated workloads pile up before they
  flush.
- pprof is enabled and holding profiles.

## Recovery

- Lower `config.monitor.maxTerminated`, or set it to 0 to turn the feature
  off.
- Shorten the scrape interval so flushes happen more often.
- Raise `daemonset.resources.limits.memory`. If the pod was OOMKilled,
  raise the limit first so you can collect a heap dump through pprof.
