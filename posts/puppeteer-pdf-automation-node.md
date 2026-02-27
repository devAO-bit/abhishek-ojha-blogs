---
title: "Automating PDF Reports with Puppeteer & PDF-lib in Node.js"
excerpt: Cutting 30% of manual reporting time by building a fully automated document generation pipeline. A deep dive into streaming architecture and Puppeteer gotchas.
tags:
  - Node.js
  - Puppeteer
  - Automation
  - PDF
date: "2025-10-03"
featured: false
coverEmoji: "📄"
---

---

## Automating Document Generation at Scale: From Manual Chaos to a 4-Minute Pipeline

Document generation is one of those tasks that sounds easy—until you’re dealing with a dozen report types, deeply nested data, and stakeholders who expect pixel-perfect PDFs every time.

This is the story of how I built an automated document generation pipeline that eliminated hours of repetitive manual work and turned reporting into a fully hands-off process.

---

## The Problem

At one point, the team was generating monthly reports manually using a mix of Excel sheets and Word documents.

The math wasn’t pretty:

* **12 report types**
* **~2 hours per report**
* **An entire workday lost every month**

Beyond the time cost, the process was error-prone, inconsistent, and impossible to scale. My goal was simple: **automate everything**—from data extraction to delivery—without compromising formatting or accuracy.

---

## Choosing the Right Tech Stack

The solution required flexibility, precision, and production-grade reliability. Here’s what I landed on:

* **Puppeteer** — Headless Chrome for converting HTML to pixel-perfect PDFs
* **PDF-lib** — For merging documents, stamping templates, and adding watermarks
* **PptxGenJS** — To generate PowerPoint decks directly from structured data
* **Handlebars** — HTML templating with clean dynamic data injection

Each tool solved a specific problem in the pipeline without unnecessary overlap.

---

## The End-to-End Pipeline

At a high level, the system looks like this:

```
Data Source (MongoDB)
  → Handlebars Template (HTML)
    → Puppeteer (HTML → PDF)
      → PDF-lib (merge + watermark)
        → S3 Upload
          → Email Notification
```

Each step is isolated, testable, and replaceable—making the pipeline easy to extend as new report types are added.

---

## Puppeteer Tips for Production Use

Puppeteer is powerful, but it can become a bottleneck if used naïvely. A few optimizations made a massive difference.

### 1. Launch Once, Reuse the Browser

Spinning up a new Chromium instance per request is painfully slow and resource-intensive. Instead, I used a singleton browser instance shared across jobs.

```js
// browserPool.js
let browser;

export const getBrowser = async () => {
  if (!browser) {
    browser = await puppeteer.launch({
      headless: 'new',
      args: ['--no-sandbox', '--disable-setuid-sandbox'],
    });
  }
  return browser;
};
```

### 2. Wait for Network Stability

When loading external fonts, images, or charts, always render PDFs using:

```
waitUntil: 'networkidle0'
```

This avoids half-rendered documents and layout shifts.

### 3. Set an Explicit Viewport

Matching the viewport to your design’s expected width ensures consistent pagination and alignment across all reports.

---

## Streaming Large PDFs Without Crashing the Server

Some reports exceeded 100 pages. Buffering the entire PDF in memory quickly led to out-of-memory crashes on smaller instances.

The fix? **Stream the PDF directly to S3.**

```js
const stream = await page.createPDFStream({
  format: 'A4',
  printBackground: true,
});

await s3.upload({
  Bucket,
  Key,
  Body: stream,
}).promise();
```

This approach kept memory usage flat, regardless of document size.

---

## Results

* All **12 reports generated in under 4 minutes**
* Fully automated via a **midnight cron job**
* Zero manual intervention since deployment

On paper, it saved about **30% of the team’s time**. In reality, it felt like much more—because it eliminated an entire class of tedious, failure-prone work.

---

## Final Takeaway

Document generation isn’t just about exporting PDFs—it’s about building a system that’s:

* Deterministic
* Scalable
* Maintainable
* And invisible to the team once it’s live

If you’re still manually assembling reports, that’s not a process problem—it’s an automation opportunity.
