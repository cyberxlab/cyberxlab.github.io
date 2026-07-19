---
title: "Blog"
layout: wide-articles
---

<section class="not-prose hx:relative hx:overflow-hidden hx:rounded-2xl hx:mb-12 hx:w-full" style="height: 280px; display: block;">
  <img src="/images/articles-hero.webp" alt="Cyber·X·Lab Blog" style="position:absolute; inset:0; width:100%; height:100%; object-fit:cover; object-position:center; display:block;" loading="lazy" decoding="async" />
  <!-- 暗色渐变：从左往右加深，网络节点背景均衡分布 -->
  <div style="position:absolute; inset:0; width:100%; height:100%; background:linear-gradient(to right, rgba(0,0,0,0.85), rgba(0,0,0,0.45), transparent);"></div>
</section>

<!-- RSS 订阅入口（纯 HTML，避免 shortcode 嵌套 bug） -->
<a href="index.xml"
   class="hx:inline-flex hx:items-center hx:gap-2 hx:mb-6 hx:px-3 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-gray-700 hx:text-sm hx:text-gray-700 hx:dark:text-gray-300 hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800 hx:transition-colors">
  <svg height="14" width="14" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" aria-hidden="true">
    <path stroke-linecap="round" stroke-linejoin="round" d="M6 5c7.18 0 13 5.82 13 13M6 11a7 7 0 017 7m-6 0a1 1 0 11-2 0 1 1 0 012 0z" />
  </svg>
  <span>RSS Feed</span>
</a>

{{< cards >}}
  {{< card
        link="/blog/cyberxlab-launch/"
        title="Cyber·X·Lab Launch Announcement"
        subtitle="Cyber·X·Lab is officially live! Learn about the platform, our mission, and what's coming next."
  >}}
{{< /cards >}}
