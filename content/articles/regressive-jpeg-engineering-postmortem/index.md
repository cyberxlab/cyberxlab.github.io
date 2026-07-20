---
title: "Regressive JPEG: A Postmortem on Baseline-Fallback Regression"
date: 2026-07-20T08:03:38Z
draft: false
tags: ["jpeg", "postmortem", "image-optimization", "web-performance"]
categories: ["Technical Practices"]
author: "Cyber·X·Lab"
description: "A postmortem-style walkthrough of a real progressive-to-baseline JPEG regression: how it slips past CI, how to detect it, and how to lock it down."
summary: "A focused postmortem on a progressive-JPEG-to-baseline regression in a Hugo/static pipeline: detection with file signatures, root-cause in image-processing libraries, and a CI guard you can copy."
toc: true
---

## Background and Motivation

For years the standard advice in every web-performance guide was simple: **export progressive JPEG**. On slow networks a progressive JPEG paints the entire frame in a blurry first scan, then refines it across subsequent scans, so users see composition almost immediately instead of the dreaded top-to-bottom " Venetian blind" reveal of a baseline image. Perceptual time-to-first-frame drops, bounce falls, the experience is meaningfully better.

Then two things happened.

1. **WebP and AVIF became the default delivery format** for everything except email and legacy CMS pipelines, so JPEG handling quietly fell off the radar of many teams.
2. **Static-site generators and image-processing pipelines silently re-encode** any image that passes through them. Hugo, for instance, uses Go's `image/jpeg` package — and as the Hugo Discourse thread on ["Progressive JPEG defaults to baseline after image processing"](https://discourse.gohugo.io/t/progressive-jpeg-defaults-to-baseline-after-image-processing/21432) documents, the standard Go encoder has never emitted progressive JPEG. Take a perfectly good progressive JPEG, run it through `Resources.Resize`, and the output silently becomes baseline. The file extension is the same. The visual quality is the same. Only the internal scan structure — and the loading behavior — is gone.

This is the class of bug this article calls a **regressive JPEG**: a JPEG that was progressive upstream and regressed to baseline downstream without anyone noticing. The canonical postmortem pattern is:

- A designer exports hero images as progressive JPEG.
- A build pipeline resizes / crops / recompresses them.
- The deployed artifact is baseline.
- Performance dashboards do not move because LCP measures the **final** rendered bytes, not intermediate scans. The regression is invisible to synthetic monitoring.
- Real users on EDGE / 3G / congested Wi-Fi start complaining about "images loading slow" with no metric to correlate against.

The purpose of this article is to make this class of regression **visible, reproducible, and CI-trappable**.

## Prerequisites

- A POSIX shell and `file` (BSD or libmagic) for signature inspection.
- `libjpeg-turbo` tooling, specifically `jpegtran` and `djpeg` — available as `libjpeg-turbo-progs` / `libjpeg-tools` / `libjpeg-progs` depending on your distro.
- Optional: `mozjpeg` if you want lossless re-progressive-ization.
- An existing image pipeline you can run end-to-end (Hugo, Eleventy, Next.js `next/image`, a Rails `active_storage` variant, anything that calls an encoder under the hood).
- familiarity with a hex-dump tool (`xxd` / `hexdump -C`) — not required but clarifying.

## Step 1: Preparation

The first thing to internalize is that **progressive vs baseline is not a quality setting — it is a file-structure variant**. Both encodings use the identical DCT-based lossy pipeline. The difference is purely how the resulting quantized coefficients are laid out in the byte stream:

- Baseline: coefficients for every 8×8 MCU are written fully, in raster order, MCUs top-to-bottom.
- Progressive: coefficients are split into multiple scans. Early scans carry low-frequency DCT coefficients (DC + a few ACs) at full frame resolution; later scans fill in higher-frequency coefficients.

