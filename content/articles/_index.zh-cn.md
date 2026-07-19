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

<!-- RSS 订阅入口：banner 下方，左对齐（纯 HTML 避免短代码嵌套 bug） -->
<a href="index.xml"
   class="hx:inline-flex hx:items-center hx:gap-2 hx:mb-6 hx:px-3 hx:py-1.5 hx:rounded-full hx:border hx:border-gray-300 hx:dark:border-gray-700 hx:text-sm hx:text-gray-700 hx:dark:text-gray-300 hx:hover:bg-gray-100 hx:dark:hover:bg-neutral-800 hx:transition-colors">
  <svg height="14" width="14" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="2" stroke="currentColor" aria-hidden="true">
    <path stroke-linecap="round" stroke-linejoin="round" d="M6 5c7.18 0 13 5.82 13 13M6 11a7 7 0 017 7m-6 0a1 1 0 11-2 0 1 1 0 012 0z" />
  </svg>
  <span>RSS 订阅</span>
</a>

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
  {{< card
        link="/zh-cn/articles/rate-limit-handling/"
        title="速率限制处理：构建能在 429 中存活的 HTTP 客户端"
        subtitle="本文系统讲解生产环境 API 客户端处理 HTTP 429 速率限制的分层策略，覆盖带抖动的指数退避、Retry-After 头部、客户端令牌桶，以及用于多调用方场景的内部代理服务。"
  >}}
  {{< card
        link="/zh-cn/articles/wordpress-preauth-rce-anatomy/"
        title="未授权 RCE 解剖：拆解 wp2shell WordPress 核心漏洞链"
        subtitle="本文从工程角度拆解 wp2shell 未授权 RCE：REST API 批量路由混淆（CVE-2026-63030）与 WP_Query 的 author__not_in SQL 注入（CVE-2026-60137）如何串成未授权代码执行，并给出暴露面探测、检测与在补丁之外进一步硬化站点的方法。"
  >}}
  {{< card
        link="/zh-cn/articles/powershell-ssh-remoting-2026/"
        title="PowerShell SSH 远程处理 2026：生产实践指南"
        subtitle="本文系统讲解 2026 年 PowerShell SSH 远程处理的端到端实践：从 OpenSSH 安装与子系统配置，到密钥认证、加密算法加固，以及 Linux→Windows 与 Windows→Linux 场景的运维排查。"
  >}}
  {{< card
        link="/zh-cn/articles/navigation-api-baseline/"
        title="Navigation API：面向现代 SPA 的客户端路由原语"
        subtitle="本文提供 Navigation API 的生产级实践指南，涵盖 intercept 语义、两阶段提交、手动滚动控制、状态管理，以及与 View Transitions 的集成。包含特性探测、优雅降级策略，以及跨浏览器 SPA 路由的验证清单。"
  >}}
{{< /cards >}}
