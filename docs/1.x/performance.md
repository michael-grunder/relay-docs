---
title: Performance
---

# Performance

Relay's writer limit is the main control for balancing cache throughput against system utilization. Tune it for your application's workload and available CPU capacity.

**Version note:** The shared-cache model below applies to the development version after v0.50.0.

Each endpoint owns one cache in shared memory, with a separate map and writer lock for each Redis database. All PHP workers in the same process pool can read that cache concurrently through Relay's lock-free read path. A configurable number of tracking connections can populate the cache and manage their own invalidations.

## `relay.max_db_writers`

This directive limits the number of tracking connections allowed to populate each endpoint's shared cache. The default is `4`, and the supported range is `1–1024`. The limit counts connections: multiple connections in one PHP worker can each occupy a writer slot.

Connections without a writer slot can read cached data and fall back to Redis when needed, but cannot populate the cache. This limit does not restrict how many connections can read the cache or issue writes to Redis.

Increasing the writer limit can improve throughput, even at very high writer counts, but the gains can come at a substantial cost in CPU usage and overall system utilization. More writers share the same cache locks and each manages its own invalidations. The setting that achieves the highest throughput may leave too little capacity for the rest of your application.

Start with the default and increase it gradually under a representative workload. Measure throughput, application latency, and CPU utilization together, including periods when the cache is being populated or frequently invalidated. Choose a limit that meets your throughput needs while preserving capacity for PHP and other services. The number of CPU cores is useful context, but is not a hard ceiling on the writer limit.

```ini
relay.max_db_writers = 4
relay.key_leases = 32
```

These settings are loaded when PHP starts. Restart PHP workers after changing them, and ensure the lease pool can accommodate the writers you intend to use.

## `relay.key_leases`

Each admitted tracking connection needs its own lease. `relay.key_leases` controls the shared pool of leases across all endpoints, including separate Redis servers and individual cluster nodes. Its default is `32`.

Relay allocates at least as many slots as `relay.max_db_writers`, rounds the allocation up to a power of two, and uses a minimum of `16` and a maximum of `32768` slots. This automatic sizing covers the writer limit for one endpoint; deployments using several endpoints may need a larger pool. For example, four endpoints with eight writers each need at least 32 leases in total.

An available lease does not bypass the per-endpoint writer limit. Conversely, a connection needs an available lease to become a writer even if its endpoint has room for more writers.

## `relay.locks.*`

The default locking mechanism used for the in-memory cache and allocator is `adaptive-mutex` with a fallback to `mutex` if glibc is not available on the system. 

- `spinlock`: The lowest latency lock which will busy-wait until the lock is available. This is likely the right choice on machines with only a few cores.
- `mutex`: When contention is detected, this lock will sleep until it is available. It has higher latency than a spinlock but uses far less CPU. On machines with many cores it is likely the right choice.
- `adaptive-mutex`: This lock is a hybrid of the two above. When contention is detected it will first spin waiting for the lock to free and then sleep if the lock is still not available. Each time it spins it will update its strategy depending on how long it took. Requires glibc and will fall back to `mutex` if glibc is not available.

## `relay.flush_batch_size`

Since v0.50.0, Relay reclaims flushed cache memory incrementally. Flushing removes the affected data from the active cache, so new reads cannot use those entries. Once existing readers can no longer reference the retired data, Relay frees its entries in batches. This spreads cleanup work across callbacks instead of processing a large cache all at once.

The `relay.flush_batch_size` directive sets the maximum number of cached entries reclaimed per cleanup callback for a flushed database map. It defaults to `1024`; values below `1` are treated as `1`.

```ini
relay.flush_batch_size = 1024
```

Set this directive in your INI configuration before PHP starts and restart PHP workers after changing it. It cannot be changed with `ini_set()`.

- Smaller batches reduce work per callback and can help limit cleanup pauses, but keep retired cache memory allocated for longer.
- Larger batches reclaim more entries per callback, making memory available for reuse sooner at the cost of longer cleanup pauses.

Start with the default and measure application latency and memory usage during cache flushes and repopulation. The batch size counts entries, not bytes or milliseconds, so it does not impose a fixed latency limit. Reclaimed space becomes available within Relay's shared memory allocation; flushing does not shrink `relay.maxmemory` or return that allocation to the operating system.

This setting controls memory cleanup after a flush; it does not delay cache invalidation. For example, [`Relay\Relay::flushMemory()`](https://docs.relay.so/api/develop/Relay/Relay.html#method_flushMemory) flushes Relay's local cache without deleting data from Redis. Since v0.50.0, it covers all existing databases in the requested scope, including those without active writers, while memory reclamation can continue after the call returns.

## `relay.max_endpoint_dbs`

Removed in the development version after v0.50.0. Each endpoint now has one shared cache, and this directive is ignored. Replace the old cache-count tuning with [`relay.max_db_writers`](#relaymax_db_writers).

If your deployment previously used multiple caches per endpoint, you may need to increase `relay.max_db_writers` to preserve your intended total writer capacity. Relay does not multiply the old cache count into the writer limit automatically. Benchmark the new setting and size [`relay.key_leases`](#relaykey_leases) for the total writers across endpoints.

## `relay.cap_endpoint_dbs`

Removed alongside `relay.max_endpoint_dbs` and ignored. Relay no longer caps the number of endpoint caches to the CPU count; each endpoint has one shared cache. Tune the writer limit using the throughput and utilization guidance above.
