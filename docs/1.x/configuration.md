---
title: Configuration
---

# Configuration

Relay provides many configuration directives and the `relay.ini` file can be located by running:

```bash
php --ini
```

It’s recommended to at least adjust the `relay.maxmemory` and `relay.eviction_policy` directives. The `relay.max_db_writers` directive controls how many connections can populate each endpoint’s shared cache. See [Performance](/docs/1.x/performance) for tuning throughput, CPU usage, and locking.

If you're running a licensed binary, be sure to set the `relay.key` and `relay.environment` as well.

## Memory limits

Relay will allocate what `relay.maxmemory` is set to when PHP starts, even if no `relay.key` is set. After 60 minutes of runtime (if no valid license was set), Relay will downsize the allocated memory to 16 MB.

## Disabling the cache

Sometimes you may wish to install the Relay extension, but not have it allocate memory, either because you want to only use it as a faster alternative to PhpRedis, or to keep it dormant for future use.

To disable all in-memory caching and memory allocation `relay.maxmemory` can be set to `0`.

## Shared cache configuration

In the development version after v0.50.0, each endpoint has one cache shared across PHP workers, with a separate map for each Redis database. `relay.max_db_writers` limits the connections that can populate it, and `relay.key_leases` sizes the shared pool of writer leases across all endpoints.

