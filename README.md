# Creative OS

**An AI-powered platform that generates advertising creative end to end — from a single brief to finished, scored ad assets.**

Creative OS takes a campaign brief and a brand context, then runs it through a multi-stage AI pipeline that writes the copy, generates the imagery, composes the layout, scores the result, and learns from outcomes over time. One pipeline, multiple formats (carousel, banner, video script), continuous learning.

Built and maintained solo. NestJS + Next.js, deployed on Railway and Vercel.

---

## What it does

Give it a brief like *"launch a fishing app to US beginner anglers"* and a brand, and it produces ready-to-publish ad creative:

- **Carousel ads** — multi-slide Instagram/TikTok/Facebook sequences, each slide written, illustrated, and composed individually
- **Display banners** — static creative across standard ad sizes
- **Video scripts** — VSL and short-form scripts across multiple frameworks

Every asset is generated, rendered to a real image, scored against performance proxies, and surfaced in a live editor where it can be refined or branched into A/B variations.

---

## How it works

```
Brief → Brain → Compositor → Studio → Performance
```

1. **Brief → Concept** — Claude (Sonnet) extracts audience, emotion, core message, and goal from the brief
2. **Concept → Angle** — the system selects from 40+ marketing angles, each with an evolution lineage (it mutates weak angles and exploits winning ones over time)
3. **Angle → Copy** — per-slide AI copywriting with structured slot instructions, streamed to the frontend slide by slide
4. **Copy → Image** — image generation (Flux Schnell / gpt-image-1), stored on a CDN and passed downstream as URLs, not raw data
5. **Image + Copy → Compositor** — a Gemini Vision "art director" analyzes each generated image and decides typography, text placement, color scheme, and overlay treatment; a template engine then renders the final layout (HTML → PNG)
6. **Output → Studio** — finished slides stream into a live editor over SSE
7. **Studio → Editing** — an AI copy editor handles both copy and visual edits; users can branch into variations or apply changes across all slides

---

## The interesting engineering

A few problems that were non-trivial to solve:

**The art director is a vision model, not a heuristic.** Instead of rule-based text placement, a Gemini Vision pass looks at each generated image and returns a structured spec — granular X/Y text position (0–100%), text case, font pairing, overlay style, and a 9-zone focal point. The focal point logic actively keeps text off faces and subjects. Typography is treated as a visual decision the model makes per slide, not a template constant.

**Two render paths for speed.** Image-based slides render through Puppeteer (HTML → PNG). Text-only slides render through Satori (SVG) in ~50ms instead of ~800ms — on a 5-slide carousel that saves nearly 4 seconds. The studio editor skips PNG entirely and previews live HTML in an iframe, so editing is instant.

**Streaming everything.** Generation and variations stream over SSE so the user watches slides arrive one at a time instead of waiting for the full batch. Each slide's full compositor input is persisted, so the editor can re-render and edit without regenerating from scratch.

**A learning loop.** Outcome signals feed an angle-scoring system (EWMA weights). Weak angles get mutated by Claude into new variants; strong angles get exploited for new creative. Scoring blends CTR, engagement, conversion, and clarity proxies.

**Scalability-driven data decisions.** Images live on Supabase CDN as URLs rather than base64 in the database — base64 was producing 5MB+ rows that didn't scale past a handful of users. A Puppeteer request interceptor exists as a fallback for the Alpine Chromium base64 rendering limitation.

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router), TypeScript, Tailwind |
| Backend | NestJS, TypeScript |
| Database | PostgreSQL + Prisma ORM |
| Storage / Auth | Supabase |
| AI — Copy | Anthropic Claude (Sonnet) |
| AI — Vision | Google Gemini (Flash) |
| AI — Images | Flux Schnell / gpt-image-1 (via fal.ai) |
| Video | Kling / Veo |
| Rendering | Puppeteer (image slides) + Satori (text-only) |
| Deployment | Railway (backend) + Vercel (frontend) |

---

## Scale of the system

| Metric | Value |
|---|---|
| Backend TypeScript files | 417+ |
| Frontend TypeScript/TSX files | 153+ |
| Template renderers | 43 unique (60+ template IDs) |
| Font pairings | 20 (model-selected per slide) |
| Marketing angles | 40+ with evolution lineage |
| Formats | Carousel, Banner, Video |
| Platforms | Instagram, TikTok, Facebook, Google Display |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│              Next.js Frontend (Vercel)           │
│   /create · /studio/:id · /campaigns/:id         │
│   SSE streaming · React state machine            │
└──────────────────┬──────────────────────────────┘
                   │ REST + SSE
┌──────────────────▼──────────────────────────────┐
│             NestJS Backend (Railway)             │
│                                                  │
│   Carousel ──► Image (Flux / gpt-image-1)        │
│      │                                           │
│      └──────►  Compositor                        │
│                 │            │                   │
│            Puppeteer    Satori (text-only)       │
│                                                  │
│   Art Director (Gemini Vision)                   │
│   Angle · Campaign · Brand · Scoring · Winner    │
└──────────────────┬──────────────────────────────┘
                   │ Prisma
┌──────────────────▼──────────────────────────────┐
│          PostgreSQL + Supabase Storage           │
└──────────────────────────────────────────────────┘
```

---

## Notes

The Creative OS codebase is private and solo-built. This README is a public-facing overview of the system. Architecture and technical detail are documented internally in `ARCHITECTURE.md` and `TECHNICAL_ASSESSMENT.md`.

A walkthrough or read access to the full codebase is available on request.
