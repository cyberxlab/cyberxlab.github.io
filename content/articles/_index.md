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
        link="/articles/hermes-quickstart/"
        title="Hermes Agent Quickstart"
        subtitle="Get from zero to a working Hermes Agent setup — install, choose a provider, verify a working chat, and know what to do when something breaks."
  >}}
  {{< card
        link="/articles/data-driven-shortcodes/"
        title="Data-Driven Markdown Shortcodes"
        subtitle="Explore how to use data files (JSON/YAML) in Hugo to drive reusable shortcodes, decoupling content from presentation."
  >}}
  {{< card
        link="/articles/rate-limit-handling/"
        title="Rate Limit Handling: Building HTTP Clients That Survive 429s"
        subtitle="This article walks through a layered strategy for handling HTTP 429 rate-limit responses in production API clients, covering exponential backoff with jitter, the Retry-After header, client-side token buckets, and an internal proxy service for multi-callers scenarios."
  >}}
{{< /cards >}}
