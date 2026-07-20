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
        link="/zh-cn/articles/regressive-jpeg-engineering-postmortem/"
        title="回归式 JPEG：从渐进式退化到 baseline 的故障复盘"
        subtitle="围绕 Hugo/静态站管线里一次渐进式 JPEG 被静默重新编码成 baseline 的回归案例：用文件签名定位问题、追因到图像库，并给出可入 CI 的拦截脚本。"
  >}}
  {{< card
        link="/zh-cn/articles/gitroot-self-hosted-pr-merge-tool/"
        title="深入剖析 GitRoot：基于代码驱动 PR 合并的自托管 Git 极简 Forge"
        subtitle="探索 GitRoot 如何通过将所有元数据（Issue、Graft 分支、看板）以纯文本文件形式直接存储在 Git 仓库中，实现无数据库、极其高弹性的自托管 Git Forge。同时详细分析其 Grafter 插件如何在不离开 IDE 的情况下完成 PR 评审和自动合并。"
  >}}
  {{< card
        link="/zh-cn/articles/bitset-enum-flags-pattern/"
        title="为 C++ 枚举类设计类型安全位标志"
        subtitle="从 scoped enum 基础到通用 BitFlags 模板，本文展示如何兼顾类型安全与可审计的位标志操作，避免跨枚举误用与运算符优先级陷阱。"
  >}}
  {{< card
        link="/zh-cn/articles/linux-scheduler-metrics-evaluation/"
        title="评估 Linux 调度器延迟：实用指标与测量技术"
        subtitle="了解如何诊断 CPU 饥饿并评估 Linux 调度器延迟。本指南涵盖了基于 eBPF 的 runqlat 跟踪，以及通过 proc 文件系统获取的超低开销指标采样。"
  >}}
  {{< card
        link="/zh-cn/articles/openssl-hollowbyte-dos-anatomy/"
        title="HollowByte 解剖：11 字节如何让一台服务器在无一枚 CVE 的情况下搁浅"
        subtitle="拆解 Okta Red Team 于 2026 年 7 月公开的 OpenSSL HollowByte DoS：4 字节的 TLS 握手头如何让未校验的预分配一路涨到 131 KB、glibc 碎片化如何把已释放的块变成永久堆积、OpenSSL 的修复如何转向惰性增量分配，以及站点可靠性工程师如何检测并硬化这一漏洞类别。"
  >}}
  {{< card
        link="/zh-cn/articles/cache-directory-tagging-spec/"
        title="保护备份的 43 字节约定：缓存目录标记实战"
        subtitle="一个 43 字节的魔数头部 —— Signature: 8a477f597d28d172789f06886806bc55 —— 让应用把自己的缓存目录标记出来，tar、Borg、restic 默认跳过；本文拆解规范的确切语义、各采纳方差异、以及让盲目排除变得危险的那个安全权衡。"
  >}}
  {{< card
        link="/zh-cn/articles/navigation-api-baseline/"
        title="Navigation API：面向现代 SPA 的客户端路由原语"
        subtitle="本文提供 Navigation API 的生产级实践指南，涵盖 intercept 语义、两阶段提交、手动滚动控制、状态管理，以及与 View Transitions 的集成。包含特性探测、优雅降级策略，以及跨浏览器 SPA 路由的验证清单。"
  >}}
  {{< card
        link="/zh-cn/articles/powershell-ssh-remoting-2026/"
        title="PowerShell SSH 远程处理 2026：生产实践指南"
        subtitle="本文系统讲解 2026 年 PowerShell SSH 远程处理的端到端实践：从 OpenSSH 安装与子系统配置，到密钥认证、加密算法加固，以及 Linux→Windows 与 Windows→Linux 场景的运维排查。"
  >}}
  {{< card
        link="/zh-cn/articles/wordpress-preauth-rce-anatomy/"
        title="未授权 RCE 解剖：拆解 wp2shell WordPress 核心漏洞链"
        subtitle="本文从工程角度拆解 wp2shell 未授权 RCE：REST API 批量路由混淆（CVE-2026-63030）与 WP_Query 的 author__not_in SQL 注入（CVE-2026-60137）如何串成未授权代码执行，并给出暴露面探测、检测与在补丁之外进一步硬化站点的方法。"
  >}}
  {{< card
        link="/zh-cn/articles/rate-limit-handling/"
        title="速率限制处理：构建能在 429 中存活的 HTTP 客户端"
        subtitle="本文系统讲解生产环境 API 客户端处理 HTTP 429 速率限制的分层策略，覆盖带抖动的指数退避、Retry-After 头部、客户端令牌桶，以及用于多调用方场景的内部代理服务。"
  >}}
  {{< card
        link="/zh-cn/articles/data-driven-shortcodes/"
        title="数据驱动 Markdown 短代码"
        subtitle="本文介绍在 Hugo 中使用数据文件驱动短代码的思路与实现步骤，包括数据结构设计、短代码模板编写、样式处理以及使用示例，帮助读者实现内容与展示的解耦，提升可维护性。"
  >}}
  {{< card
        link="/zh-cn/articles/hermes-quickstart/"
        title="Hermes Agent 快速上手"
        subtitle="从零开始搭建一个完整的 Hermes Agent 工作环境 —— 安装、选择模型供应商、验证可用的对话，并掌握常见问题的排查方法。"
  >}}
{{< /cards >}}
