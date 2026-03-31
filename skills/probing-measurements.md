# Skill: Probing Measurements with nxthdr

This skill teaches how to run active Internet measurements using the nxthdr CLI and analyze the results from ClickHouse.

## Prerequisites

- The [nxthdr CLI](https://github.com/nxthdr/cli) installed
- Authenticated: `nxthdr login`

## Platform Overview

**Saimiris** is nxthdr's probing platform. It sends IPv6 probes (ICMPv6 or UDP) from distributed vantage points and stores replies in a public ClickHouse database.

Key characteristics:
- **IPv6 only** (AS215011)
- **10,000 probes/day** credit limit per user
- **7-day data retention** in ClickHouse
- **Unicast and anycast** source addressing
- Probes follow the [Caracal format](https://dioptra-io.github.io/caracal/usage/)

## Step 1: Discover Available Agents

```bash
nxthdr probing agents
```

Each agent has two IPv6 `/48` prefixes:
- **Unicast prefix**: advertised only from that agent (unique return path)
- **Anycast prefix**: `2a0e:97c0:8a0::/48`, advertised from all agents simultaneously (reply routed to BGP-closest agent)

## Step 2: Check Credits

```bash
nxthdr probing credits
```

## Step 3: Create a Probe File

Each line defines one probe with 5 comma-separated fields:

```
dst_addr,src_port,dst_port,ttl,protocol
```

| Field | Description |
|-------|-------------|
| `dst_addr` | IPv6 target address |
| `src_port` | Source port (0-65535). For ICMP, varies flow ID via checksum |
| `dst_port` | Destination port (0-65535). Only used for UDP |
| `ttl` | Time-To-Live / Hop Limit (1-255) |
| `protocol` | `icmpv6` or `udp` (case-insensitive) |

### Ping example (single probe, high TTL)

```csv
2001:4860:4860::8888,24000,0,64,icmpv6
```

### Traceroute example (incrementing TTL)

```csv
2001:4860:4860::8888,24000,33434,1,udp
2001:4860:4860::8888,24000,33434,2,udp
2001:4860:4860::8888,24000,33434,3,udp
...
2001:4860:4860::8888,24000,33434,32,udp
```

### Using prowl for probe generation

[prowl](https://github.com/nxthdr/prowl) generates probes from a target list:

```bash
# targets.csv format: prefix,protocol,min_ttl,max_ttl,n_flows
echo "2001:4860:4860::8888/128,ICMPv6,1,32,3" > targets.csv
prowl --tool traceroute --mapper sequential targets.csv > probes.csv
```

## Step 4: Send Probes

### Single agent

```bash
nxthdr probing send --agent vltcdg01 probes.csv
```

### Multiple agents (same probes from different vantage points)

```bash
nxthdr probing send --agent vltcdg01 --agent vltewr01 --agent vltsgp01 probes.csv
```

When sending from multiple agents, the CLI generates correlated source IPs that share the same lower 48 bits. This allows querying all agents' results with a single source IP filter.

### From stdin (pipe from prowl)

```bash
prowl --tool traceroute --mapper sequential targets.csv | nxthdr probing send --agent vltcdg01
```

### Output

The CLI prints:
1. The measurement ID
2. The derived source IP per agent
3. A ready-to-run `nxthdr probing results` command

```
measurement: abc123
source IPs:
  vltcdg01: 2a0e:97c0:8a4:42:abcd::1
  vltewr01: 2a0e:97c0:8a3:42:abcd::1
hint: nxthdr probing results --src-ip 2a0e:97c0:8a4:42:abcd::1 --src-ip 2a0e:97c0:8a3:42:abcd::1
```

## Step 5: Check Measurement Status

```bash
nxthdr probing measurement-status <measurement-id>
```

## Step 6: Retrieve Results

### Via CLI

```bash
nxthdr probing results --src-ip 2a0e:97c0:8a4:42:abcd::1
```

Optional time window:
```bash
nxthdr probing results --src-ip 2a0e:97c0:8a4:42:abcd::1 --since "2026-03-19 21:00:00" --until "2026-03-19 22:00:00"
```

### Via ClickHouse directly

All measurement data is publicly available:

```
Endpoint: https://nxthdr.dev/api/query/
Username: read
Password: read
```

## ClickHouse Query Reference

### Table: `saimiris.replies`

| Column | Type | Description |
|--------|------|-------------|
| `time_received_ns` | UInt64 | Timestamp in nanoseconds |
| `agent_id` | String | Agent that received the reply |
| `probe_src_addr` | IPv6 | Source address of the sent probe |
| `probe_dst_addr` | IPv6 | Destination address of the probe |
| `probe_src_port` | UInt16 | Source port of the probe |
| `probe_dst_port` | UInt16 | Destination port of the probe |
| `probe_ttl` | UInt8 | TTL set in the probe |
| `reply_src_addr` | IPv6 | Source address of the reply (the hop that responded) |
| `reply_icmp_type` | UInt8 | ICMP type (129=echo reply, 3=dest unreachable, 11=time exceeded) |
| `reply_icmp_code` | UInt8 | ICMP code |
| `rtt` | UInt16 | Round-trip time in **tenths of milliseconds** (divide by 10 for ms) |

### Important notes

- **RTT unit**: stored as tenths of milliseconds. Use `rtt / 10.0 AS rtt_ms` for milliseconds.
- **IPv6 column filtering**: use `toString()` for LIKE patterns or `toIPv6()` for exact matches. Direct `LIKE` on IPv6 columns fails.
- **Data retention**: 7 days.

### Example queries

#### Basic: list replies for a measurement

```sql
SELECT
    agent_id,
    probe_dst_addr,
    reply_src_addr,
    probe_ttl,
    reply_icmp_type,
    rtt / 10.0 AS rtt_ms
FROM saimiris.replies
WHERE probe_src_addr IN (toIPv6('2a0e:97c0:8a4:42:abcd::1'))
ORDER BY probe_dst_addr, probe_ttl
FORMAT CSVWithNames
```

#### Traceroute reconstruction

```sql
SELECT
    probe_dst_addr,
    probe_ttl AS hop,
    reply_src_addr AS hop_addr,
    rtt / 10.0 AS rtt_ms
FROM saimiris.replies
WHERE probe_src_addr IN (toIPv6('2a0e:97c0:8a4:42:abcd::1'))
  AND reply_icmp_type IN (11, 129, 1)  -- time exceeded, echo reply, dest unreachable
ORDER BY probe_dst_addr, probe_ttl
FORMAT CSVWithNames
```

#### RTT comparison across agents

```sql
SELECT
    agent_id,
    probe_dst_addr,
    round(avg(rtt / 10.0), 1) AS avg_rtt_ms,
    count() AS replies
FROM saimiris.replies
WHERE probe_src_addr IN (
    toIPv6('2a0e:97c0:8a4:42:abcd::1'),
    toIPv6('2a0e:97c0:8a3:42:abcd::1')
)
AND probe_ttl = 64
GROUP BY agent_id, probe_dst_addr
ORDER BY probe_dst_addr, agent_id
FORMAT CSVWithNames
```

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

#### GeoIP / ASN enrichment

IPInfo dictionaries are available in ClickHouse for IP-to-country and IP-to-ASN lookups:

```sql
SELECT
    reply_src_addr,
    dictGetString('ipinfo.country_asn_val', 'asn',
        dictGetUInt64('ipinfo.country_asn_net', 'pointer',
            tuple(toIPv6(reply_src_addr)))) AS asn,
    dictGetString('ipinfo.country_asn_val', 'as_name',
        dictGetUInt64('ipinfo.country_asn_net', 'pointer',
            tuple(toIPv6(reply_src_addr)))) AS as_name,
    dictGetString('ipinfo.country_asn_val', 'country',
        dictGetUInt64('ipinfo.country_asn_net', 'pointer',
            tuple(toIPv6(reply_src_addr)))) AS country,
    count() AS replies
FROM saimiris.replies
WHERE probe_src_addr IN (toIPv6('2a0e:97c0:8a4:42:abcd::1'))
GROUP BY reply_src_addr, asn, as_name, country
ORDER BY replies DESC
FORMAT CSVWithNames
```

#### Anycast catchment analysis

When probes are sent with an anycast source (`2a0e:97c0:8a0::/48`), the reply is BGP-routed to whichever agent is closest from the target's perspective. The `agent_id` field reveals which agent caught the reply:

```sql
SELECT
    agent_id AS catcher,
    count() AS replies,
    round(100.0 * replies / sum(replies) OVER (), 1) AS pct
FROM saimiris.replies
WHERE toString(probe_src_addr) LIKE '2a0e:97c0:8a0:%'
GROUP BY agent_id
ORDER BY replies DESC
FORMAT CSVWithNames
```

## Querying with curl

```bash
curl -s -X POST "https://nxthdr.dev/api/query/" \
  -u "read:read" \
  -H "Content-Type: text/plain" \
  -d "SELECT agent_id, count() FROM saimiris.replies
      WHERE time_received_ns >= now() - INTERVAL 1 HOUR
      GROUP BY agent_id FORMAT CSVWithNames"
```

## Common Experiment Patterns

### Ping campaign (latency measurement)

1. Prepare targets with TTL=64 and ICMPv6
2. Send from multiple agents
3. Compare RTT across vantage points

### Traceroute campaign (topology discovery)

1. Use prowl to generate probes with TTL 1-32
2. Send from one or more agents
3. Reconstruct paths by ordering replies by `probe_ttl`

### Anycast catchment mapping

1. Send probes using the anycast prefix (default source)
2. Query which `agent_id` received each reply
3. Group by destination ASN/country to map catchment boundaries

### Forward-path decomposition

1. Send unicast traceroutes from multiple agents to same targets
2. Send unicast pings (TTL=64) from all agents
3. Compare per-hop RTT to decompose the forward path into segments
