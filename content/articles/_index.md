---
title: Articles
description: "Articles and guides from the Cyber·X·Lab community."
toc: false
layout: wide-articles
---

<section class="not-prose hx:relative hx:overflow-hidden hx:rounded-2xl hx:mb-6" style="min-height: 280px;">
  <img src="/images/articles-hero.webp" alt="Cyber·X·Lab Articles" class="hx:absolute hx:inset-0 hx:w-full hx:h-full hx:object-cover" loading="lazy" decoding="async" />
  <!-- 暗色渐变：从左往右加深，网络节点背景均衡分布，白字标题落在左侧暗区 -->
  <div class="hx:absolute hx:inset-0 hx:bg-gradient-to-r hx:from-black/85 hx:via-black/45 hx:to-transparent"></div>
  <div class="hx:relative hx:flex hx:flex-col hx:justify-end hx:items-start hx:px-8 hx:py-10 hx:text-left" style="min-height: 280px;">
    <h1 class="hx:text-3xl hx:font-bold hx:text-white hx:mb-2 hx:tracking-tight">Articles</h1>
    <p class="hx:text-base hx:max-w-2xl" style="color: rgba(255, 255, 255, 0.85) !important;">Articles and guides from the Cyber·X·Lab community — deep dives, tutorials, and hands-on guides for technology enthusiasts.</p>
  </div>
</section>

<!-- RSS 订阅入口：banner 下方，左对齐（纯 HTML 避免短代码嵌套 bug） -->
<a href="index.xml"
   class="hx:inline-flex hx:items-center hx:gap-2 hx:mb-6 hx:px-3 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-gray-700 hx:text-sm hx:text-gray-700 hx:dark:text-gray-300 hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800 hx:transition-colors">
  <svg height="14" width="14" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" aria-hidden="true">
    <path stroke-linecap="round" stroke-linejoin="round" d="M6 5c7.18 0 13 5.82 13 13M6 11a7 7 0 017 7m-6 0a1 1 0 11-2 0 1 1 0 012 0z" />
  </svg>
  <span>RSS Feed</span>
</a>

{{< cards >}}
  {{< card
        link="/articles/bitset-enum-flags-pattern/"
        title="Type-Safe Bit Flags for C++ Enum Classes"
        subtitle="From scoped enum foundation to a generic BitFlags template, this guide shows how to keep type safety while making C++ bit-flag enums ergonomic and easy to audit."
  >}}
  {{< card
        link="/articles/linux-scheduler-metrics-evaluation/"
        title="Evaluating Linux Scheduler Latency: Practical Metrics and Measurement Techniques"
        subtitle="Learn how to diagnose CPU starvation and evaluate scheduler latency on Linux. This guide covers eBPF-based tracing with runqlat and low-overhead stats from the proc filesystem."
  >}}
  {{< card
        link="/articles/openssl-hollowbyte-dos-anatomy/"
        title="HollowByte Anatomy: How 11 Bytes Can Strand a Server Without a Single CVE"
        subtitle="An engineering decomposition of HollowByte, the OpenSSL DoS disclosed by Okta Red Team in July 2026: how a 4-byte TLS handshake header triggers an unvalidated pre-allocation up to 131 KB per connection, how glibc fragmentation turns attacks into permanent RSS bloat, how the fix moved to lazy incremental buffer growth, and how site-reliability engineers detect and harden against the class."
  >}}
  {{< card
        link="/articles/cache-directory-tagging-spec/"
        title="The 43-Byte Convention That Protects Backups: Cache Directory Tagging in Practice"
        subtitle="How a 43-byte magic header — Signature: 8a477f597d28d172789f06886806bc55 — lets apps mark cache trees so tar, Borg and restic skip them by default; the spec's exact semantics, adopter quirks, and the security tradeoff that makes blind exclusion dangerous."
  >}}
  {{< card
        link="/articles/navigation-api-baseline/"
        title="The Navigation API: Modern Client-Side Routing for SPAs"
        subtitle="This article provides a production-ready guide to the Navigation API, covering intercept semantics, the two-phase commit model, manual scroll control, state management, and integration with View Transitions. Includes feature detection, graceful degradation strategies, and a verification checklist for cross-browser SPA routing."
  >}}
  {{< card
        link="/articles/powershell-ssh-remoting-2026/"
        title="PowerShell SSH Remoting in 2026: Production Guide"
        subtitle="This article walks through the end-to-end setup of PowerShell 7+ SSH remoting in 2026, from OpenSSH installation and subsystem configuration to key-based auth, cipher hardening, and operational troubleshooting for Linux-to-Windows and Windows-to-Linux scenarios."
  >}}
  {{< card
        link="/articles/wordpress-preauth-rce-anatomy/"
        title="Anatomy of a Pre-Auth RCE: Decomposing the wp2shell WordPress Core Chain"
        subtitle="An engineering decomposition of the wp2shell pre-auth RCE in WordPress core: how a REST API batch-route confusion (CVE-2026-63030) and a WP_Query author__not_in SQL injection (CVE-2026-60137) chained into unauthenticated code execution, how to detect exposure, and how to harden sites beyond patching."
  >}}
  {{< card
        link="/articles/rate-limit-handling/"
        title="Rate Limit Handling: Building HTTP Clients That Survive 429s"
        subtitle="This article walks through a layered strategy for handling HTTP 429 rate-limit responses in production API clients, covering exponential backoff with jitter, the Retry-After header, client-side token buckets, and an internal proxy service for multi-callers scenarios."
  >}}
  {{< card
        link="/articles/data-driven-shortcodes/"
        title="Data-Driven Markdown Shortcodes"
        subtitle="Explore how to use data files (JSON/YAML) in Hugo to drive reusable shortcodes, decoupling content from presentation."
  >}}
  {{< card
        link="/articles/hermes-quickstart/"
        title="Hermes Agent Quickstart"
        subtitle="Get from zero to a working Hermes Agent setup — install, choose a provider, verify a working chat, and know what to do when something breaks."
  >}}
{{< /cards >}}
