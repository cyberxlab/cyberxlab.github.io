---
title: "未授权 RCE 解剖：拆解 wp2shell WordPress 核心漏洞链"
date: 2026-07-19T06:10:00Z
draft: false
tags: ["Security", "WordPress", "REST API"]
categories: ["技术实践"]
author: "X 实验室"
description: "对 CVE-2026-63030（wp2shell）的工程化拆解：一个 REST 批量路由混淆与一个 WP_Query SQL 注入如何串成 WordPress 核心的未授权远程代码执行。"
summary: "本文从工程角度拆解 wp2shell 未授权 RCE：REST API 批量路由混淆（CVE-2026-63030）与 WP_Query 的 author__not_in SQL 注入（CVE-2026-60137）如何串成未授权代码执行，并给出暴露面探测、检测与在补丁之外进一步硬化站点的方法。"
toc: true
---

## 背景与动机

2026 年 7 月 17 日，WordPress 发布了一次紧急安全版本——7.0.2，并回滚补丁到 6.9.5 与 6.8.6，以关闭一个被公开命名为 **wp2shell** 的漏洞（[WordPress 安全发布公告](https://wordpress.org/news/2026/07/wordpress-7-0-2-release/)）。这个漏洞在 CMS 世界里属于极为罕见的一类：**未授权、远程、不依赖任何插件**的远程代码执行（RCE）链，能够直达一个 *默认安装* 的 WordPress 实例。漏洞报告方 Searchlight Cyber（研究员 Adam Kues）将它的攻击条件描述为 *无任何前置依赖* —— 不需要账户、不需要插件、不需要用户交互（[Searchlight Cyber 研究页面](https://slcyber.io/research-center/wp2shell-pre-authentication-rce-in-wordpress-core/)）。

对于绝大多数运维同学而言，处置动作其实非常直接——打补丁。但是对于负责开发框架、RPC API、或任何对外暴露 *批量端点* 的工程师而言，更有意思的问题是 **两个独立跟踪的漏洞是如何组合起来的**。wp2shell 是一处典型的 *漏洞链*：一个路由层的混淆（CVE-2026-63030）让攻击者能够把某个值灌进一段代码路径（CVE-2026-60137，即 `WP_Query::author__not_in` 的 SQL 注入），而这段代码路径原本被认为只有 *已授权的内部调用方* 才能到达。本文拆解这条漏洞链，演示如何从运行中的站点内部探测暴露面，并收集那些能预防这一 *类* 漏洞而不只是这一次事件本身的硬化方法。

## 前置条件

- 一个由你掌控的 WordPress 6.8.x – 7.0.1 站点（绝对不要在生产环境上验证——见步骤三的安全验证说明）。
- 对 `wp-content/` 有文件 Shell 访问权限即可，需要阅读 `class-wp-rest-server.php`、`class-wp-query.php`、`rest-api.php` 三个文件。
- 拥有 7.0.1 与 7.0.2 之间针对上述三个文件的差异快照。可借助官方 SVN/Git 镜像得到：
  ```bash
  git diff 7.0.1..7.0.2 -- \
    wp-includes/rest-api/class-wp-rest-server.php \
    wp-includes/class-wp-query.php \
    wp-includes/rest-api.php
  ```
- 可选但强烈推荐：一个持久化对象缓存后端（Redis 或 Memcached）。wp2shell 一个值得关注的运维事实是：**RCE 路径仅在未启用持久化对象缓存时可达**——对于自托管与小型共享主机环境，这是一个非常常见的默认配置（[Rapid7 ETR](https://www.rapid7.com/blog/post/etr-cve-2026-63030-wp2shell-a-critical-remote-code-execution-vulnerability-in-wordpress-core/)）。

## 步骤一：准备工作

这条链路在“信任边界失效”示意图里包含两段。在阅读任何源码之前请先固定词汇：

| 组件 | 失效点（CVE） | 单独存在时的作用 | 在链路中的作用 |
|---|---|---|---|
| `WP_REST_Server::serve_batch_request_v1()` | CVE-2026-63030 | 让一个子请求以错误的“匹配路由”上下文进行分发 | 把攻击者控制的参数塞进“本应被匹配路由校验”的处理器中 |
| `WP_Query::author__not_in` | CVE-2026-60137 | 由于整数列表参数未做归一化而促成 SQL 注入 | 为 RCE 链路提供注入原语 |

两份公告（[GHSA-ff9f-jf42-662q](https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-ff9f-jf42-662q)、[GHSA-fpp7-x2x2-2mjf](https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-fpp7-x2x2-2mjf)）均于 2026-07-17 公开。

写一个一次性侦察小工具——一个 must-use 插件，用于打印当前站点是否落在受影响版本区间内、以及是否启用了持久化对象缓存。这是非破坏性的，可以直接在 staging 上跑：

```php
<?php
// wp-content/mu-plugins/wp2shell_recon.php
add_action('admin_init', function () {
    if (!current_user_can('administrator')) return;
    global $wp_version;
    $v = $wp_version;                       // 例如 "6.9.4"
    $branches = [
        '6.8' => ['min' => '6.8.0', 'fixed' => '6.8.6', 'rce' => false],
        '6.9' => ['min' => '6.9.0', 'fixed' => '6.9.5', 'rce' => true],
        '7.0' => ['min' => '7.0.0', 'fixed' => '7.0.2', 'rce' => true],
    ];
    [$major] = explode('.', $v);
    $branch  = "$major." . explode('.', $v)[1];
    $b = $branches[$branch] ?? null;

    echo "<!-- wp2shell recon: version=$v branch=$branch";
    if (!$b) { echo "; 当前分支不在受影响区间内 -->"; return; }
    $exposed_sqli  = version_compare($v, $b['min'], '>=') && version_compare($v, $b['fixed'], '<');
    $exposed_rce   = $exposed_sqli && $b['rce'];
    $persistent    = wp_using_ext_object_cache();
    printf("; sqli_exposed=%s rce_exposed=%s persistent_obj_cache=%s -->",
        $exposed_sqli ? 'YES' : 'no',
        $exposed_rce ? 'YES' : 'no',
        $persistent ? 'YES' : 'no');
});
```

激活后访问 `/wp-admin/` 查看源码中的注释行。`RCE` 列只在 版本受影响 **且** 没有外部对象缓存时显示 `YES`——这正是 Cloudflare 标记为“现实暴露面”的那部分环境。

## 步骤二：核心实现

### 步骤 2a — 批量端点与路由混淆

WordPress 的 REST 批量端点 (`/wp-json/batch/v1`) 自 5.6（2020-11）开始提供，欢迎 API 客户端在一次往返中发送多个子请求——典型场景有插件安装流、区块编辑器预保存、OEmbed 扇出。分发器 `WP_REST_Server::serve_batch_request_v1()`（[源码](https://github.com/WordPress/wordpress-develop/blob/6.9/wp-includes/rest-api/class-wp-rest-server.php)）对每个子请求迭代调用 `match_request_to_route()` 然后逐个服务。

CVE-2026-63030 的失效点在于：**子请求与子请求之间，已匹配路由的状态被复用**。正确的批量分发应当在读取用户输入之前对每个子请求做 *已匹配路由的重置*；而旧实现如果某子请求未能完整命中路由，则会 **沿用前一次匹配路由的响应上下文**。攻击者可以构造一个批量体，其第一个子请求为一个正常公开路由（如 `/wp/v2/posts`），第二个子请求 *复用* 上一次的路由上下文，但走在一个事实上带有身份校验、却被混淆绕过的“错误处理路径”上——从而把不可信参数送到了本不该被未授权调用方触达的代码段中。

补丁版本在读取用户输入前无条件执行“每子请求 matched-route 重置”：

```php
// 7.0.2 修复后简化结构
foreach ($data['requests'] as $i => $single) {
    $matched_route = null;            // 每个子请求都重置
    $result = $this->serve_single_request($single, $matched_route);
    if (is_wp_error($result) && $result->get_error_code() === 'rest_no_route') {
        // 旧行为：下一个子请求会继承陈旧的 $matched_route
        continue;
    }
    $results['responses'][ $i ] = $result;
}
```

### 步骤 2b — author__not_in 进入链路

`WP_Query::author__not_in` 是“排除这些作者的文章”的查询参数，其值应当是整数或整数数组。受影响版本下，`class-wp-query.php` 在该值 *以单元素数组且元素为非整数字符串* 传入时并未做强制类型归一化。结合路由混淆弱点，被注入的字符串值未转义便进入了 MySQL 的 `AND post_author NOT IN (...)` 子句。

演示 *SQL 注入* 的最小请求体如下（注意这仅是 SQLi，公开来源里未发布完整的 RCE PoC）：

```http
POST /wp-json/batch/v1 HTTP/1.1
Host: vulnerable.example
Content-Type: application/json

{
  "validation": "require-all-properties-validate",
  "requests": [
    {"path": "/wp/v2/posts", "method": "GET"},
    {"path": "/wp/v2/posts",
     "method": "GET",
     "params": {"author__not_in": ["0) UNION (SELECT user_login, user_pass FROM wp_users-- "]}}
  ]
}
```

> **切勿在生产环境的 WordPress 上跑这段请求来“证明需要打补丁”。** 一个本地 staging 克隆或一个 Docker WordPress 就足够；安全公告本身已经确认了该漏洞（[WordPress 7.0.2 发布](https://wordpress.org/news/2026/07/wordpress-7-0-2-release/)）。Penligent 的拆解专门点了这个反模式（[Penligent](https://www.penligent.ai/hackinglabs/cve-2026-63030-wp2shell/)）——在生产环境动用武器化 PoC，事后都是清理取证与日志整改的活。

### 步骤 2c — 从 SQLi 提升到 RCE

最后一步是与系统实现相关的：从 `wp_users.user_pass` 取回的凭据使用 phpPortable 哈希；离线爆破后可登入管理员账户。一旦具备管理员权限，攻击者即可上传一个伪造插件，获得任意 PHP 执行权。由于 SQLi 已经通过批量路由混淆在未授权条件下可触发，因此整条链 **无需任何合法账户**。持久化对象缓存的“前置条件”在 Cloudflare 的分析中是关键豁免项——启用对象缓存后，WordPress 不会再对查询结果走 *每次请求都直打 MySQL* 的回退路径，恰好就是参数被注入时所走的路径。没有外部缓存后端的网站，按比例换算，每多跑一次 `WP_Query`，未来相同类别的攻击面就随之扩张。

## 步骤三：验证与调优

1. **确认版本传播。** 仪表盘 → 工具 → 站点健康 → 信息 → WordPress 版本。或通过 WP-CLI：
   ```bash
   wp core version
   wp core verify-checksums          # 文件树是否与发布版本一致
   ```
   WordPress 已对受影响分支启用强制自动更新——但不要“假设”已落地。逐台主机核实：文件系统权限、`DISALLOW_FILE_MODS`、不可变容器镜像、托管主机锁定都可能让自动更新默默失败。

2. **检查批量端点的访问日志**，时间维度回溯到 7 月 17 日及更早：
   ```bash
   grep -E '(/wp-json/batch/v1|rest_route=/batch/v1)' /var/log/nginx/access.log \
     | awk '{print $1}' | sort | uniq -c | sort -rn | head
   ```
   过去 30 天内向该端点发送异常大体量 `POST` 的 IP 都需要进一步取证——可以从 WAF 或 full-post 日志里拉原始负载。

3. **当无法升级镜像时**，临时在 nginx 层拦截批量路径：
   ```nginx
   # nginx vhost
   location ^~ /wp-json/batch/v1 {
       deny all;
       return 410;
   }
   # 同时拦截查询参数别名
   if ($args ~* "rest_route=/batch/v1") { return 410; }
   ```
   先在 staging 上验证对合法功能的影响：区块编辑器的预发布批量请求、部分插件安装流都会用到这个端点。请把拦截当成“过渡补丁桥”，而非长期解决办法。

4. **强制启用持久化对象缓存。** 即便补丁已经落地，从架构层面也能看出：缺持久化缓存的网站 *每个* 高成本查询都要走 MySQL，这恰好放大了任何未来查询层注入类漏洞的暴露面。部署 Redis（`wp redis enable`）或 Memcached（部署 `WP_CACHE` 与 drop-in）：
   ```bash
   wp cache flush && wp cache add kc_test 'hello' && wp cache get kc_test
   ```

## 最佳实践小结

- **先打补丁，再做校验和。** CVE 在 6.8.x 上只触发 SQLi，在 6.9.x / 7.0.x 上触发完整 RCE 链；修复版本分别为 6.8.6、6.9.5、7.0.2。务必核对实际跑的版本，而不是默认已升级。
- **把双 CVE 链作为单事件处理。** 只报告 CVE-2026-63030 的扫描器会低估 6.8 环境的暴露；只报告 CVE-2026-60137 的扫描器会低估 6.9 / 7.0 的风险。两个漏洞真实耦合——同一张工单处理。
- **无条件启用持久化对象缓存。** 它对于这一类 bug 来说是 *运维维度的硬化层*：没有它，每一次 `WP_Query` 都回到数据库，未来任何同类注入 bug 的攻击窗口都随之放大。
- **无法补丁时，在边缘拦截 `/wp-json/batch/v1`。** 把 WAF 规则视作 *补丁桥*，而非补丁替代品——Cloudflare 和 Aikido Zen 都明确声明其 WAF 规则不能替代升级（[Cloudflare WAF 公告](https://blog.cloudflare.com/wordpress-vulnerabilities/)）。
- **审计你自己开发的批量端点。** wp2shell 的教训可被泛化：任何“在子请求之间复用 matched-route 状态”的分发器都继承了这一类失败。对每个子请求无条件重置；绝不让未命中的子请求继承上一命中的响应。
- **不要对生产环境使用公开 PoC 武器。** 漏洞披露窗口变窄了，但并未消失；用 staging 或一次性 Docker WordPress 完成验证。

## 参考资料

- [WordPress 7.0.2 安全发布公告](https://wordpress.org/news/2026/07/wordpress-7-0-2-release/)
- [GHSA-ff9f-jf42-662q — CVE-2026-63030 批量路由混淆](https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-ff9f-jf42-662q)
- [GHSA-fpp7-x2x2-2mjf — CVE-2026-60137 WP_Query SQL 注入](https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-fpp7-x2x2-2mjf)
- [Searchlight Cyber — wp2shell Pre-Auth RCE in WordPress Core](https://slcyber.io/research-center/wp2shell-pre-authentication-rce-in-wordpress-core/)
- [Rapid7 ETR — CVE-2026-63030 wp2shell](https://www.rapid7.com/blog/post/etr-cve-2026-63030-wp2shell-a-critical-remote-code-execution-vulnerability-in-wordpress-core/)
- [SOCRadar — wp2shell CISO FAQ & Fix](https://socradar.io/blog/wp2shell-wordpress-rce-cve-2026-63030/wp2shell-wordpress-rce-cve-2026-63030)
- [Hadrian — wp2shell Pre-Auth RCE in REST Batch API](https://hadrian.io/blog/wp2shell-a-pre-authentication-rce-in-wordpress-cores-rest-batch-api)
- [Aikido — Unauthenticated RCE in WordPress core (wp2shell)](https://www.aikido.dev/blog/unauthenticated-rce-in-wordpress-wp2shell)
- [Penligent — CVE-2026-63030 wp2shell patch priority & safe validation](https://www.penligent.ai/hackinglabs/cve-2026-63030-wp2shell/)
- [Cloudflare Blog — WAF protections for WordPress vulnerabilities](https://blog.cloudflare.com/wordpress-vulnerabilities/)
- [The Hacker News — wp2shell WordPress Core Flaw](https://thehackernews.com/2026/07/new-wp2shell-wordpress-core-flaw-lets.html)