Because of this, you can convert between the two without any re-loss by simply **rescanning** the existing_entropy-coded data. `jpegtran -progressive` does exactly that — it repacks the bitstream without touching the dequantized coefficients. Similarly, `jpegtran -baseline` does the inverse. Neither operation changes the visual content; only the scan ordering does.

Practical implication: any tool that round-trips through a decode-then-encode (ImageMagick `convert`, Go's `image/jpeg.Encode`, libvips in lossy mode) **will** re-encode and has an opportunity to write baseline even if the input was progressive. Tools that call `jpegtran` under the hood (mozjpeg does this for lossless operations) preserve the existing scan structure unless told otherwise.

For this article, prepare a single test image:

```bash
# Make sure you have jpegtran available
command -v jpegtran || sudo apt-get install -y libjpeg-turbo-progs

# Prepare a reasonably sized source JPEG (something a hero pipeline would produce)
curl -sL -o /tmp/source.jpeg https://www.gstatic.com/webp/gallery/1.jpg
ls -l /tmp/source.jpeg
```

Establish a baseline expectation:

```bash
# Convert the same image two ways for comparison
jpegtran -progressive -outfile /tmp/prog.jpeg   /tmp/source.jpeg
jpegtran -baseline   -outfile /tmp/base.jpeg    /tmp/source.jpeg
wc -c /tmp/prog.jpeg /tmp/base.jpeg
```

Typical result: the progressive variant is within ±2% of baseline (often smaller, occasionally a hair larger for very small images). This is the cost band you operate in.

## Step 2: Core Implementation

### 2.1 Detect the scan structure of a JPEG on disk

The JFIF structure (`SOF0` = baseline, `SOF2` = progressive) is enough for a cheap classifier that has no false positives on well-formed JPEGs:

```bash
#!/usr/bin/env bash
# jpeg-scan-type.sh — prints "progressive" / "baseline" / "unknown" for a JPEG file
# Uses the SOF marker to identify scan structure. Pure POSIX shell + hexdump.
set -euo pipefail

f="$1"
[ -r "$f" ] || { echo "unknown: $f not readable" >&2; exit 2; }

# SOF0 (0xFFC0) = baseline; SOF2 (0xFFC2) = progressive; SOF1 is extended sequential
# (rarely used); treat both sequential variants as non-progressive.
marker=$(hexdump -C "$f" | grep -m1 -oE 'ff c[02]  ' | head -1)

case "$marker" in
  *"ff c0"*) echo "baseline"   ;;
  *"ff c2"*) echo "progressive" ;;
  *)         echo "unknown"     ;;
esac
```

Run it:

```bash
$ bash jpeg-scan-type.sh /tmp/prog.jpeg
progressive
$ bash jpeg-scan-type.sh /tmp/base.jpeg
baseline
```

If the tool answers `unknown` the file is either truncated or uses a SOF marker we did not enumerate (`SOF1`, `SOF3` lossless, arithmetic coding `SOF9`, `SOF10`...). For web production that almost never happens; if it does, pipe the file through `djpeg -fast -ppm > /dev/null` and treat any failure as "cannot decode, must re-encode."

### 2.2 Lock the production pipeline to progressive

Most static-site pipelines cannot be told "keep my JPEG progressive" directly. The robust fix is to add a **post-build pass** that walks every generated `.jpg` / `.jpeg` and re-progressive-izes any file that came out baseline:

```bash
#!/usr/bin/env bash
# enforce-progressive-jpeg.sh — re-progressive any baseline JPEG in a tree.
# Idempotent: -progressive on an already-progressive file is a no-op.
set -euo pipefail

ROOT="${1:-public}"
count_baseline=0
count_skipped=0

while IFS= read -r -d '' f; do
  # Skip already-progressive to avoid touching mtime unnecessarily
  if hexdump -C "$f" | grep -qE 'ff c2  '; then
    count_skipped=$((count_skipped+1))
    continue
  fi

  # Rewrite in-place via a temp file so we never lose data if jpegtran explodes
  tmp="$(mktemp --suffix=.jpeg)"
  if jpegtran -copy none -progressive -outfile "$tmp" "$f"; then
    # Guard against the well-known jpegtran 0-byte output on grayscale JPEGs
    if [ -s "$tmp" ]; then
      mv "$tmp" "$f"
      count_baseline=$((count_baseline+1))
    else
      echo "WARN: jpegtran produced empty output for $f — left unchanged" >&2
      rm -f "$tmp"
    fi
  else
    echo "WARN: jpegtran failed on $f — left unchanged" >&2
    rm -f "$tmp"
  fi
done < <(find "$ROOT" -type f \( -iname '*.jpg' -o -iname '*.jpeg' \) -print0)

echo "enforce-progressive-jpeg: $count_baseline re-encoded, $count_skipped already-progressive"
```

Hook this into your build script immediately after the static generator writes to `public/` (or whatever your output directory is):

```bash
hugo --minify --gc
./enforce-progressive-jpeg.sh public/
```

For a Hugo pipeline specifically, this is the only mechanism that works against `image/jpeg.Encode`. The Discourse thread quoted above is nine years old and the Go stdlib JPEG encoder still does not emit progressive output — patching the encoder is out of scope for most teams.

### 2.3 Add a CI guard that fails BEFORE the broken build ships

The script in 2.2 is a remediation; what you want in CI is a **gate** that fails the build the moment a JPEG leaves the test pipeline as baseline:

```bash
#!/usr/bin/env bash
# ci-jpeg-progressive-gate.sh — fail if any JPEG under public/ is baseline.
# Exits non-zero on the first baseline file found; CI then aborts the deploy.
set -euo pipefail

ROOT="${1:-public}"
violations=()

while IFS= read -r -d '' f; do
  if ! hexdump -C "$f" | grep -qE 'ff c2  '; then
    violations+=("$f")
  fi
done < <(find "$ROOT" -type f \( -iname '*.jpg' -o -iname '*.jpeg' \) -print0)

if [ "${#violations[@]}" -gt 0 ]; then
  echo "ERROR: ${#violations[@]} JPEG(s) are baseline (not progressive):" >&2
  printf '  - %s\n' "${violations[@]}" >&2
  echo "Run enforce-progressive-jpeg.sh $ROOT/ before deploying." >&2
  exit 1
fi

echo "ci-jpeg-progressive-gate: all JPEGs are progressive ✅"
```

Add it to your CI workflow:

```yaml
# .github/workflows/build.yml (excerpt)
- name: Build
  run: hugo --minify --gc
- name: Enforce progressive JPEG
  run: ./enforce-progressive-jpeg.sh public/
- name: Gate — no baseline JPEG
  run: ./ci-jpeg-progressive-gate.sh public/
```

## Step 3: Verification and Tuning

### 3.1 Byte-size sanity

Re-progressive-izing does not always save bytes. Empirically the change is in the ±2% band:

```bash
$ wc -c /tmp/base.jpeg /tmp/prog.jpeg
   28544 /tmp/base.jpeg
   28089 /tmp/prog.jpeg   # ~1.6% smaller
```

For very small images (under ~10KB after optimization) the structural overhead of multiple scan headers can make the progressive variant slightly **larger**. The usual recommendation from the [ShortPixel "Progressive JPEG vs Baseline JPEG: Does It Still Matter in 2026?" post](https://shortpixel.com/blog/progressive-jpeg-vs-baseline-jpeg-does-it-still-matter-in-2026/) is to use baseline for thumbnails below ~10KB. A reasonable tunable for the gate above:

```bash
# Skip gates for files below 10240 bytes; small files don't benefit from progressive scans.
size=$(stat -c%s "$f")
[ "$size" -lt 10240 ] && continue
```

### 3.2 Decode-cost reality check

Progressive JPEG costs more CPU to decode — Google's own measurement, quoted both by [ctrl.blog](https://www.ctrl.blog/entry/jpeg-progressive-loading.html) and `theimagecdn.com`'s [comparison guide](https://theimagecdn.com/docs/progressive-jpeg-vs-baseline-jpeg), puts progressive decode at roughly **3× baseline** on average. On high-end desktops this is unmeasurable. On low-end mobiles it can be visible in profile. Two mitigations:

- **Reduce the number of scans.** `jpegtran -progressive` uses a default scan script of about 14 scans. `mozjpeg` ships a tighter 4-scan script. Fewer scans ⇒ fewer decode passes ⇒ less CPU.
- **Use the "semi-progressive" pattern** (3–6 scans) for hero imagery on mobile-heavy sites. Trade-off: a tiny bit less incremental sharpening during load, but a flat CPU profile.

### 3.3 Choose the right format for the right job

Progressive JPEG does not obsolete WebP/AVIF. The 2024 HTTP Archive Media chapter (cited by the image CDN article above) still has JPEG at 32.4% of mobile image resources — so JPEG fallbacks matter — but the headline recommendation is unchanged:

1. Serve **AVIF** to browsers that accept it.
2. Serve **WebP** to browsers without AVIF support.
3. Deliver **progressive JPEG** as the final fallback, not baseline.

Concretely in HTML:

```html
<picture>
  <source type="image/avif"  srcset="/hero.avif">
  <source type="image/webp"  srcset="/hero.webp">
  <img src="/hero.jpeg" alt="Hero"           <!-- this one should be progressive -->
       loading="lazy" decoding="async">
</picture>
```

And in CDN configuration, ask for format negotiation so the same origin URL serves the right bytes per `Accept` header.

### 3.4 Filter the well-known false negative

`jpegtran` will write a **0-byte file** when given an already-grayscale JPEG and certain scan scripts. The script in 2.2 already guards against this with `[ -s "$tmp" ]` (test for non-empty file). Do not skip that test; you will lose data.

## Best Practices Summary

- **Treat progressive-vs-baseline as a file structure, not a quality setting.** Your build pipeline's re-encoder cares only about its `SOF0`-emitting code path; it does not care that you imported a progressive source.
- **Add an explicit post-build `jpegtran -progressive` pass** for any pipeline (Hugo, Eleventy, Rails, etc.) that decodes-then-encodes images. Idempotent and zero-loss.
- **CI gate before deploy.** A 30-line shell script that walks `public/**/*.jp(e)?g` and refuses any file without an `0xFFC2` marker catches the regression in minutes, not in production.
- **Skip the gate for thumbnails under ~10KB.** Progressive overhead can exceed its savings below that size; baseline is fine for thumbnails.
- **Pair progressive JPEG with AVIF/WebP first.** Modern `<picture>` ordering: AVIF → WebP → progressive JPEG fallback. The progressiveness of the JPEG matters for the long-tail fallback audience, especially on EDGE / congested Wi-Fi.
- **Prefer `mozjpeg` over stock `jpegtran` for production.** It defaults to a tight spectrum-progressive scan script and provides measurable further byte savings beyond `libjpeg-turbo`'s defaults without re-loss.

## References

- ShortPixel — [Progressive JPEG vs Baseline JPEG: Does It Still Matter in 2026?](https://shortpixel.com/blog/progressive-jpeg-vs-baseline-jpeg-does-it-still-matter-in-2026/)
- ctrl.blog — [Progressive JPEGs make a meaningful impact on perceived performance](https://www.ctrl.blog/entry/jpeg-progressive-loading.html)
- The Image CDN — [Progressive JPEG vs Baseline JPEG - Which Should You Use?](https://theimagecdn.com/docs/progressive-jpeg-vs-baseline-jpeg)
- Hugo Discourse — [Progressive JPEG defaults to baseline after image processing](https://discourse.gohugo.io/t/progressive-jpeg-defaults-to-baseline-after-image-processing/21432)
- web.dev — [Learn Images: JPEG](https://web.dev/learn/images/jpeg)
