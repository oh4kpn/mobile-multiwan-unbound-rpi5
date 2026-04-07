# Multi-Mobile-WAN DNS Setup with Unbound

This repository documents a robust Unbound configuration designed for multi-mobile-WAN environments, where network traffic can be extremely unreliable—think **“like peeing in the wind.”**

## Why This Setup?

In networks with multiple mobile WANs (4G, 5G, Starlink):
* **Instability is the baseline:** Packets can get lost, delayed, or arrive out-of-order.
* **Path Selection Issues:** Standard DNS setups often fail; even TCP-based client queries may not reliably reach the internet if the default path is congested.
* **Race for Speed:** "Fastest-first" logic is required to choose the best available path per individual query.
* **Resilience:** Aggressive prefetching and stale-cache serving improve perceived performance and stability during backhaul flapping.

## Key Features

-   **Multi-Homing:** Multiple outgoing interfaces for all available WANs (e.g., DNA, Elisa, Telia) using MACVLAN.
-   **Pure Recursion:** Root-hints based recursion to avoid relying on (often slow or filtered) ISP forwarders.
-   **Fastest-First Selection:** Tuned `unknown-server-time-limit` to bypass slow or hanging WAN paths instantly.
-   **Aggressive Recovery:** Short `infra-host-ttl` to react quickly to changing mobile network conditions.
-   **Extreme Resilience:** 72-hour `serve-expired` TTL allows the network to stay functional even during total backhaul failure.
-   **RPi5 Optimized:** Threaded configuration optimized for high-performance ARM-based gateways (Sisu).

---

## Technical Architecture

### 1. Network Layer (Linux MACVLAN)
Instead of a single IP, the gateway uses virtual MACVLAN interfaces. This allows a single physical NIC to present multiple MAC addresses to the upstream router (e.g., UniFi Gateway), enabling Policy-Based Routing (PBR) per virtual interface.

**Example Interface Mapping:**
- `eth0`: `10.120.1.3` (Management/Default)
- `mac-dna`: `10.120.1.40` -> PBR to WAN1 (DNA)
- `mac-elisa`: `10.120.1.41` -> PBR to WAN2 (Elisa)
- `mac-telia`: `10.120.1.42` -> PBR to WAN3 (Telia)

### 2. Unbound Configuration

Key stability and speed parameters from `/etc/unbound/unbound.conf`:

```yaml
server:
    # Bind to all virtual WAN interfaces
    # outgoing-interface: 10.120.1.3
    outgoing-interface: 10.120.1.40
    outgoing-interface: 10.120.1.41
    outgoing-interface: 10.120.1.42

    # WAN Competition & Speed Logic
    infra-host-ttl: 60             # Forget slow paths quickly (re-test latency every min)
    unknown-server-time-limit: 375 # If a WAN lags >375ms, try another path immediately
    target-fetch-policy: 3 2 1 0 0 # Aggressively fetch records in parallel

    # Reliability Boosters
    prefetch: yes                  # Renew popular records before they expire
    prefetch-key: yes              # Keep DNSSEC keys fresh
    serve-expired: yes             # Use the cache even if the upstream is down
    serve-expired-ttl: 259200      # 72 hours of "memory" for outages
    serve-expired-reply-ttl: 30    # Answer immediately with stale data while updating

    # Performance (RPi5 / 4-core optimized)
    num-threads: 2
    msg-cache-slabs: 4
    rrset-cache-slabs: 4
    infra-cache-slabs: 4
    key-cache-slabs: 4
    so-rcvbuf: 4m
    so-sndbuf: 4m

    # Security & Recursion
    root-hints: "/etc/unbound/root.hints"
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: yes
    edns-buffer-size: 1232
