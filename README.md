# Betterhomes — Storytelling Marketing Framework

An internal implementation guide for the Betterhomes digital marketing team.

This repo translates the "story, not sale" framework (Markov / LSTM / GNN / Transformer-driven journey marketing) into something concrete and buildable for **bhomes.com** — anchored in the data we already have in PostHog, with an honest list of what's broken, what's missing, and what to fix first.

> **Premise:** We are no longer chasing the keyword that wins the click. We are architecting the experience that wins the buyer. Every touchpoint is a story beat. Our job is to know which beats convict, which beats break, and which beats are missing entirely.

---

## Table of Contents

1. [What the data actually says today](#1-what-the-data-actually-says-today)
2. [The critical finding — custom event tracking is broken](#2-the-critical-finding--custom-event-tracking-is-broken)
3. [The buyer journey — six story beats](#3-the-buyer-journey--six-story-beats)
4. [The stack — PostHog-first, free tools where possible](#4-the-stack--posthog-first-free-tools-where-possible)
5. [The ML layer, translated for bhomes (free-tier edition)](#5-the-ml-layer-translated-for-bhomes-free-tier-edition)
6. [The dashboards we actually need](#6-the-dashboards-we-actually-need)
7. [The four metrics that matter](#7-the-four-metrics-that-matter)
8. [Implementation roadmap (phased)](#8-implementation-roadmap-phased)
9. [What I need from you to move this forward](#9-what-i-need-from-you-to-move-this-forward)

---

## 1. What the data actually says today

Numbers below are pulled directly from PostHog (project 198002, "Websites") on 2026-05-11.

### 1.1 Property categories — where attention actually goes

Last 180 days, pageviews by URL category:

| Category | Pageviews | Unique users | Sessions | Share of named categories |
|---|---:|---:|---:|---:|
| Sale | 225,017 | 160,285 | 165,891 | **42%** |
| Rent | 156,507 | 103,723 | 109,283 | **29%** |
| Blog / Market Reports | 105,169 | 73,257 | 79,236 | 20% |
| Property Detail (uncategorized `/property/<slug>`) | 139,260 | 94,321 | 100,308 | — (overlaps above) |
| Agent Pages | 19,303 | 11,765 | 13,896 | 4% |
| Off-plan | 18,472 | 11,129 | 12,905 | 3% |
| Area Guides | 7,948 | 6,162 | 6,973 | 1% |
| Commercial | 273 | 216 | 220 | < 0.1% |

**What this tells us.** Sale + Rent are roughly 70% of intent. Off-plan, which the team invests heavily in (4 Motion ad workspaces — Al Ansari Nova Tower, Vayla, betterhomes offplan 1, betterhomes marketing), is only ~3% of organic pageviews. Either off-plan demand is paid-channel-only (likely — those Motion campaigns drive direct traffic that doesn't show in organic browsing), or organic discovery of off-plan is underbuilt. Blog/Market Reports is **third-biggest at 105k pageviews** — content marketing is doing real work in awareness/discovery.

### 1.2 Top areas — what Dubai geography people care about

Top 10 Area Guide pages, last 90 days (real human-readable areas, not internal listing IDs):

| Rank | Area | Views | Users |
|---|---|---:|---:|
| 1 | Dubai Creek Harbour | 539 | 495 |
| 2 | Damac Hills 2 | 259 | 208 |
| 3 | Saadiyat Island | 186 | 168 |
| 4 | Al Barsha | 138 | 114 |
| 5 | Green Community | 130 | 121 |
| 6 | Dubai Marina | 126 | 94 |
| 7 | Arabian Ranches | 123 | 112 |
| 8 | Majan | 123 | 106 |
| 9 | Sheikh Zayed Road | 113 | 98 |
| 10 | Emaar Beachfront | 96 | 84 |

These are the areas the metrics should weight toward. Note **Saadiyat Island and Yas Island** in the top 20 — Abu Dhabi geography, not Dubai — meaning the framework needs to handle multi-emirate journeys, not just Dubai.

### 1.3 Geographic distribution of users — who is actually visiting

Last 90 days, top locations:

| Location | Pageviews | Unique users | Interpretation |
|---|---:|---:|---|
| United States — Ashburn, VA | 211,437 | 108,343 | **Almost certainly bots** — Ashburn is the world's largest AWS datacenter cluster |
| China (no city) | 60,952 | 60,898 | **Likely bots / proxies** — null city + 1:1 user:pageview ratio is suspicious |
| **United Arab Emirates — Dubai** | **58,725** | **28,355** | **Primary real audience** |
| United States — Columbus, OH | 26,828 | 21,478 | Mixed — some real expat traffic, some bot |
| Singapore | 13,063 | 12,639 | Real — investor / expat audience |
| Hong Kong | 12,723 | 12,659 | Real — investor / expat audience |
| **United Arab Emirates — Abu Dhabi** | **10,670** | **6,031** | Real — second UAE city |
| **United Arab Emirates — Sharjah** | **4,543** | **2,847** | Real — third UAE city |
| India — Mumbai | 1,430 | 742 | Real — common origin for Dubai property buyers |
| Pakistan — Lahore | 5,798 | 547 | Mixed — 547 users / 5,798 views = ~10 views/user, plausible |

**What this tells us.** The team has a measurable bot-traffic problem — the PostHog project's `test_account_filters` currently only filters `localhost`. We are over-counting traffic by 2–3x in most dashboards. **Add a bot filter in Phase 0** (PostHog has built-in bot detection that can be enabled). The real audience splits cleanly into: domestic UAE (Dubai/Abu Dhabi/Sharjah ≈ 37k unique users) and international investors (Singapore/Hong Kong/Mumbai ≈ 16k unique users).

### 1.4 SEO baseline (Semrush UAE, 2026-05)

- bhomes.com: rank 444 in UAE database
- 11,825 organic keywords
- ~180,735 monthly organic visits
- 0 paid keywords (we're 100% organic for SEO — paid is on Meta via Motion)

---

## 2. The critical finding — custom event tracking is broken

This is the single most important thing in the doc. Read this before anything else.

**Finding.** Running a HogQL query against the events table for the last 30 days, filtering for any event that is not a standard PostHog system event (`$pageview`, `$pageleave`, `$autocapture`, `$rageclick`, `$web_vitals`, `$feature_flag_called`, `$exception`):

> **Zero rows returned.**

Not one custom event has fired in the last 30 days. The 50+ enquiry events in the taxonomy (`Schedule Viewing For Sales Enquiry`, `Property Valuation Enquiry`, `Contact us Enquiry`, etc.) — last fired **2025-11-03**. The viewing events (`Book a Viewing`, `Request a Viewing`) — same date. The property page custom events (`similar_listing_cta_clicked`, `overlay_displayed`) — last fired **2026-02-03**.

**What this means.**

- The 6-stage buyer journey dashboard (1487354) currently works because Zahra wrote it on top of `$pageview` + URL patterns, not on top of the custom event taxonomy. Smart fallback. But it's a proxy funnel, not a true funnel.
- Every "Enquiry" dashboard the team has built is showing historical data that stopped updating six months ago. Anyone reading those dashboards today is looking at a frozen lake.
- The `lead_submitted` event isn't just missing — the entire conversion event layer is dark.

**Why this likely happened.** Most likely cause: a tracking-snippet change, a CMS migration, or a frontend release dropped the `posthog.capture('...')` calls. The events that still fire (`$pageview`, autocapture, etc.) are the ones PostHog captures automatically without code. Everything that requires an explicit `capture()` call has stopped.

**This is Phase 0 priority #1, before anything else in this guide.** No ML, no Markov, no journey dashboards are worth building on broken event tracking.

---

## 3. The buyer journey — six story beats

Zahra's 6-stage funnel is the right backbone. We use it as the canonical journey across every channel.

| Stage | Story beat | Buyer's "Aha" | Signal | Live today? |
|---|---|---|---|---|
| 1. Awareness | "Dubai is where I should be looking." | Category is real and relevant. | Branded + non-branded impressions | Semrush (yes), paid (yes via Motion) |
| 2. Discovery | "Betterhomes exists and seems credible." | First touch on bhomes.com. | First `$pageview` per `distinct_id` | ✅ Yes |
| 3. Research | "These properties match my situation." | Property + area-guide engagement. | Property page views, scroll depth, time-on-page | ⚠️ Pageview-only (custom events dark) |
| 4. Shortlist | "I'm narrowing to a few I'd pursue." | Repeat visits, favorites, CTA hovers. | `Add to Favorites`, `similar_listing_cta_clicked` | ❌ Dark since Feb 2026 |
| 5. Contact | "I'm ready to talk to a human." | Phone/WhatsApp/email click, form submit. | Contact page reach (proxy), 50+ Enquiry events | ❌ Dark since Nov 2025 |
| 6. Conversion | "I picked Betterhomes." | Signed deal — lead → close. | `lead_submitted` + CRM | ❌ Not instrumented; CRM (Engage/Metabase) not accessible to us |

Stages 4–5 are technically tracked in the schema but not firing. Stage 6 needs both (a) `lead_submitted` instrumentation, and (b) future CRM access.

---

## 4. The stack — PostHog-first, free tools where possible

We're operating PostHog-only for now (CRM access via Engage/Metabase is deferred until we get visibility into it). Free ML tooling where possible. No paid procurement asks in the near term.

### 4.1 What we use

| Layer | Tool | Status |
|---|---|---|
| Web/product analytics | **PostHog** (project 198002) | Primary source of truth |
| HogQL queries | PostHog data warehouse | Already used by Zahra's dashboards |
| Sankey / Paths / Funnels / Cohorts | PostHog native features | Underused — Phase 1 unlock |
| Session replay + heatmaps + rage clicks | PostHog | Live, 90-day retention |
| Feature flags + experiments + surveys | PostHog | Live, underused for surveys (good for sentiment Phase 3) |
| Meta ad creative analytics | Motion (4 workspaces) | Meta only |
| SEO research | Semrush | Use via MCP for keyword + competitor research |
| Python / ML environment | **Google Colab (free)** for notebooks, **local Python** on your PC for prototyping | No procurement needed |
| LLM inference for sentiment | Claude / OpenAI / Gemini API — pay-per-call, minimal $ | No procurement needed |

### 4.2 What we're deferring (not blocking us today)

- **CRM data join** (Engage / Metabase). Once we get access, pCLV becomes real. Until then, journey-based proxies.
- **Supermetrics renewal**. License expired 2026-04-24. We can revisit once PostHog Marketing Analytics has fully replaced what we need, or once a specific source (e.g. GA4, Google Ads) becomes a bottleneck.
- **TikTok / LinkedIn / Google Ads connectors**. Not needed yet — Meta + organic + AI search cover today's mix.

### 4.3 Free ML environment options (pick one or use both)

| Option | Good for | Limits |
|---|---|---|
| **Google Colab (free tier)** | Markov attribution, basic LSTM, sentiment scoring via API, exploratory notebooks | 12-hour runtime cap, free GPU is small (T4), no persistent storage (save to Drive) |
| **Local Python on your PC** | Markov, transition matrices, dashboarding, anything that doesn't need GPU | Limited by your machine; fine for everything Phase 0–2 |
| Kaggle Notebooks (free) | LSTM training, free GPU sessions (30 hrs/week) | Same limits as Colab roughly |
| **GitHub Codespaces (60 free hrs/month for individual accounts)** | Same as local Python but in cloud | Hours cap; no GPU on free tier |
| Hugging Face Spaces (free) | Hosting a small Transformer model behind an API | Sleeps after inactivity; CPU only on free tier |

**Recommended starting setup:** Colab for any notebook work + local Python for everyday scripts. Free, no IT involvement, runs anywhere.

---

## 5. The ML layer, translated for bhomes (free-tier edition)

Four model families from the original framework. For each: what it does for bhomes, what data it needs, what we can build today on PostHog + Colab.

### 5.1 Markov Chains + Hidden Markov Models — Multi-Touch Attribution

**For bhomes.** A typical Dubai buyer journey touches 6–12 surfaces: a Property Finder listing, an Instagram reel, a Google search for "apartments in Dubai Creek Harbour", a market-report download, a return visit, a WhatsApp tap. Last-click attribution credits whichever fired last (usually branded search or direct). Markov tells us which steps actually drive conversion via removal effect.

**Data.** Ordered event sequence per `distinct_id`. **Right now this is pageview sequences only** (custom events are dark). That still works — pageview type + URL pattern gives us a meaningful step taxonomy.

**Build today, free.** HogQL pull → Colab notebook:

```python
# In Colab — install once
!pip install pandas posthog

# Pull ordered sessions
import requests
PROJECT = 198002
API_KEY = "phx_<personal_api_key>"  # generate in PostHog account settings
query = """
SELECT distinct_id,
       arraySort(x -> x.2, groupArray((properties.$current_url, timestamp))) AS path,
       max(if(properties.$current_url LIKE '%/contact%', 1, 0)) AS reached_contact
FROM events
WHERE event = '$pageview'
  AND timestamp >= now() - INTERVAL 90 DAY
GROUP BY distinct_id
HAVING length(path) >= 2
"""
# Hit https://us.posthog.com/api/projects/{PROJECT}/query/ with the HogQL query
# Then build a transition matrix and compute removal effect in pandas
```

We will iterate this query into a proper script in Phase 2.

**HMM (hidden states like "curious", "comparing", "ready").** Needs `hmmlearn` in Python. Runs fine in Colab on CPU. Phase 3.

### 5.2 RNNs and LSTMs — Next-Action Prediction

**For bhomes.** Given the first 3–5 pages a user visits, predict whether they'll reach the contact page or drop off. Useful for: deciding when to fire an overlay (`overlay_displayed` infrastructure exists even if dark right now), trigger a WhatsApp ping, or surface a different listing.

**Data.** Tokenized page sequences. We have this.

**Build.** Train a small GRU (faster than LSTM, comparable results) in Colab with Keras. Score live sessions via PostHog's reverse-ETL or by writing predictions back as a person property via the API. **Realistic Phase 3 build:** ~2–3 weeks once Phase 0 (event tracking) is fixed.

### 5.3 Graph Neural Networks — Hidden touchpoint relationships

**For bhomes.** Finds non-obvious combinations: "users who read Q1 market report AND visit Dubai Creek Harbour area guide AND view 3+ listings convert at 8x base rate." Patterns last-click and Markov miss.

**Verdict.** **Don't build this in 2026.** It needs (a) custom events firing, (b) CRM outcomes for training labels, (c) a non-trivial PyTorch Geometric setup. Tag it as the Phase 4 frontier.

### 5.4 Transformers — Sentiment and Intent

**For bhomes.** Read text → output sentiment + intent. Sources we could plug in:

| Source | How to get it | Status |
|---|---|---|
| **PostHog Surveys** | Enabled, underused. Launch one survey on key pages today, get free text in 24 hours. | Easiest start |
| Google Business Profile reviews | Free API, per branch | Medium — needs API key |
| Property Finder agent reviews | Likely no public API; would need scraping or partnership | Hard |
| Bayut agent reviews | Same | Hard |
| Trustpilot | Free tier API | Easy |
| Support transcripts | If Betterhomes uses Zendesk/Intercom/Freshdesk | Depends on tool |

**Build.** No model training required. Pipe text into Claude or OpenAI API ($1–5/month at our volume) and ask for sentiment + intent label. Aggregate in PostHog as a person/property dimension.

**First build:** Launch a 1-question PostHog Survey ("What's the one thing that almost stopped you from contacting us?") on the contact page. Score responses with Claude weekly. Phase 3 candidate that doesn't need CRM data.

---

## 6. The dashboards we actually need

### 6.1 Sankey path visualization

PostHog Paths insight does this natively. Build one with URL-based waypoints (since custom events are dark):

```
Entry channel → Top landing page → Property/area page → Contact page → Exit
```

Status: build-ready in PostHog UI once Phase 0 fixes bot-filter contamination.

### 6.2 Predictive cohort grids

PostHog Cohorts is enabled. Group by **journey type**, not signup date. Recommended cohorts:

- "AI-referred" (Copilot/Bing leads — non-obvious finding from Zahra's dashboard)
- "Market-report-first" (entered via Blog/Reports — 20% of named-category pageviews)
- "Off-plan-only researcher" (3% of organic, big paid investment)
- "High-frequency returner, no contact" (cold leads worth re-engaging)
- "Investor expat" (Singapore/HK/Mumbai geo + sale-category interest)

### 6.3 Real-time velocity tracking

PostHog supports live event streams. Once Phase 0 is done, we can show:
- Active sessions by stage
- Avg minutes per stage in the last hour
- Stage-to-stage transition rate
- Friction signals: `$rageclick`, `$exception`, dead-clicks

---

## 7. The four metrics that matter

### 7.1 Time-to-Value (TTV)

Seconds between first `$pageview` and first meaningful "Aha" — practical proxy: first time on a property page after a search.

**Today:** HogQL-buildable on pageviews alone. Build it.

### 7.2 Micro-Conversion Velocity

Cumulative "yes" events per session per stage.

**Today:** Depends on the custom events being live. Tracked as a stretch goal until Phase 0 closes.

**Interim proxy:** scroll depth + time-on-page + pages-per-session per stage, all derivable from pageviews.

### 7.3 Path-to-Purchase Complexity

Median count of distinct page types between first `$pageview` and a Stage-5 signal (contact page reach), by channel.

**Today:** Buildable. Single best leading indicator of whether the framework is working. **Build this first in Phase 1.**

### 7.4 Predictive Customer Lifetime Value (pCLV)

Forecast revenue per user given journey type + channel + category interest + engagement depth.

**Today: blocked.** Until we get CRM access (Engage / Metabase), we can't train on lead → close. **Interim proxy:** journey-quality score (weighted sum of micro-conversion velocity, stage progression, friction inverse). Treat it as "engagement value" today; upgrade to revenue value when CRM data lands.

---

## 8. Implementation roadmap (phased)

### Phase 0 — Fix the foundation (weeks 1–3)

Goal: get the tracking layer back to a state where every other phase is meaningful.

- **0.1 — Audit custom event firing.** Pull a recent build of bhomes.com and verify which `posthog.capture(...)` calls are still present in the codebase vs. expected. Compare against the 50+ event names in the taxonomy.
- **0.2 — Restore conversion event tracking site-wide.** Specifically: every enquiry form on bhomes.com (main, blog, off-plan, Unbounce), every WhatsApp/phone/email CTA, every viewing-related action.
- **0.3 — Add the unified `lead_submitted` event** alongside existing form-specific events, with properties `form_type`, `property_id`, `enquiry_type`, `area_guide` (if applicable). This is the one event the journey funnel needs.
- **0.4 — Enable PostHog bot filtering.** Currently `test_account_filters` only excludes localhost. Add country-based or AWS-IP-range filters for the obvious bot sources (Ashburn, bare-China-no-city). Even a simple `$geoip_city_name = 'Ashburn'` exclusion rule would clean up most dashboards immediately.
- **0.5 — Document the canonical event spec** in this repo so the team has one place to look up "what should fire when".

**Exit criteria.** Running the "any custom event in last 7 days" query returns non-zero rows. Director Overview dashboard (1312369) reflects real, current activity.

### Phase 1 — PostHog-native quick wins (weeks 4–7)

No external tooling.

- **1.1** Sankey path insight from entry channel → contact page.
- **1.2** Five journey-type cohorts (Section 6.2).
- **1.3** Path-to-Purchase Complexity dashboard tile + TTV tile (the two metrics buildable today).
- **1.4** HogQL view that tags each session with its current stage; reuse across dashboards.
- **1.5** Launch 1 PostHog Survey on the contact page for the Transformer sentiment pipeline (Phase 3 prep).

### Phase 2 — HogQL attribution + journey scoring (weeks 8–11)

- **2.1** Markov attribution in HogQL → Colab notebook → channel report. First "Markov vs. last-click" report shared with directors.
- **2.2** Journey-quality score (engagement value) as a session property, written back to PostHog via API.

### Phase 3 — ML layer in Colab (weeks 12–20)

- **3.1** GRU/LSTM next-action predictor in Colab, scored back into PostHog as `dropoff_risk_score` person property.
- **3.2** HMM with 3 hidden states ("curious", "comparing", "ready") on event sequences.
- **3.3** Transformer sentiment via Claude/OpenAI API on accumulated PostHog Survey responses + (if available) Google Business Profile reviews via free API.

### Phase 4 — Advanced (parked)

GNN, full pCLV with CRM data — only after we get Engage/Metabase access and Phases 0–3 are paying off.

---

## 9. What I need from you to move this forward

To start Phase 0, I need decisions / info on:

1. **Who owns the bhomes.com frontend code?** Phase 0 is mostly an engineering ask — someone needs to (a) audit which `posthog.capture(...)` calls are still in the codebase, (b) restore the missing ones, (c) add `lead_submitted`. Whoever owns the main site + the blog + the off-plan microsites + Unbounce. Could be one team or four.
2. **When can we get read-only access to Engage / Metabase** (or the Engage CRM data layer)? Even a CSV export of "lead created → outcome" per month would unblock pCLV in Phase 2 rather than Phase 4.
3. **Bot filter scope.** Are you OK with us aggressively filtering Ashburn + bare-China-no-city + AWS IP ranges from all marketing dashboards? Director Overview will look smaller but truer.
4. **Confirmation on category weighting for metrics.** Based on what I'm seeing in PostHog: **Sale (42%) and Rent (29%) should be the primary focus**, Blog/Reports (20%) is the awareness driver, Off-plan (3% organic but heavy paid via Motion) gets a separate "paid funnel" view, and Area Guides (1%) get used as the geo-routing layer (Dubai Creek Harbour, Damac Hills 2, Saadiyat Island lead the list). Does that match how you'd weight it internally?
5. **Survey question for Phase 1.** I suggested "What's the one thing that almost stopped you from contacting us?" on the contact page. Do you want to phrase it differently, or run it elsewhere?

Answer 1 and 3 and I can start drafting Phase 0 tickets. The others can wait a week.

---

*Maintained by Digital Marketing. Branch: `claude/storytelling-marketing-framework-VEC1F`.*
