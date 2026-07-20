---
title: "回归式 JPEG：从渐进式退化到 baseline 的故障复盘"
date: 2026-07-20T08:03:38Z
draft: false
tags: ["jpeg", "postmortem", "image-optimization", "web-performance"]
categories: ["技术实践"]
author: "X 实验室"
description: "一次实际工程上从渐进式 JPEG 退化到 baseline 的复盘：成因、可复现的检测方法，以及可以直接复制的 CI 防线脚本。"
summary: "围绕 Hugo/静态站管线里一次渐进式 JPEG 被静默重新编码成 baseline 的回归案例：用文件签名定位问题、追因到图像库，并给出可入 CI 的拦截脚本。"
toc: true
---

## 背景与动机

多年来，“导出渐进式 JPEG”几乎是所有 Web 性能文章都会列出的一条建议。原理不复杂：渐进式 JPEG（progressive JPEG）把图像数据拆成多次扫描存放，第一遍扫描就能给出整张图的概览模糊图，后续扫描逐步填补高频细节；用户在慢网络上拿到第一帧画面的时间显著提前，而完整加载下来视觉质量与 baseline 完全相同。基准指标 LCP 一般不动，但感知上的差异在 3G / EDGE / 拥塞 Wi-Fi 下是肉眼可见的。

近几年的现状发生了两点变化：

