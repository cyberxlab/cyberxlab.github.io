---
title: "HTMLProofer 自动化检查 的实践指南"
date: 2026-07-18T00:00:00+08:00
draft: false
tags: ["HTMLProofer,自动化检查"]
categories: ["技术实践"]
author: "X 实验室"
description: "一步步演示如何在 X 实验室官网上落地 HTMLProofer 自动化检查 的最佳实践。"
toc: true
---

## 背景与目标

在现代静态站点构建流程中，确保生成的 HTML 没有损坏的链接、缺失的 alt 属性或其他常见错误是保证质量的关键步骤。HTMLProofer 是一个成熟的 Ruby 工具，能够对生成的 HTML 文件进行全面检查。本指南将演示如何在 X 实验室的 Hugo 站点中集成 HTMLProofer，并通过 GitHub Actions 实现自动化检查。

## 前置条件

- 已安装 Ruby（推荐使用 rbenv 或 rvm 管理）
- 已安装 Bundler（`gem install bundler`）
- 项目中已有 `Gemfile`（若没有请先创建）
- 已经推送到 GitHub 的 Hugo 站点仓库（本例为 `cyberxlab.github.io`）
- 已启用 GitHub Actions

## 步骤一：准备工作

1. 在项目根目录下创建或编辑 `Gemfile`，加入 htmlproofer 依赖：

   ```ruby
   source 'https://rubygems.org'
   gem 'html-proofer'
   ```

2. 安装依赖：

   ```bash
   bundle install
   ```

3. 新建一个检查脚本 `scripts/htmlproofer.sh`（确保可执行）：

   ```bash
   #!/usr/bin/env bash
   set -e
   # 假设 Hugo 输出目录为 ./public
   bundle exec htmlproofer ./public --allow-href-hash --assume-extension
   ```

4. 将该脚本加入到仓库并提交。

## 步骤二：核心实现

在 GitHub Actions 工作流中添加 HTMLProofer 检查步骤。编辑 `.github/workflows/site.yml`（或新建）：

```yaml
name: Site CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Install dependencies
        run: bundle install

      - name: Build Hugo site
        run: |
          hugo --minify

      - name: Run HTMLProofer
        run: ./scripts/htmlproofer.sh

      - name: Upload artifact (if needed)
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: htmlproofer-report
          path: ./tmp/htmlproofer
```

> 参考资料：  
> - HTMLProofer 官方仓库：https://github.com/gjtorikian/html-proofer  
> - GitHub Marketplace Action 示例：https://github.com/marketplace/actions/htmlproofer  
> - 自动化测试工具最佳实践参考：https://blog.csdn.net/2401_84563987/article/details/138455684  

## 步骤三：验证与调优

1. **本地验证**：提交后在本地运行 `bundle exec htmlproofer ./public`，确认无误。
2. **查看 Action 日志**：在 GitHub Actions 页面查看 `Run HTMLProofer` 步骤的输出，根据报告修复问题。
3. **调整忽略规则**：若有特定需要忽略的链接或文件，可在脚本中加入相应选项，例如 `--url-ignore "/#foo/,/example.com/"` 或 `--file-ignore ./public/404.html`。
4. **性能优化**：对于大站点，可考虑使用 `--only-4xx` 仅检查客户端错误，或增加并行线程（`--parallel`）。

## 最佳实践小结

- 在 CI 中尽早介入 HTML 检查，能够在 PR 阶段即时发现问题。
- 将 HTMLProofer 集成到构建流程中，确保每次提交都经过质量门禁。
- 根据项目实际情况调整检查范围，避免误报（如外部链接临时不可用）。
- 定期更新 `html-proofer` gem，以获得最新的检查规则和安全修复。
- 将检查报告作为构件保存，便于事后分析。

## 参考资料

- https://github.com/gjtorikian/html-proofer
- https://github.com/marketplace/actions/htmlproofer
- https://blog.csdn.net/2401_84563987/article/details/138455684
- https://zhi.oscs1024.com/18022.html