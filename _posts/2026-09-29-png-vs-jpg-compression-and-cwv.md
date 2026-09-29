---
title: "PNG vs JPG Compression, and How Images Affect Core Web Vitals (CWV)"
date: 2026-09-29 00:00:00 +0900
categories: [Web, Frontend]
tags: [image-optimization, png, jpg, core-web-vitals, lcp, cls, inp, seo]
---

Downloading the same image as both PNG and JPG produced a surprising result (PNG 946B vs. JPG 7KB) — the opposite of the usual assumption that JPG is always smaller. This post walks through why that happened, and how image optimization connects to Core Web Vitals (CWV).

## 1. The Situation

The same image was downloaded in two formats:

- `.png` file: **946B**
- `.jpg` file: **7KB**

It's commonly assumed that "JPG is smaller than PNG," but this case was the reverse. The image in question was a **1080x1080, completely solid-color (black) image**.

## 2. How PNG and JPG Compression Differ

### PNG — Lossless Compression

- Preserves every original pixel exactly; nothing is discarded
- Uses the DEFLATE algorithm (similar principle to ZIP) to compress **repeating patterns**
- Best suited for images with large uniform color areas or few distinct colors (screenshots, logos, simple graphics)

### JPG — Lossy Compression

- Discards color information the human eye barely notices, to reduce size
- Splits the image into 8x8 pixel blocks and compresses them using DCT (Discrete Cosine Transform)
- Best suited for **photographic images** where colors blend smoothly
- Even the simplest possible image carries **mandatory structural overhead** — headers, quantization tables, Huffman tables — typically around 2–5KB

| Image type | Usually smaller format | Why |
|---|---|---|
| Photos (naturally blended colors) | JPG | Lossy compression handles gradients efficiently |
| Screenshots / logos / simple graphics (few colors, sharp edges) | PNG | Lossless compression handles repeating patterns efficiently |

## 3. Why PNG Was Smaller in This Case

The uploaded image was **filled with the exact same black color from edge to edge**, with zero variation.

- **PNG (946B)**: The DEFLATE algorithm can express "this same value repeats to the end" in just a few bytes — an extreme success case for lossless compression.
- **JPG (7KB)**: Regardless of how simple the image content is, the fixed metadata required (headers/tables) still takes up a baseline amount of space — so JPG rarely drops below a certain size floor, no matter how simple the image.

**Conclusion**: The rule isn't "JPG is always smaller" — it's that **JPG wins for photo-like images with complex color variation, while PNG wins for solid-color or simple graphics**. This result was a normal outcome given that the image was a flat, single-color graphic.

## 4. What Is Core Web Vitals (CWV)?

A set of three metrics defined by Google that measure **how pleasant a webpage actually feels to real users**. Rather than measuring raw server speed, CWV captures the loading, interactivity, and visual stability that users **actually experience in their browser**.

| Metric | What it measures | "Good" threshold |
|---|---|---|
| **LCP** (Largest Contentful Paint) | Time until the largest content element on screen finishes rendering | 2.5 seconds or less |
| **INP** (Interaction to Next Paint) | Time from a click/tap/keypress to the page visually responding | Under 200ms |
| **CLS** (Cumulative Layout Shift) | How much visible content unexpectedly shifts during loading | Under 0.1 |

- INP became the official Core Web Vital replacing the older FID (First Input Delay) as of March 2024. If older material mentions FID, it is no longer part of the current CWV set.
- Passing is **not based on averages** — it's based on the **75th percentile (p75)**. At least 75% of visitors need a "good" score on all three metrics for a URL to be considered passing.

## 5. Which Metrics Do Images (PNG/JPG) Actually Affect?

### LCP — The most direct impact

- If the largest element on a page is an image, that image becomes the LCP measurement target.
- The larger the file size, the longer it takes to download, directly delaying LCP. Format choice and compression quality matter here.

### CLS — An indirect impact

- If an image is missing explicit `width`/`height` (or CSS `aspect-ratio`), content below it jumps once the image loads, worsening CLS.
- This is more about **how the HTML/CSS is written** than about file size itself.

### INP — Essentially unrelated

- Images themselves have little effect on INP. INP is mainly driven by JavaScript execution time.

> Note: a difference of a few KB, like the 946B vs. 7KB case discussed here, is effectively negligible on modern networks. Real CWV impact comes from **photos in the hundreds of KB to multi-MB range**, where resolution, compression quality, and adopting modern formats like WebP/AVIF actually matter.

## 6. What Happens When CWV Is Poor?

**① Search Visibility (SEO)**
- Google officially recommends that site owners achieve good Core Web Vitals, stating this aligns with what its core ranking systems seek to reward.
- That said, it's only one of many ranking factors — content quality matters far more. It's best understood as something that can tip the balance between otherwise similar competing pages, not a guarantee of ranking.

**② User Experience**
- Slow loading (LCP), unresponsive interactions (INP), and shifting layouts (CLS) naturally lead to user frustration and higher bounce rates — a reasonable inference from each metric's own definition, even without citing specific bounce-rate statistics here.

**③ Reporting Tools**
- Pass/fail status shows up in the Chrome UX Report (CrUX), PageSpeed Insights, and Search Console, flagged as warnings when it fails.

## 7. What Backend Developers Can Contribute

- **TTFB (Time to First Byte)** is included in LCP measurement. No matter how optimized the frontend is, a slow server response delays LCP.
- Setting proper `Cache-Control` headers on image responses and using a CDN directly helps LCP.
- Server-side image resizing and WebP conversion are also common and effective optimizations.

## TL;DR

- PNG is lossless (great at repeating patterns), JPG is lossy (great at gradients) — **which one ends up smaller depends entirely on the image content**
- For a perfectly solid-color image, PNG can end up dramatically smaller than JPG, since JPG always carries a minimum structural overhead
- **CWV = LCP (loading) + INP (responsiveness) + CLS (visual stability)**, judged at the p75 percentile
- Image file size mainly affects **LCP**; missing `width`/`height` mainly affects **CLS** (INP is largely unrelated)
- Poor CWV can affect SEO ranking signals, user retention, and trigger warnings in Google's reporting tools
- From a backend perspective, the levers are: reducing TTFB, setting cache headers, using a CDN, and server-side image optimization
