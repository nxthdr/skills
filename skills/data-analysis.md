# Skill: Data Analysis with nxthdr

This skill teaches how to query and analyze the nxthdr open datasets in ClickHouse: probing results, BGP routing updates, network traffic flows, and IP enrichment data.

## Data Licensing

All nxthdr datasets are released under the **Public Domain Dedication and License v1.0** ([PDDL](http://opendatacommons.org/licenses/pddl/1.0/)). You are free to use, share, and build upon the data for any purpose, including commercial use, without attribution requirements.

## Access & Authentication

```
Endpoint: https://nxthdr.dev/api/query/
Username: read
Password: read
```

These credentials are intentionally public. Authentication exists solely to prevent automated scraping.

Compatible clients: curl, ClickHouse CLI, Python/R/JavaScript HTTP libraries, Jupyter notebooks, Datasette, or any tool that speaks HTTP + SQL.

## Available Datasets

| Database | Table | Description | Source | GitHub |
|----------|-------|-------------|--------|--------|
| `saimiris` | `replies` | Active measurement results (traceroute, ping) | [saimiris](https://github.com/nxthdr/saimiris) agents | [schema](https://github.com/nxthdr/infrastructure/blob/main/clickhouse-tables/saimiris/saimiris.sql) |
| `bmp` | `updates` | BGP routing updates via BMP | [risotto](https://github.com/nxthdr/risotto) collector | [schema](https://github.com/nxthdr/infrastructure/blob/main/clickhouse-tables/bmp/bmp.sql) |
| `flows` | `records` | sFlow v5 sampled network flows (IPv6 only) | [pesto](https://github.com/nxthdr/pesto) collector | [schema](https://github.com/nxthdr/infrastructure/blob/main/clickhouse-tables/flows/flows.sql) |
| `ipinfo` | (dictionaries) | IP-to-country/ASN enrichment | [IPinfo](https://ipinfo.io/) free database via [mmdb-to-clickhouse](https://github.com/maxmouchet/mmdb-to-clickhouse) | [schema](https://github.com/nxthdr/infrastructure/blob/main/clickhouse-tables/ipinfo/ipinfo.sql) |

All infrastructure is open-source at [github.com/nxthdr/infrastructure](https://github.com/nxthdr/infrastructure).

## Data Pipeline

All three main datasets follow the same architecture:

```
Source → Collector → Redpanda (Kafka) → ClickHouse (Kafka engine → Materialized View → MergeTree)
```

- **Probing**: Saimiris agents → Redpanda (`saimiris-replies`) → `saimiris.replies`
- **BGP**: BIRD routers → Risotto (BMP) → Redpanda (`risotto-updates`) → `bmp.updates`
- **Flows**: Network devices → Pesto (sFlow) → Redpanda (`pesto-sflow`) → `flows.records`

All messages use [Cap'n Proto](https://capnproto.org/) serialization.

## Table Schemas

### `saimiris.replies` — Probing Results

| Column | Type | Description |
|--------|------|-------------|
| `date` | Date | Partition key |
| `time_received_ns` | DateTime64(9) | Timestamp (nanosecond precision) |
| `agent_id` | String | Vantage point identifier (e.g., `vltcdg01`) |
| `probe_src_addr` | IPv6 | Source address of the sent probe |
| `probe_dst_addr` | IPv6 | Destination address of the probe |
| `probe_src_port` | UInt16 | Source port |
| `probe_dst_port` | UInt16 | Destination port |
| `probe_ttl` | UInt8 | TTL / hop limit set in the probe |
| `probe_protocol` | UInt8 | Protocol (58 = ICMPv6, 17 = UDP) |
| `probe_size` | UInt16 | Probe packet size |
| `probe_id` | UInt16 | Probe identifier |
| `reply_src_addr` | IPv6 | Source of the reply (the hop that responded) |
| `reply_dst_addr` | IPv6 | Destination of the reply |
| `reply_ttl` | UInt8 | TTL in the reply packet |
| `reply_quoted_ttl` | UInt8 | Quoted TTL from ICMP error payload |
| `reply_protocol` | UInt8 | Reply protocol |
| `reply_icmp_type` | UInt8 | ICMP type (129=echo reply, 3=dest unreachable, 11=time exceeded) |
| `reply_icmp_code` | UInt8 | ICMP code |
| `reply_size` | UInt16 | Reply packet size |
| `reply_id` | UInt16 | Reply identifier |
| `reply_mpls_labels` | Array(Tuple(UInt32, UInt8, UInt8, UInt8)) | MPLS label stack |
| `rtt` | UInt16 | Round-trip time in **tenths of milliseconds** |

**RTT unit**: stored as 0.1 ms. Use `rtt / 10.0 AS rtt_ms` for milliseconds. Max value: 6553.5 ms.

**Order key**: `(time_received_ns, agent_id, probe_protocol, probe_src_addr, probe_dst_addr, probe_src_port, probe_dst_port, probe_ttl)`

### `bmp.updates` — BGP Routing Updates

| Column | Type | Description |
|--------|------|-------------|
| `date` | Date | Partition key |
| `time_received_ns` | DateTime64(9) | Timestamp (nanosecond precision) |
| `time_bmp_header_ns` | DateTime64(9) | BMP header timestamp |
| `router_addr` | IPv6 | Router sending the BMP message |
| `router_port` | UInt16 | Router port |
| `peer_addr` | IPv6 | BGP peer address |
| `peer_bgp_id` | IPv4 | Peer BGP router ID |
| `peer_asn` | UInt32 | Peer AS number |
| `prefix_addr` | IPv6 | Announced/withdrawn prefix address |
| `prefix_len` | UInt8 | Prefix length |
| `announced` | Bool | true = announcement, false = withdrawal |
| `synthetic` | Bool | Synthetic update (e.g., session reset) |
| `is_post_policy` | Bool | Post-policy update |
| `is_adj_rib_out` | Bool | Adj-RIB-Out update |
| `origin` | String | BGP origin attribute (IGP, EGP, INCOMPLETE) |
| `as_path` | Array(UInt32) | AS path |
| `next_hop` | IPv6 | BGP next hop |
| `multi_exit_disc` | UInt32 | MED value |
| `local_preference` | UInt32 | Local preference |
| `only_to_customer` | UInt32 | OTC attribute (RFC 9234) |
| `atomic_aggregate` | Bool | Atomic aggregate flag |
| `aggregator_asn` | UInt32 | Aggregator ASN |
| `aggregator_bgp_id` | IPv4 | Aggregator BGP ID |
| `communities` | Array(Tuple(UInt32, UInt16)) | Standard BGP communities |
| `extended_communities` | Array(Tuple(UInt8, UInt8, String)) | Extended communities |
| `large_communities` | Array(Tuple(UInt32, UInt32, UInt32)) | Large BGP communities |
| `originator_id` | IPv4 | Originator ID |
| `cluster_list` | Array(UInt32) | Cluster list |

**Order key**: `(time_received_ns, router_addr, peer_addr, prefix_addr, prefix_len)`

### `flows.records` — Network Traffic Flows

| Column | Type | Description |
|--------|------|-------------|
| `date` | Date | Partition key |
| `time_received_ns` | DateTime64(9) | Timestamp (nanosecond precision) |
| `sequence_num` | UInt32 | Datagram sequence number |
| `sampling_rate` | UInt64 | sFlow sampling rate (e.g., 1-in-N) |
| `sampler_address` | IPv6 | sFlow agent address |
| `sampler_port` | UInt16 | sFlow agent port |
| `src_addr` | IPv6 | Source IP |
| `dst_addr` | IPv6 | Destination IP |
| `src_port` | UInt32 | Source port |
| `dst_port` | UInt32 | Destination port |
| `protocol` | UInt32 | IP protocol number |
| `etype` | UInt32 | Ethernet type |
| `packet_length` | UInt32 | Raw packet length |
| `bytes` | UInt64 | Byte count (= packet_length for single sample) |
| `packets` | UInt64 | Packet count (always 1 per row) |

**Estimating real traffic**: multiply `bytes` and `packets` by `sampling_rate`:
```sql
sum(bytes * sampling_rate) AS estimated_total_bytes
```

**Order key**: `(time_received_ns, src_addr, dst_addr)`

**Note**: Only IPv6 flows are stored. IPv4 flows are mapped to IPv4-mapped IPv6 addresses.

## IPinfo Enrichment

[IPinfo](https://ipinfo.io/)'s free Country & ASN database is loaded daily into ClickHouse via [mmdb-to-clickhouse](https://github.com/maxmouchet/mmdb-to-clickhouse). The data is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

### `country_asn()` lookup function

```sql
SELECT country_asn(toIPv6('8.8.8.8'), 'as_name')  -- 'Google LLC'
SELECT country_asn(toIPv6('8.8.8.8'), 'country')   -- 'US'
```

### Available attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `asn` | Autonomous System Number | `15169` |
| `as_name` | AS organization name | `Google LLC` |
| `as_domain` | AS domain | `google.com` |
| `country` | Country code (ISO 3166) | `US` |
| `country_name` | Full country name | `United States` |
| `continent` | Continent code | `NA` |
| `continent_name` | Full continent name | `North America` |

### `asn_to_name` dictionary (ASN number lookups)

When you have an ASN number (e.g., from `as_path`) and no IP address:

```sql
SELECT dictGet('ipinfo.asn_to_name', 'as_name', toUInt64(13335))
-- 'Cloudflare, Inc.'
```

### Low-level dictionary access

The `country_asn()` function is syntactic sugar over two dictionaries. For advanced use (e.g., in GROUP BY), use them directly:

```sql
SELECT
    dictGetString('ipinfo.country_asn_val', 'as_name',
        dictGetUInt64('ipinfo.country_asn_net', 'pointer',
            tuple(toIPv6(reply_src_addr)))) AS as_name
FROM saimiris.replies
```

## Querying with curl

All examples below use curl. Append `FORMAT CSVWithNames` for CSV output, `FORMAT JSONEachRow` for JSON, or `FORMAT Pretty` for human-readable output.

```bash
curl -s -X POST "https://nxthdr.dev/api/query/" \
  -u "read:read" \
  -H "Content-Type: text/plain" \
  -d "<SQL QUERY>"
```

## Example Queries

### Probing Dataset

#### Top destinations by RTT from a specific agent

```sql
SELECT
    probe_dst_addr,
    count() AS measurements,
    round(avg(rtt / 10.0), 2) AS avg_rtt_ms
FROM saimiris.replies
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
  AND agent_id = 'vltcdg01'
GROUP BY probe_dst_addr
ORDER BY avg_rtt_ms DESC
LIMIT 10
FORMAT CSVWithNames
```

#### Traceroute reconstruction

```sql
SELECT
    probe_dst_addr,
    probe_ttl AS hop,
    reply_src_addr AS hop_addr,
    rtt / 10.0 AS rtt_ms,
    reply_icmp_type
FROM saimiris.replies
WHERE probe_src_addr = toIPv6('2a0e:97c0:8a4:42:abcd::1')
  AND reply_icmp_type IN (11, 129, 3)
ORDER BY probe_dst_addr, probe_ttl
FORMAT CSVWithNames
```

#### RTT comparison across vantage points

```sql
SELECT
    agent_id,
    probe_dst_addr,
    round(avg(rtt / 10.0), 1) AS avg_rtt_ms,
    count() AS replies
FROM saimiris.replies
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
  AND probe_ttl = 64
GROUP BY agent_id, probe_dst_addr
ORDER BY probe_dst_addr, agent_id
FORMAT CSVWithNames
```

#### Enrich probing results with geolocation

```sql
SELECT
    probe_dst_addr,
    country_asn(probe_dst_addr, 'country') AS country,
    country_asn(probe_dst_addr, 'as_name') AS as_name,
    round(avg(rtt / 10.0), 2) AS avg_rtt_ms
FROM saimiris.replies
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY probe_dst_addr
ORDER BY avg_rtt_ms DESC
LIMIT 10
FORMAT CSVWithNames
```

#### MPLS tunnel detection

```sql
SELECT
    probe_dst_addr,
    probe_ttl,
    reply_src_addr,
    reply_mpls_labels,
    rtt / 10.0 AS rtt_ms
FROM saimiris.replies
WHERE time_received_ns >= now() - INTERVAL 1 DAY
  AND length(reply_mpls_labels) > 0
ORDER BY probe_dst_addr, probe_ttl
LIMIT 50
FORMAT CSVWithNames
```

### BGP Dataset

#### Recent updates for a prefix

```sql
SELECT
    time_received_ns,
    router_addr,
    peer_addr,
    peer_asn,
    announced,
    as_path
FROM bmp.updates
WHERE prefix_addr = toIPv6('2606:4700::')
  AND prefix_len = 48
  AND time_received_ns >= now() - INTERVAL 1 HOUR
ORDER BY time_received_ns DESC
LIMIT 20
FORMAT CSVWithNames
```

#### Top origin ASNs by prefix count

```sql
SELECT
    as_path[-1] AS origin_asn,
    dictGet('ipinfo.asn_to_name', 'as_name', toUInt64(origin_asn)) AS as_name,
    uniqExact(concat(toString(prefix_addr), '/', toString(prefix_len))) AS prefix_count
FROM bmp.updates
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
  AND announced = true
  AND length(as_path) > 0
GROUP BY origin_asn
ORDER BY prefix_count DESC
LIMIT 20
FORMAT CSVWithNames
```

#### AS path length distribution

```sql
SELECT
    length(as_path) AS path_length,
    count() AS updates
FROM bmp.updates
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
  AND announced = true
  AND length(as_path) > 0
GROUP BY path_length
ORDER BY path_length
FORMAT CSVWithNames
```

#### Prefixes with the most BGP communities

```sql
SELECT
    concat(toString(prefix_addr), '/', toString(prefix_len)) AS prefix,
    max(length(communities)) AS n_communities,
    max(length(large_communities)) AS n_large_communities
FROM bmp.updates
WHERE time_received_ns >= now() - INTERVAL 1 DAY
  AND announced = true
GROUP BY prefix
ORDER BY n_communities DESC
LIMIT 10
FORMAT CSVWithNames
```

#### Announcement vs withdrawal rate over time

```sql
SELECT
    toStartOfMinute(time_received_ns) AS minute,
    countIf(announced = true) AS announcements,
    countIf(announced = false) AS withdrawals
FROM bmp.updates
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute
FORMAT CSVWithNames
```

#### Enrich AS paths with AS names

```sql
WITH as_path[1] AS first_asn
SELECT
    first_asn,
    dictGet('ipinfo.asn_to_name', 'as_name', toUInt64(first_asn)) AS as_name,
    count() AS updates
FROM bmp.updates
WHERE time_received_ns >= now() - INTERVAL 1 DAY
  AND announced = true
  AND first_asn != 0
GROUP BY first_asn
ORDER BY updates DESC
LIMIT 10
FORMAT CSVWithNames
```

### Traffic Dataset

#### Top destinations by estimated bandwidth

```sql
SELECT
    dst_addr,
    sum(bytes * sampling_rate) / (3600) AS avg_bytes_per_second,
    count() AS flow_samples,
    sum(bytes * sampling_rate) AS estimated_total_bytes
FROM flows.records
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY dst_addr
ORDER BY avg_bytes_per_second DESC
LIMIT 10
FORMAT CSVWithNames
```

#### Protocol distribution

```sql
SELECT
    protocol,
    count() AS samples,
    sum(bytes * sampling_rate) AS estimated_bytes,
    round(100.0 * estimated_bytes / sum(estimated_bytes) OVER (), 2) AS pct
FROM flows.records
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY protocol
ORDER BY estimated_bytes DESC
FORMAT CSVWithNames
```

#### Traffic by source AS

```sql
SELECT
    country_asn(src_addr, 'asn') AS src_asn,
    country_asn(src_addr, 'as_name') AS src_as_name,
    count() AS flow_samples,
    sum(bytes * sampling_rate) AS estimated_bytes
FROM flows.records
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY src_asn, src_as_name
ORDER BY estimated_bytes DESC
LIMIT 10
FORMAT CSVWithNames
```

#### Traffic time series

```sql
SELECT
    toStartOfMinute(time_received_ns) AS minute,
    sum(bytes * sampling_rate) AS estimated_bytes,
    count() AS samples
FROM flows.records
WHERE time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute
FORMAT CSVWithNames
```

### Cross-Dataset Queries

#### Correlate BGP paths with measured latency

```sql
-- Step 1: Get current AS paths for a prefix
SELECT
    peer_addr,
    as_path,
    time_received_ns
FROM bmp.updates
WHERE prefix_addr = toIPv6('2001:4860:4860::')
  AND prefix_len = 32
  AND announced = true
ORDER BY time_received_ns DESC
LIMIT 5
FORMAT CSVWithNames

-- Step 2: Get probing latency to the same prefix
SELECT
    agent_id,
    probe_dst_addr,
    round(avg(rtt / 10.0), 2) AS avg_rtt_ms,
    count() AS probes
FROM saimiris.replies
WHERE toString(probe_dst_addr) LIKE '2001:4860:4860:%'
  AND time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY agent_id, probe_dst_addr
FORMAT CSVWithNames
```

#### Correlate traffic volume with routing changes

```sql
-- Traffic to a prefix over time
SELECT
    toStartOfFiveMinutes(time_received_ns) AS ts,
    sum(bytes * sampling_rate) AS estimated_bytes
FROM flows.records
WHERE toString(dst_addr) LIKE '2a06:de00:5b:%'
  AND time_received_ns >= now() - INTERVAL 6 HOUR
GROUP BY ts
ORDER BY ts
FORMAT CSVWithNames

-- BGP updates for the same prefix in the same window
SELECT
    toStartOfFiveMinutes(time_received_ns) AS ts,
    countIf(announced = true) AS announcements,
    countIf(announced = false) AS withdrawals
FROM bmp.updates
WHERE prefix_addr = toIPv6('2a06:de00:5b::')
  AND prefix_len = 48
  AND time_received_ns >= now() - INTERVAL 6 HOUR
GROUP BY ts
ORDER BY ts
FORMAT CSVWithNames
```

## Typical Analysis Patterns

### Latency analysis
1. Query `saimiris.replies` filtered by agent, destination, or time window
2. Compute avg/min/max/percentiles of `rtt / 10.0`
3. Enrich with `country_asn()` for geographic context
4. Compare across agents for multi-vantage-point analysis

### Topology discovery
1. Query traceroute results ordered by `probe_ttl`
2. Reconstruct hop-by-hop paths from `reply_src_addr`
3. Enrich each hop with ASN for AS-level topology
4. Detect MPLS tunnels via `reply_mpls_labels`

### BGP routing analysis
1. Query `bmp.updates` for a prefix or ASN of interest
2. Track `as_path` changes over time
3. Measure convergence by counting updates per time interval
4. Analyze community propagation via `communities` and `large_communities`
5. Enrich with `asn_to_name` dictionary for readable output

### Traffic profiling
1. Query `flows.records` with `sampling_rate` correction
2. Group by src/dst ASN, protocol, or port
3. Build time series with `toStartOfMinute()` or `toStartOfFiveMinutes()`
4. Correlate with BGP changes in `bmp.updates`

### Joint peering + probing
1. Announce prefix via PeerLab (see peering-experiments skill)
2. Send probes via Saimiris to targets
3. Compare control plane (`bmp.updates` AS paths) with data plane (`saimiris.replies` measured paths)
4. Correlate with traffic volume in `flows.records`

## Limitations

### Data retention
- **7-day TTL** on all main tables (`saimiris.replies`, `bmp.updates`, `flows.records`)
- **30-day TTL** on IPinfo enrichment tables
- Data is automatically deleted after the retention period — there is no historical archive

### Query limits (read-only user)
The public `read` user goes through [chproxy](https://github.com/contentsquare/chproxy) with these constraints:

| Limit | Value |
|-------|-------|
| Max query execution time | **120 seconds** |
| Max memory per query | **4 GB** |
| Max total memory (all queries) | **8 GB** |
| Max rows to read | **100 million** |
| Max bytes to read | **10 GB** |
| Max threads per query | **2** |
| Max concurrent queries | **6** |

If a query hits a limit, ClickHouse returns an error. Strategies to stay within limits:
- Add time filters (`WHERE time_received_ns >= now() - INTERVAL ...`) to narrow the scan
- Use `LIMIT` to cap output rows
- Avoid `SELECT *` — only select needed columns
- Use `toDate(time_received_ns)` partition pruning (e.g., `WHERE date = today()`)
- Pre-aggregate with `GROUP BY` instead of returning raw rows

### Server resources
- **Single-node** ClickHouse (no cluster) on a 16 GB RAM server
- **8 max concurrent queries** at the server level
- CPU and memory shared with other services (Redpanda, PostgreSQL, Grafana, etc.)

### IPv6 column filtering
ClickHouse IPv6 columns do not support `LIKE` directly. Use:
- `toString(ip_column) LIKE '2a0e:97c0:%'` for pattern matching
- `ip_column = toIPv6('...')` for exact match
- `ip_column IN (toIPv6('...'), toIPv6('...'))` for multiple values

### sFlow sampling
Traffic data in `flows.records` is **sampled** (not every packet). Always multiply by `sampling_rate` for volume estimates. Absolute traffic values are approximations.

### BMP specifics
- `bmp.updates` contains both post-policy and pre-policy updates — filter with `is_post_policy` if needed
- `synthetic = true` marks updates generated during session resets, not real routing changes
- The `announced` field distinguishes announcements (true) from withdrawals (false)
- Communities are stored as tuples, not strings — use tuple comparison: `hasAll(communities, [(65000, 100)])`

## Tools and Integrations

- **MCP server**: Use [mcp-clickhouse](https://github.com/ClickHouse/mcp-clickhouse) for natural language queries from AI assistants. Config: host `clickhouse.nxthdr.dev`, user `read`, password `read`.
- **Python**: Use `clickhouse-connect` or `requests` with the HTTP endpoint
- **Marimo notebooks**: Recommended for interactive analysis and sharing — see [publication guidelines](https://docs.nxthdr.dev/docs/datasets/guidelines/)
- **Datasette**: For lightweight data exploration and sharing
- **prowl**: Python library for generating probes ([github.com/nxthdr/prowl](https://github.com/nxthdr/prowl))
- **nxthdr CLI**: For sending probes and managing peering ([github.com/nxthdr/cli](https://github.com/nxthdr/cli))
