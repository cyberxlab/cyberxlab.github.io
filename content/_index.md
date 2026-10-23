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

## Research

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="AI & Embodied Intelligence"
    subtitle="Large language models, automated code/research agents, and AI applied across disciplines."
    icon="chip"
    image="/images/cards/card-research-ai.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(99,102,241,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Biotech & Computational Biology"
    subtitle="Synthetic biology, AI-driven drug design, and digital biology research."
    icon="beaker"
    image="/images/cards/card-research-biotech.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(34,197,94,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Space Tech & Orbital Computing"
    subtitle="Satellite communications, space data processing, and frontier aerospace engineering."
    icon="sparkles"
    image="/images/cards/card-research-space.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(249,115,22,0.15), hsla(0,0%,100%,0));"
  >}}
{{< /hextra/feature-grid >}}

## Build

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="Open-Source Tools & Infrastructure"
    subtitle="Building geek-friendly open-source projects and toolchains to lower the barrier to deployment."
    icon="code"
    image="/images/cards/card-build-opensource.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(168,85,247,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Science Experiments & Hardcore Tutorials"
    subtitle="Structured hands-on guides and experiment documentation — turning theory into practice."
    icon="beaker"
    image="/images/cards/card-build-experiments.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(236,72,153,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Frontier Digital Tech Practice"
    subtitle="Cross-disciplinary tech fusion: computational physics, simulation, and hardcore engineering."
    icon="terminal"
    image="/images/cards/card-build-frontier.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(14,165,233,0.15), hsla(0,0%,100%,0));"
  >}}
{{< /hextra/feature-grid >}}

## Share

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="Deep Articles & Research Reports"
    subtitle="Accessible yet rigorous deep dives into frontier science — breaking down knowledge walls."
    icon="document-text"
    image="/images/cards/card-share-articles.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(234,179,8,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Cross-Disciplinary Community"
    subtitle="Connecting enthusiasts and experts across AI, biology, aerospace, and software engineering."
    icon="users"
    image="/images/cards/card-share-community.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(34,197,94,0.15), hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Geek Innovation Incubation"
    subtitle="Encouraging questions, discussion, and open-source collaboration — incubating frontier experiments."
    icon="light-bulb"
    image="/images/cards/card-share-incubation.jpg"
        class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    style="background: radial-gradient(ellipse at 50% 80%, rgba(249,115,22,0.15), hsla(0,0%,100%,0));"
  >}}
{{< /hextra/feature-grid >}}
