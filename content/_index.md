---
title: Cyber·X·Lab
layout: hextra-home
---

<section class="not-prose hx:relative hx:overflow-hidden hx:rounded-2xl hx:mb-12 hx:mt-6 hx:w-full" style="height: 480px; display: block;">
  <img src="/images/home-hero.webp" alt="Cyber·X·Lab" style="position:absolute; inset:0; width:100%; height:100%; object-fit:cover; object-position:center; display:block;" loading="lazy" decoding="async" />
  <!-- 暗色渐变：WALL-E 与主图在左侧，所以从右向左加深，右侧留空以融入 dark 主题 -->
  <div style="position:absolute; inset:0; width:100%; height:100%; background:linear-gradient(to left, rgba(0,0,0,0.8), rgba(0,0,0,0.4), transparent);"></div>
</section>

{{< hextra/feature-grid >}}
  {{< hextra/feature-card
    title="Fast and Full-featured"
    subtitle="Simple and easy to use, yet powerful and feature-rich."
    class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    image="images/hextra-doc.webp"
    imageClass="hx:top-[40%] hx:left-[24px] hx:w-[180%] hx:sm:w-[110%] hx:dark:opacity-80"
    style="background: radial-gradient(ellipse at 50% 80%,rgba(194,97,254,0.15),hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Markdown is All You Need"
    subtitle="Compose with just Markdown. Enrich with Shortcode components."
    class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-lg:min-h-[340px]"
    image="images/hextra-markdown.webp"
    imageClass="hx:top-[40%] hx:left-[36px] hx:w-[180%] hx:sm:w-[110%] hx:dark:opacity-80"
    style="background: radial-gradient(ellipse at 50% 80%,rgba(142,53,74,0.15),hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Full Text Search"
    subtitle="Built-in full text search with FlexSearch, no extra setup required."
    class="hx:aspect-auto hx:md:aspect-[1.1/1] hx:max-md:min-h-[340px]"
    image="images/hextra-search.webp"
    imageClass="hx:top-[40%] hx:left-[36px] hx:w-[110%] hx:sm:w-[110%] hx:dark:opacity-80"
    style="background: radial-gradient(ellipse at 50% 80%,rgba(221,210,59,0.15),hsla(0,0%,100%,0));"
  >}}
  {{< hextra/feature-card
    title="Lightweight as a Feather"
    subtitle="No dependency or Node.js is needed to use Hextra. Powered by Hugo, one of *the fastest* static site generators, building your site in just seconds with a single binary."
  >}}
  {{< hextra/feature-card
    title="Responsive with Dark Mode Included"
    subtitle="Looks great on different screen sizes. Built-in dark mode support, with auto-switching based on user's system preference."
  >}}
  {{< hextra/feature-card
    title="Build and Host for Free"
    subtitle="Build with GitHub Actions, and host for free on GitHub Pages. Alternatively it can be hosted on any static hosting service."
  >}}
  {{< hextra/feature-card
    title="Multi-Language Made Easy"
    subtitle="Create multi-language pages by just adding locales suffix to the Markdown file. Adding i18n support to your site is intuitive."
  >}}
  {{< hextra/feature-card
    title="And Much More..."
    icon="sparkles"
    subtitle="Syntax highlighting / Table of contents / SEO / RSS / LaTeX / Mermaid / Customizable / and more..."
  >}}
{{< /hextra/feature-grid >}}