1. **AVIF / WebP 成为主交付格式**，JPEG 只剩下邮件、传统 CMS、以及兼容性兜底场景，于是“JPEG 是不是渐进式”逐渐掉出了团队的注意力。
2. **静态站点生成器、图像处理管线普遍对图片先解码后重编码。** 例如 Hugo 官方使用的就是 Go 的 `image/jpeg` 包，而 Go 标准库的 JPEG 编码器**只会输出 baseline**。这一点在 Hugo Discourse 上的[《Progressive JPEG defaults to baseline after image processing》](https://discourse.gohugo.io/t/progressive-jpeg-defaults-to-baseline-after-image-processing/21432)早有记录：设计师精心导出的 progressive JPEG 一旦经过 `Resources.Resize`，输出就静默变成 baseline。扩展名没变，画质没变，只有文件内部的扫描结构变了，连“哪里出问题”都看不见。

本文把这一类回归命名为**回归式 JPEG（regressive JPEG）**：上游是渐进式、下游悄悄变回 baseline，并且团队的指标体系完全察觉不到的故障。典型现场是：

- 设计师导出渐进式 hero 图；
- 构建流程 resize / crop / 重压；
- 部署到 CDN 上的产物是 baseline；
- LCP 测的是最终渲染事件，所以合成监控看不出异常；
- 真实低端 / 慢网用户开始抱怨“图加载慢”，但指标对不上号。

这篇文章的目的是把这类回归**看得见、能复现、能在 CI 卡住**。

## 前置条件

- POSIX shell + `file`（BSD 或 libmagic），用于查看文件签名。
- `libjpeg-turbo` 工具链中的 `jpegtran` 与 `djpeg`；不同发行版下包名可能为 `libjpeg-turbo-progs` / `libjpeg-tools` / `libjpeg-progs`。
- 可选：`mozjpeg`，用于无损地把 baseline 重排为渐进式。
- 任意一条“调用底层编码器”的图像流水线，可以是 Hugo / Eleventy / Next.js `next/image` / Rails `active_storage` 等。
- 不强求但是建议有：`xxd` / `hexdump -C` 一类十六进制查看工具，能帮助理解底层 marker。

## 步骤一：准备工作

最先需要厘清的一点：**progressive 与 baseline 不是“画质”区别，而是文件结构变体**。两种编码用同样的 DCT 有损压缩管线，差别只在最终量化系数在字节流里如何排布：

- Baseline：每个 8×8 MCU 的全部系数字字完整写出，MCU 按栅格顺序排列；
- Progressive：系数被拆成多次扫描，前几次扫描只放低频系数（DC + 少量 AC），后几次扫描再补上高频系数。

因为只是“排布不同”，所以两种格式之间可以在不重新做有损压缩的情况下**直接重排**：`jpegtran -progressive` 把 baseline 重排成 progressive，`jpegtran -baseline` 反之。两个操作都不会改动已反量化的系数——这也是为什么这是无损操作。

推论：任何**先解码再编码**的库（ImageMagick `convert`、Go 的 `image/jpeg.Encode`、libvips 的有损模式等）都有机会把渐进式“丢回” baseline，因为编码器走的是自己的 SOF0 输出路径。**只有调用 `jpegtran` 做无损重排的工具才会保住渐进结构**（mozjpeg 的 lossless 模式就是这样做的）。

```bash
# 准备一张能代表 hero 流程的 JPEG
curl -sL -o /tmp/source.jpeg https://www.gstatic.com/webp/gallery/1.jpg
ls -l /tmp/source.jpeg
```

然后建立两条对照：

```bash
# lossless 重排同源数据，得到 baseline / progressive 两个变体
jpegtran -progressive -outfile /tmp/prog.jpeg   /tmp/source.jpeg
jpegtran -baseline   -outfile /tmp/base.jpeg    /tmp/source.jpeg
wc -c /tmp/prog.jpeg /tmp/base.jpeg
```

常见结果是两份字节数在 ±2% 区间内。这是后面所有调优的成本带。

## 步骤二：核心实现

### 2.1 用文件签名判断 JPEG 扫描结构

JFIF 头里的 SOF marker 足以判定扫描结构：`SOF0`（字节序列 `FF C0`）= baseline，`SOF2`（`FF C2`）= progressive。一个最小分类器：

```bash
#!/usr/bin/env bash
# jpeg-scan-type.sh — 打印 progressive / baseline / unknown
set -euo pipefail

f="$1"
[ -r "$f" ] || { echo "unknown: $f not readable" >&2; exit 2; }

# SOF0 (0xFFC0) = baseline; SOF2 (0xFFC2) = progressive;
# SOF1 是扩展顺序式（罕用），同样按非渐进式处理。
marker=$(hexdump -C "$f" | grep -m1 -oE 'ff c[02]  ' | head -1)

case "$marker" in
  *"ff c0"*) echo "baseline"   ;;
  *"ff c2"*) echo "progressive" ;;
  *)         echo "unknown"     ;;
esac
```

直接验证：

```bash
$ bash jpeg-scan-type.sh /tmp/prog.jpeg
progressive
$ bash jpeg-scan-type.sh /tmp/base.jpeg
baseline
```

如果输出 `unknown`，多半是文件截断、或者扫描脚本里没枚举到 `SOF1 / SOF3 / SOF9 / SOF10` 这些少见 marker，生产环境几乎碰不到；万一遇到，干脆把文件喂给 `djpeg -fast -ppm > /dev/null`，凡解析失败就标记成“必须重编码”。

### 2.2 把生产流水线强制锁回渐进式

很多静态站点流水线无法直接告诉它“保持 JPEG 为渐进式”。最稳妥的做法是**在构建产物完成之后追加一次扫描结构重排**，把所有 baseline JPEG 重做成 progressive；本身就是渐进式的文件不动 mtime：

```bash
#!/usr/bin/env bash
# enforce-progressive-jpeg.sh — 把树里所有 baseline JPEG 无损重排为 progressive
set -euo pipefail

ROOT="${1:-public}"
count_baseline=0
count_skipped=0

while IFS= read -r -d '' f; do
  # 已经是 progressive 的就不动，避免无谓的 mtime 抖动
  if hexdump -C "$f" | grep -qE 'ff c2  '; then
    count_skipped=$((count_skipped+1))
    continue
  fi

  # 务必走临时文件，否则 jpegtran 半路崩溃会丢原图
  tmp="$(mktemp --suffix=.jpeg)"
  if jpegtran -copy none -progressive -outfile "$tmp" "$f"; then
    # gray JPEG 在某些扫描脚本下会让 jpegtran 输出 0 字节文件，必须校验
    if [ -s "$tmp" ]; then
      mv "$tmp" "$f"
      count_baseline=$((count_baseline+1))
    else
      echo "WARN: jpegtran 写出空文件，跳过 $f" >&2
      rm -f "$tmp"
    fi
  else
    echo "WARN: jpegtran 失败，跳过 $f" >&2
    rm -f "$tmp"
  fi
done < <(find "$ROOT" -type f \( -iname '*.jpg' -o -iname '*.jpeg' \) -print0)

echo "enforce-progressive-jpeg: 重排 $count_baseline 个文件，跳过 $count_skipped 个已是渐进式"
```

把它接到静态站生成之后：

```bash
hugo --minify --gc
./enforce-progressive-jpeg.sh public/
```

对 Hugo 用户来说，这是目前唯一真正可用的对策——前面引用的 Discourse 帖子已经开了 9 年，Go 标准 JPEG 编码器至今没修，团队级 patch 编码器不现实。

### 2.3 给 CI 加一道闸门，让回归进不去 main

2.2 节是补救措施；真正想要的是 CI 里的一道**闸门**——只要 pipeline 输出里出现哪怕一个 baseline JPEG，build 就直接失败：

```bash
#!/usr/bin/env bash
# ci-jpeg-progressive-gate.sh — 发现任何 baseline JPEG 就退出 1，CI 据此中断 deploy
set -euo pipefail

ROOT="${1:-public}"
violations=()

while IFS= read -r -d '' f; do
  if ! hexdump -C "$f" | grep -qE 'ff c2  '; then
    violations+=("$f")
  fi
done < <(find "$ROOT" -type f \( -iname '*.jpg' -o -iname '*.jpeg' \) -print0)

if [ "${#violations[@]}" -gt 0 ]; then
  echo "ERROR: 检测到 ${#violations[@]} 个 baseline JPEG：" >&2
  printf '  - %s\n' "${violations[@]}" >&2
  echo "部署前请先执行 enforce-progressive-jpeg.sh $ROOT/" >&2
  exit 1
fi

echo "ci-jpeg-progressive-gate: 所有 JPEG 均为渐进式 ✅"
```

接到 CI 工作流：

```yaml
# .github/workflows/build.yml （节选）
- name: Build
  run: hugo --minify --gc
- name: Enforce progressive JPEG
  run: ./enforce-progressive-jpeg.sh public/
- name: Gate — no baseline JPEG
  run: ./ci-jpeg-progressive-gate.sh public/
```

## 步骤三：验证与调优

### 3.1 字节数合理性

重排成 progressive **不一定**让文件更小，常见区间是 ±2%：

```bash
$ wc -c /tmp/base.jpeg /tmp/prog.jpeg
   28544 /tmp/base.jpeg
   28089 /tmp/prog.jpeg   # 大约小 1.6%
```

对非常小的图（优化后约 10KB 以下），多扫描头本身的体积开销可能反过来让 progressive 略大。所以 [ShortPixel 2026 那篇《Progressive JPEG vs Baseline JPEG》](https://shortpixel.com/blog/progressive-jpeg-vs-baseline-jpeg-does-it-still-matter-in-2026/)给的建议是：10KB 以下缩略图用 baseline 就行。给上面的 gate 增加一条小文件豁免：

```bash
# 小于 10240 字节的文件不参与 gate
size=$(stat -c%s "$f")
[ "$size" -lt 10240 ] && continue
```

### 3.2 解码 CPU 成本的现实

渐进式 JPEG 的解码 CPU 开销更高——Google 公布过的实测是平均 **3 倍**于 baseline（这条数据在 [ctrl.blog 那篇渐进式复盘](https://www.ctrl.blog/entry/jpeg-progressive-loading.html)和 [theimagecdn.com 的对比页](https://theimagecdn.com/docs/progressive-jpeg-vs-baseline-jpeg)都有引用）。桌面设备上这点开销通常是“量不到”的级别，但低端安卓机就能在 profiler 里看出来。两条缓解手段：

- **减少扫描次数**。`jpegtran -progressive` 默认扫描脚本大约 14 次；`mozjpeg` 自带一个收敛到 4 次扫描的脚本，扫描次数少 ⇒ 解码次数少 ⇒ CPU 更平。
- **采用“半渐进式”模式**（3–6 次扫描）来配置 hero。代价是中途渐进感略弱，收益是 CPU 抖动面变平。

### 3.3 不同格式各司其职

渐进式 JPEG 并不能取代 WebP / AVIF。HTTP Archive 2024 Media 章节仍显示移动端图像资源里 JPEG 占 32.4%（数据来源同上面 theimagecdn.com 一文），所以兜底链仍然重要；但推荐顺序不变：

1. 支持 AVIF 的浏览器优先 AVIF；
2. 不支持 AVIF 但支持 WebP 的，走 WebP；
3. 最后的 JPEG 兜底，**用 progressive 而不是 baseline**。

落到 HTML 上：

```html
<picture>
  <source type="image/avif"  srcset="/hero.avif">
  <source type="image/webp"  srcset="/hero.webp">
  <img src="/hero.jpeg" alt="Hero"           <!-- 这张必须是渐进式 -->
       loading="lazy" decoding="async">
</picture>
```

落到 CDN 上，请求按 `Accept` 协商格式，一张 origin 图按 UA 自动派发不同字节。

### 3.4 一个已知“假阴性”永远不会跳过

`jpegtran` 在某些扫描脚本下，对**已经是灰度的 JPEG**会输出 **0 字节文件**。所以 2.2 里的脚本必须用 `[ -s "$tmp" ]`（文件非空判断）兜住——少了它就会被空文件覆盖原图造成数据丢失。

## 最佳实践小结

- **把 progressive / baseline 当文件结构看，而不是画质开关。** 重编码器只关心自己的 SOF0 路径，不关心你导入的是不是 progressive。
- **任何先解码后编码的流水线（Hugo / Eleventy / Rails 等）都额外加一遍 `jpegtran -progressive`**。无损、幂等、零代价。
- **部署前用 CI 闸门卡住。** 一份几十行 shell 走 `public/**/*.jp(e)?g`，凡是没有 `0xFFC2` 标记的文件就令 build 失败。出问题在生产之前几分钟就能发现，而不是在生产里。
- **小于 ~10KB 的缩略图豁免闸门。** 这个体积档 progressive 的开销常常盖过收益，baseline 反而更划算。
- **progressive JPEG 配合 AVIF / WebP 一起用。** 现代 `<picture>` 顺序是 AVIF → WebP → progressive JPEG 兜底，渐进式只在“最末端受众”那儿起决定作用，而这恰恰就是 EDGE / 拥塞 Wi-Fi 那一波人。
- **生产环境优先用 `mozjpeg` 而不是 stock `jpegtran`。** 自带更紧的扫描脚本，且相对 `libjpeg-turbo` 的默认还能再省下可观字节，依然无损。

## 参考资料

- ShortPixel — [Progressive JPEG vs Baseline JPEG: Does It Still Matter in 2026?](https://shortpixel.com/blog/progressive-jpeg-vs-baseline-jpeg-does-it-still-matter-in-2026/)
- ctrl.blog — [Progressive JPEGs make a meaningful impact on perceived performance](https://www.ctrl.blog/entry/jpeg-progressive-loading.html)
- The Image CDN — [Progressive JPEG vs Baseline JPEG - Which Should You Use?](https://theimagecdn.com/docs/progressive-jpeg-vs-baseline-jpeg)
- Hugo Discourse — [Progressive JPEG defaults to baseline after image processing](https://discourse.gohugo.io/t/progressive-jpeg-defaults-to-baseline-after-image-processing/21432)
- web.dev — [Learn Images: JPEG](https://web.dev/learn/images/jpeg)