The former `relay.max_endpoint_dbs` and `relay.cap_endpoint_dbs` directives have been removed and are ignored in this version. See [Performance](/docs/1.x/performance#relaymax_endpoint_dbs) for migration guidance.

Set `relay.max_db_writers`, `relay.key_leases`, and `relay.databases` in your INI configuration before PHP starts. Restart PHP workers after changing them.

## Configuration directives

| Directive                         | Default          | Description                                                         |
| --------------------------------- | ---------------- | ------------------------------------------------------------------- |
| `relay.key`                       |                  | Relay license key. Without a license key Relay will throttle to 16MB memory one hour after startup. May also be set via `RELAY_KEY` environment variable. |
| `relay.environment`               | `development`    | The environment Relay is running in. Supported values: `production`, `staging`, `testing`, `development` |
| `relay.maxmemory`                 | `16MB`           | How much memory Relay allocates on startup. This value can either be a number like `134217728` [or a unit](https://php.net/manual/faq.using.php#faq.using.shorthandbytes) (e.g. `128M`) like `memory_limit`. Relay will allocate at least 16M for overhead structures. Set to `0` to disable in-memory caching and use as client only. |
| `relay.maxmemory_pct`             | `95`             | At what percentage of used memory should Relay start evicting keys. |
| `relay.eviction_policy`           | `noeviction`     | How should Relay evict keys. This has been designed to mirror Redis’ options. Supported values: `noeviction`, `lru`, and `random` |
| `relay.eviction_sample_keys`      | `128`            | How many keys should we scan each time we process evictions. |
| `relay.flush_batch_size`          | `1024`           | Maximum cached entries reclaimed per cleanup callback for a flushed database map. Minimum `1`. Available since v0.50.0; set before PHP starts. Smaller batches reduce work per callback but retain retired cache memory longer. See [Performance](/docs/1.x/performance#relayflush_batch_size). |
| `relay.databases`                 | `16`             | The number of Redis database maps per endpoint cache. This setting should match the `databases` setting in your `redis.conf`. |
| `relay.max_db_writers`            | `4`              | The maximum number of tracking connections allowed to populate each endpoint’s shared cache. Supported values: `1–1024`. Other connections can read the cache and fall back to Redis, but cannot populate it. See [Performance](/docs/1.x/performance#relaymax_db_writers). |
| `relay.key_leases`                | `32`             | The shared pool of writer leases across all endpoints. Relay allocates at least `max_db_writers` slots, rounded up to a power of two, with a minimum of `16` and a maximum of `32768`. Size this for the total writers across Redis servers and cluster nodes. See [Performance](/docs/1.x/performance#relaykey_leases). |
| `relay.locks.allocator`           | `adaptive-mutex` | Locking mechanism used for the allocator. Supported values: `spinlock`, `mutex`, `adaptive-mutex`. See [Performance](/docs/1.x/performance). |
| `relay.locks.cache`               | `adaptive-mutex` | Locking mechanism used for the in-memory cache (databases). Supported values: `spinlock`, `mutex`, `adaptive-mutex`. See [Performance](/docs/1.x/performance). |
| `relay.default_pconnect`          | `1`              | Default to using a persistent connection when calling `connect()`. |
| `relay.initial_readers`           | `128`            | The number of epoch readers allocated on startup. |
| `relay.invalidation_poll_freq`    | `5`              | How often (in microseconds) Relay should proactively check the connection for invalidation messages from Redis/Valkey. |
| `relay.loglevel`                  | `off`            | Whether Relay should log debug information. Supported levels: `debug`, `verbose`, `notice`, `error`, `off` |
| `relay.logfile`                   |                  | The log destination. Supports `stderr` or an absolute path. |
| `relay.prefault`                  | `off`            | Whether Relay should fault in the shared memory segment during startup, instead of lazily on first use. Supported values: `off`, `populate` (uses `MAP_POPULATE`, Linux only), `touch` (writes to every page). |

## Cluster directives

The health-check backoff settings are available since v0.50.0. See [Cluster health checks](/docs/1.x/connections#cluster-health-checks) for how they affect node recovery and interact with per-node read timeouts.

| Directive                              | Default          | Description                                                         |
| -------------------------------------- | ---------------- | ------------------------------------------------------------------- |
| `relay.cluster.seeds`                  |                  | The list of cluster nodes addresses grouped by cluster name, which will be used to initialize each cluster, encoded as URL query string, e.g. `cluster1[]=tcp://127.0.0.1:7000&cluster2[]=tcp://127.0.0.1:8000` |
| `relay.cluster.auth`                   |                  | The list of credentials for each cluster, encoded as URL query string. Password string or username/password pairs may be used, e.g. `cluster1=secret&cluster2[]=username&cluster2[]=secret` |
| `relay.cluster.timeout`                |                  | The maximum number of seconds Relay will wait while establishing connection to a single cluster node. |
| `relay.cluster.read_timeout`           |                  | The maximum number of seconds Relay will wait while reading from a cluster node. |
| `relay.cluster.slot_cache_expiry`      |                  | The TTL of the cluster slot cache. |
| `relay.cluster.shard_health_wait_base` | `1`              | Base delay in seconds for unhealthy-node health checks. Nonpositive values use `1`. |
| `relay.cluster.shard_health_wait_cap` | `60`              | Maximum health-check delay in seconds. Values below the base are raised to the base. |
| `relay.cluster.shard_health_wait_strategy` | `equal-jitter` | Health-check backoff algorithm. Supported values: `default`, `decorrelated-jitter`, `full-jitter`, `equal-jitter`, `exponential`, `uniform`, `constant`. |
| `relay.cluster.shard_health_wait_time` | `0`              | Legacy override: a positive value selects a fixed health-check delay in seconds, overriding the base, cap, and strategy. Zero or negative values use the backoff settings above. |
| `relay.session.locking_enabled`        | `0`              | Whether to enable session locking to avoid race conditions and keep session data consistent across requests. |
| `relay.session.lock_expire`            | `0`              | The number of seconds Relay will try to acquire lock. When value is zero or negative `max_execution_time` will be used. |
| `relay.session.lock_retries`           | `0`              | The number of attempts Relay will try to acquire lock. If value is zero or negative `100` will be used to be compatible with PhpRedis. |
| `relay.session.lock_wait_time`         | `0`              | The number of microseconds Relay will wait between each attempt to acquire lock. If value is zero or negative `20000` will be used to be compatible with PhpRedis. |
| `relay.session.compression`            | `none`           | Compression algorithm used for session data. Supported values: `lzf`, `lz4`, `zstd` and `none` |
| `relay.session.compression_level`      | `0`              | The used compression level. A value of `0` means the algorithm default compression level will be used. |
