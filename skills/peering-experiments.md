# Skill: Peering Experiments with nxthdr

This skill teaches how to run BGP peering experiments using the nxthdr CLI and PeerLab, and analyze routing data from ClickHouse.

## Prerequisites

- The [nxthdr CLI](https://github.com/nxthdr/cli) installed
- Authenticated: `nxthdr login`
- Docker and Docker Compose installed (for PeerLab)

## Platform Overview

**PeerLab** is nxthdr's peering platform. It lets users lease public IPv6 prefixes, announce them from real Internet Exchange Points (IXPs), and observe BGP propagation — all through AS215011.

Key characteristics:
- **IPv6 only**
- **Private ASN** assigned per user (range 65000–65999)
- **/48 prefix leases** with 1–24 hour duration
- **RPKI ROA** automatically managed
- **IXP presence**: FogIXP (Frankfurt), FranceIX (Paris), LOCIX (Frankfurt), NL-ix (Amsterdam)
- **BGP data** publicly available in ClickHouse

## Step 1: Check Your ASN

```bash
nxthdr peering asn
```

An ASN is automatically assigned on first prefix request. If already assigned, this shows your ASN.

## Step 2: Lease a Prefix

```bash
nxthdr peering prefix request <HOURS>
```

- Duration: 1–24 hours
- Allocates a /48 from the pool (currently `2a06:de00:5b::/48` and `2a06:de00:5c::/48`)
- RPKI ROA is created automatically

### List active leases

```bash
nxthdr peering prefix list
```

Shows prefix, expiration time, and RPKI status.

### Revoke a lease early

```bash
nxthdr peering prefix revoke <PREFIX>
```

Example: `nxthdr peering prefix revoke "2a06:de00:5b::/48"`

## Step 3: Manage RPKI

RPKI ROA is enabled by default when leasing. You can toggle it:

```bash
nxthdr peering prefix rpki enable <PREFIX>
nxthdr peering prefix rpki disable <PREFIX>
```

**Note:** RPKI must be enabled before revoking a prefix (platform enforces this for routing stability).

## Step 4: Set Up PeerLab (Docker Environment)

PeerLab is a Docker stack that establishes BGP sessions with nxthdr's IXP routers via a Headscale (Tailscale) overlay network.

### Generate environment configuration

```bash
nxthdr peering peerlab env
```

This outputs `USER_ASN` and `USER_PREFIXES` for your `.env` file.

### Clone and configure

```bash
git clone https://github.com/nxthdr/peerlab.git
cd peerlab
nxthdr peering peerlab env > .env
```

### Start the stack

```bash
make setup        # Start Tailscale container
make auth         # Authenticate with Headscale (opens browser)
make up           # Start full stack (bootstrap + BIRD + Caddy)
```

### Verify connectivity

```bash
make bird-status       # BIRD daemon status
make bird-protocols    # BGP session details
make bird-routes-count # Number of received IPv6 routes
make bird-exports      # Routes being advertised to peers
make bird-prefixes     # Your configured static prefixes
```

## Step 5: Routing Configuration

PeerLab uses BIRD as the BGP daemon. Configuration is auto-generated from your `.env`, but you can customize `workspace/bird.conf` for advanced use cases.

### Receive-only mode (no announcements)

Leave `USER_PREFIXES` empty in `.env`:

```
USER_ASN=65000
USER_PREFIXES=
```

### Advertise prefixes

Set your leased prefixes:

```
USER_ASN=65000
USER_PREFIXES=2a06:de00:5b::/48
```

Multiple prefixes (comma-separated):

```
USER_PREFIXES=2a06:de00:5b::/48,2a06:de00:5c::/48
```

After changing `.env`, restart:

```bash
make down && make up
```

### AS path prepending

Edit `workspace/bird.conf` and modify the export filter:

```bird
filter ExportToIXP {
    if net ~ [ 2a06:de00:5b::/48 ] then {
        bgp_path.prepend(USER_ASN);
        bgp_path.prepend(USER_ASN);
        accept;
    }
    reject;
}
```

### Selective export per IXP

Override the export filter on specific BGP protocol blocks in `workspace/bird.conf` to announce different prefixes at different IXPs.

### BGP communities

You can tag routes with standard or large BGP communities in your export filter:

```bird
bgp_community.add((65000, 100));
bgp_large_community.add((215011, 1, 1));
```

Or filter imports by community:

```bird
filter ImportFromIXP {
    if (65000, 100) ~ bgp_community then accept;
    reject;
}
```

## Step 6: Useful BIRD Commands

```bash
make bird CMD='show protocols all'           # Detailed protocol info
make bird CMD='show route for 2001:4860::/32' # Lookup specific prefix
make bird CMD='show route where bgp_path ~ [= * 13335 * =]' # Routes via Cloudflare
make bird CMD='show route export <protocol>' # What you export to a peer
make bird CMD='show route count'             # Total route count
```

## Step 7: Analyze BGP Data in ClickHouse

All BGP data from AS215011 is publicly available:

```
Endpoint: https://nxthdr.dev/api/query/
Username: read
Password: read
```

### Table: `bmp.updates`

| Column | Type | Description |
|--------|------|-------------|
| `time_received_ns` | UInt64 | Timestamp in nanoseconds |
| `peer_ip` | IPv6 | BGP peer IP (IXP router) |
| `peer_asn` | UInt32 | Peer ASN |
| `prefix` | String | Announced/withdrawn prefix |
| `next_hop` | IPv6 | BGP next hop |
| `as_path` | Array(UInt32) | AS path |
| `communities` | Array(String) | BGP communities |
| `large_communities` | Array(String) | Large BGP communities |
| `is_announcement` | UInt8 | 1 = announcement, 0 = withdrawal |

### Table: `saimiris.replies`

Probing data can be combined with peering data for joint experiments. See the probing-measurements skill for the full schema.

### Example Queries

#### Verify your prefix is being announced

```sql
SELECT
    time_received_ns,
    peer_ip,
    as_path,
    is_announcement
FROM bmp.updates
WHERE prefix = '2a06:de00:5b::/48'
ORDER BY time_received_ns DESC
LIMIT 20
FORMAT CSVWithNames
```

#### Track announcement and withdrawal history

```sql
SELECT
    fromUnixTimestamp64Nano(time_received_ns) AS ts,
    peer_ip,
    if(is_announcement, 'A', 'W') AS type,
    as_path
FROM bmp.updates
WHERE prefix = '2a06:de00:5b::/48'
ORDER BY time_received_ns DESC
LIMIT 50
FORMAT CSVWithNames
```

#### Analyze AS path propagation

```sql
SELECT
    as_path,
    count() AS times_seen,
    max(fromUnixTimestamp64Nano(time_received_ns)) AS last_seen
FROM bmp.updates
WHERE prefix = '2a06:de00:5b::/48'
  AND is_announcement = 1
GROUP BY as_path
ORDER BY times_seen DESC
FORMAT CSVWithNames
```

#### Measure AS path prepending effectiveness

```sql
SELECT
    peer_ip,
    as_path,
    length(as_path) AS path_length
FROM bmp.updates
WHERE prefix = '2a06:de00:5b::/48'
  AND is_announcement = 1
ORDER BY time_received_ns DESC
LIMIT 20
FORMAT CSVWithNames
```

#### Compare announcements across IXPs

```sql
SELECT
    peer_ip,
    count() AS updates,
    countIf(is_announcement = 1) AS announcements,
    countIf(is_announcement = 0) AS withdrawals
FROM bmp.updates
WHERE prefix = '2a06:de00:5b::/48'
GROUP BY peer_ip
ORDER BY updates DESC
FORMAT CSVWithNames
```

#### Monitor routing dynamics (update frequency)

```sql
SELECT
    toStartOfMinute(fromUnixTimestamp64Nano(time_received_ns)) AS minute,
    count() AS updates,
    countIf(is_announcement = 1) AS announcements,
    countIf(is_announcement = 0) AS withdrawals
FROM bmp.updates
WHERE prefix = '2a06:de00:5b::/48'
  AND time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute
FORMAT CSVWithNames
```

#### Monitor community propagation

```sql
SELECT
    large_communities,
    count() AS seen
FROM bmp.updates
WHERE prefix = '2a06:de00:5b::/48'
  AND is_announcement = 1
GROUP BY large_communities
ORDER BY seen DESC
FORMAT CSVWithNames
```

#### Combine with sFlow traffic data

```sql
SELECT
    toStartOfMinute(fromUnixTimestamp64Nano(time_received_ns)) AS minute,
    sum(bytes) AS total_bytes,
    count() AS samples
FROM sflow.flows
WHERE toString(dst_addr) LIKE '2a06:de00:5b:%'
  AND time_received_ns >= now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute
FORMAT CSVWithNames
```

## Querying with curl

```bash
curl -s -X POST "https://nxthdr.dev/api/query/" \
  -u "read:read" \
  -H "Content-Type: text/plain" \
  -d "SELECT as_path, count() AS c FROM bmp.updates
      WHERE prefix = '2a06:de00:5b::/48' AND is_announcement = 1
      GROUP BY as_path ORDER BY c DESC LIMIT 10
      FORMAT CSVWithNames"
```

## Common Experiment Patterns

### BGP convergence study

1. Lease a prefix and announce it from PeerLab
2. Query `bmp.updates` to observe how fast the announcement propagates across peers
3. Withdraw the prefix (`make down`) and measure withdrawal convergence
4. Compare convergence times across IXPs

### AS path prepending impact

1. Announce a prefix without prepending, wait for convergence
2. Query `bmp.updates` to record baseline AS paths
3. Add prepending in BIRD config, restart
4. Compare new AS paths — verify prepending shifts traffic

### Anycast catchment with peering

1. Announce the same prefix from PeerLab (via IXPs)
2. Send probes from Saimiris agents using the anycast source prefix
3. Correlate which agents catch replies (`saimiris.replies.agent_id`) with BGP path data
4. Map how BGP routing decisions affect probe return paths

### RPKI validation testing

1. Lease a prefix (ROA auto-created)
2. Verify announcement accepted by RPKI-validating peers
3. Disable RPKI: `nxthdr peering prefix rpki disable <PREFIX>`
4. Observe if any peers reject the now-invalid route
5. Re-enable RPKI: `nxthdr peering prefix rpki enable <PREFIX>`

### Joint peering and probing study

1. Announce prefix from PeerLab
2. Use Saimiris to traceroute toward your prefix from multiple vantage points
3. Combine `bmp.updates` (control plane) with `saimiris.replies` (data plane) to compare BGP paths vs actual forwarding paths

## Infrastructure Reference

- **AS215011**: nxthdr's autonomous system
- **IXP servers**: ixpfra01 (Frankfurt), ixpams01/ixpams02 (Amsterdam), ixpcdg01/ixpcdg02 (Paris)
- **Core prefix**: `2a06:de00:50::/44`
- **PeerLab prefix pool**: `2a06:de00:5b::/48`, `2a06:de00:5c::/48`
- **RPKI validator**: Cloudflare (`rtr.rpki.cloudflare.com:8282`), 900s refresh
- **VPN overlay**: Headscale at `headscale.nxthdr.dev`
- **Policy enforcement**: Per-user ASN + prefix validation on IXP routers (auto-generated BIRD config)

## Limitations

- IPv6 only (no IPv4)
- Limited prefix pool (currently 2 /48 prefixes)
- IXP coverage limited to Europe (Frankfurt, Paris, Amsterdam)
- Lease duration capped at 24 hours
- One ASN per user
- Max concurrent leases is per-user (default: 1)
