---
title: "Next.js SSR vs SSG: Choosing the Right Rendering Strategy for SEO"
excerpt: "A practical guide to picking between Server-Side Rendering and Static Site Generation in Next.js, with real performance benchmarks from production deployments."
tags:
  - Next.js
  - React
  - SEO
  - Performance
date: "2025-07-05"
featured: false
coverEmoji: "⚡"
---

When I joined VirarHub to build their web presence, one of the first decisions was how to handle rendering. Their content changed frequently, they cared deeply about SEO, and performance on mobile networks was a hard requirement. Here's what I learned.

## The Four Rendering Modes

- **CSR (Client-Side Rendering)** — JS renders in the browser. Bad for SEO, great for dashboards.
- **SSG (Static Site Generation)** — HTML built at build time. Blazing fast, but stale data.
- **ISR (Incremental Static Regeneration)** — SSG with time-based revalidation. Best of both worlds for most content.
- **SSR (Server-Side Rendering)** — HTML built per request. Always fresh, but slower TTFB.

## Decision Framework

Ask three questions:

1. **How often does the data change?** Real-time → SSR. Hourly → ISR. Rarely → SSG.
2. **Does the page need to be personalised per user?** Yes → SSR or CSR. No → SSG/ISR.
3. **Is SEO critical?** Yes → avoid CSR. No → CSR is fine.

## Headless CMS Integration

We paired Next.js with Sanity as the headless CMS. The key pattern is ISR with on-demand revalidation via a webhook:

```js
// app/blog/[slug]/page.jsx
export const revalidate = 3600; // revalidate every hour

export async function generateStaticParams() {
  const posts = await sanityClient.fetch(`*[_type == 'post']{ slug }`);
  return posts.map(p => ({ slug: p.slug.current }));
}
```

When an editor publishes a change in Sanity, it hits a Next.js revalidation endpoint, which purges only the affected page's cache — not the whole site.

## Core Web Vitals Impact

After migrating from a WordPress site to Next.js with ISR:

- LCP: 4.2s → 1.1s
- CLS: 0.18 → 0.02
- FID: 180ms → 40ms

The headless CMS approach also reduced content update time from a developer-dependent deploy cycle to an 80% faster self-service workflow for the content team.
