---
title: Labs
layout: hextra-home
---

<style>
.xlabs-hero-box {
  position: relative;
  width: 100%;
  overflow: hidden;
  border-radius: 1rem;
  aspect-ratio: 16 / 9;
  min-height: 200px;
  max-height: 460px;
  display: block;
}
@media (min-width: 640px) {
  .xlabs-hero-box {
    aspect-ratio: 21 / 9;
  }
}
.xlabs-hero-img {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  object-position: center;
  display: block;
}
</style>
<section class="not-prose xlabs-hero-box hx:mb-10 hx:mt-4 hx:shadow-2xl" style="aspect-ratio: 16/9; min-height: 200px; max-height: 460px; display: block;">
  <img src="/images/home-hero.webp" alt="X·Labs" class="xlabs-hero-img" style="width: 100%; height: 100%; object-fit: cover; object-position: center; display: block;" loading="eager" decoding="async" />
  <!-- 全局电影级环境微光暗角与边缘渐隐 -->
  <div class="hx:absolute hx:inset-0 hx:pointer-events-none" style="position: absolute; inset: 0; background: radial-gradient(ellipse at center, transparent 45%, rgba(0,0,0,0.45) 100%), linear-gradient(to bottom, rgba(0,0,0,0.1), transparent 25%, transparent 75%, rgba(0,0,0,0.35) 100%);"></div>
</section>

## 核心前沿研究

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="AI 与具身智能"
    subtitle="大模型、自动化代码/科学研究 Agent，以及人工智能在多学科中的应用。"
    icon="chip"
    image="/images/cards/card-research-ai.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(99,102,241,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="生物科技与计算生物"
    subtitle="合成生物学、AI 药物设计与生物数字化研究。"
    icon="beaker"
    image="/images/cards/card-research-biotech.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(34,197,94,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="太空科技与轨道计算"
    subtitle="卫星通信、太空数据处理与前沿航天工程探索。"
    icon="sparkles"
    image="/images/cards/card-research-space.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(249,115,22,0.15), hsla(0,0%,100%,0));"
  >}}
{{< /hextra/feature-grid >}}

## 工程实践与产出

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="开源工具与基础设施"
    subtitle="打造极客友好的开源项目与工具链，降低技术落地成本。"
    icon="code"
    image="/images/cards/card-build-opensource.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(168,85,247,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="科学实验与硬核教程"
    subtitle="提供结构化的动手指南与实验文档，让理论落地为实践。"
    icon="beaker"
    image="/images/cards/card-build-experiments.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(236,72,153,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="前沿数字技术实践"
    subtitle="跨学科技术融合，探索计算物理、模拟仿真等硬核工程。"
    icon="terminal"
    image="/images/cards/card-build-frontier.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(14,165,233,0.15), hsla(0,0%,100%,0));"
  >}}
{{< /hextra/feature-grid >}}

## 开源生态与分享

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="深度文章与研究报告"
    subtitle="通俗且严谨的前沿科技解读，打破知识传播墙。"
    icon="document-text"
    image="/images/cards/card-share-articles.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(234,179,8,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="跨学科交流社区"
    subtitle="连接 AI、生物、航天及软件领域的爱好者与专家。"
    icon="users"
    image="/images/cards/card-share-community.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(34,197,94,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="极客创新孵化"
    subtitle="鼓励提问、探讨与开源项目协作，孵化有趣的前沿尝试。"
    icon="light-bulb"
    image="/images/cards/card-share-incubation.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(249,115,22,0.15), hsla(0,0%,100%,0));"
  >}}
{{< /hextra/feature-grid >}}
