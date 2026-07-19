---
title: 文章
description: "来自 Cyber·X·Lab 社区的文章与指南。"
toc: false
layout: wide-articles
---

<section class="not-prose hx:relative hx:overflow-hidden hx:rounded-2xl hx:mb-6" style="min-height: 280px;">
  <img src="/images/articles-hero.webp" alt="Cyber·X·Lab 文章" class="hx:absolute hx:inset-0 hx:w-full hx:h-full hx:object-cover" loading="lazy" decoding="async" />
  <!-- 暗色渐变：从左往右加深，白字标题落在左侧暗区 -->
  <div class="hx:absolute hx:inset-0 hx:bg-gradient-to-r hx:from-black/85 hx:via-black/45 hx:to-transparent"></div>
  <div class="hx:relative hx:flex hx:flex-col hx:justify-end hx:items-start hx:px-8 hx:py-10 hx:text-left" style="min-height: 280px;">
    <h1 class="hx:text-3xl hx:font-bold hx:text-white hx:mb-2 hx:tracking-tight">文章</h1>
    <p class="hx:text-base hx:max-w-2xl" style="color: rgba(255, 255, 255, 0.85) !important;">来自 Cyber·X·Lab 社区的文章与指南 —— 深度解析、教程和实践指南，为技术爱好者打造。</p>
  </div>
</section>

<!-- RSS 订阅入口：banner 下方，左对齐 -->
<div class="hx:mb-6">
  {{< hextra/hero-badge link="index.xml" >}}
    <span>RSS 订阅</span>
    {{< icon name="rss" attributes="height=14" >}}
  {{< /hextra/hero-badge >}}
</div>

{{< cards >}}
  {{< card
        link="/articles/hermes-quickstart/"
        title="Hermes Agent 快速上手"
        subtitle="从零开始搭建一个完整的 Hermes Agent 工作环境 —— 安装、选择模型供应商、验证可用的对话，并掌握常见问题的排查方法。"
  >}}
  {{< card
        link="/zh-cn/articles/data-driven-shortcodes/"
        title="数据驱动 Markdown 短代码"
        subtitle="本文介绍在 Hugo 中使用数据文件驱动短代码的思路与实现步骤，包括数据结构设计、短代码模板编写、样式处理以及使用示例，帮助读者实现内容与展示的解耦，提升可维护性。"
  >}}
{{< /cards >}}
