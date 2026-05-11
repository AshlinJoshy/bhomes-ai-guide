# Betterhomes — Storytelling Marketing Framework

An internal implementation guide for the Betterhomes digital marketing team.

This repo translates the "story, not sale" framework (Markov / LSTM / GNN / Transformer-driven journey marketing) into something concrete and buildable for **bhomes.com** — anchored in the data we already have, with an honest list of what's missing and what to request.

> **Premise:** We are no longer chasing the keyword that wins the click. We are architecting the experience that wins the buyer. Every touchpoint is a story beat. Our job is to know which beats convict, which beats break, and which beats are missing entirely.

---

## Table of Contents

1. [Where Betterhomes actually is today](#1-where-betterhomes-actually-is-today)
2. [The buyer journey — six story beats](#2-the-buyer-journey--six-story-beats)
3. [The stack — what's connected, what's broken, what's missing](#3-the-stack--whats-connected-whats-broken-whats-missing)
4. [The ML layer, translated for bhomes](#4-the-ml-layer-translated-for-bhomes)
5. [The dashboards we actually need](#5-the-dashboards-we-actually-need)
6. [The four metrics that matter](#6-the-four-metrics-that-matter)
7. [Implementation roadmap (phased)](#7-implementation-roadmap-phased)
8. [Tools and connectors to request](#8-tools-and-connectors-to-request)
9. [What I need from you to move this forward](#9-what-i-need-from-you-to-move-this-forward)
10. [Appendix: LinkedIn post derivatives](#10-appendix-linkedin-post-derivatives)

---

## 1. Where Betterhomes actually is today

This is not a greenfield exercise. The team has already built a strong analytics foundation. The framework below is designed to extend that — not replace it.

**What's live (verified May 2026):**

| Layer | Status | Reference |
|---|---|---|
| Web analytics (PostHog, project "Websites" id 198002) | Live since Jul 2025, fully onboarded | All bhomes.com + eos.bhomes.com + mobile app |
| Custom event taxonomy | 50+ "Enquiry" event types, full viewing/property funnel events | `Schedule Viewing For Sales Enquiry`, `Property Valuation Enquiry`, `similar_listing_cta_clicked`, etc. |
| AI-channel detection | ChatGPT, Copilot/Bing, Perplexity, Gemini, Claude.ai, "For you" (engageplatform.ai) | Custom channel rules on project — ahead of most brokerages |
| 6-stage SEO+AIO buyer journey | Built Apr 2026 by Zahra | Dashboard 1487354 — Awareness → Discovery → Research → Shortlist → Contact → Conversion |
| Session replay + heatmaps + rage clicks | 90-day retention, opt-in true | Friction signals already captured |
| Feature flags + experiments | Enabled | A/B infrastructure ready |
| Meta ads creative analytics (Motion) | 4 workspaces: Al Ansari Nova Tower, Vayla, betterhomes marketing, betterhomes offplan 1 | Meta only |
| SEO baseline (Semrush, UAE) | 11,825 organic keywords, ~180,735 monthly organic visits | Rank 444 in UAE |

**What we know is missing — already flagged by the team:**

- `lead_submitted` event is **not instrumented**. Zahra's funnel marks it as "⚠ needs tracking" at Stage 5. Without it, every conversion-rate number we report is a **proxy** (contact-page reach), not the real thing. This is gap #1 and everything else downstream depends on closing it.
- Supermetrics license **expired 2026-04-24** (trial). Until renewed or replaced, we have no automated pull of Google Ads, Google Analytics, LinkedIn Ads, TikTok Ads, etc. into one place.
- No CRM / sales-outcome data is joined to web behavior in PostHog. We can see leads enter; we cannot see which leads become deals, deal size, or time-to-close. **Without this, pCLV is impossible.**

---

## 2. The buyer journey — six story beats

The team's existing 6-stage funnel is the right backbone. We use it as the canonical journey across every channel — SEO, AIO (AI chatbots), paid social, direct, referral, agent-driven.

| Stage | Story beat | "Aha" the buyer needs | Primary signal | Where we measure it |
|---|---|---|---|---|
| 1. Awareness | "Dubai is where I should be looking." | The category is real and relevant to me. | Branded + non-branded impressions | Semrush organic + Motion paid reach |
| 2. Discovery | "Betterhomes exists and seems credible." | First-touch on bhomes.com from any channel. | First `$pageview` per `distinct_id` | PostHog — already in dashboard 1487354 |
| 3. Research | "These properties match my situation." | Engagement with listings, area guides, market reports. | Property page views, `similar_listing_hover`, `View Photos`, `Area Guide Details Enquiry` | PostHog event taxonomy is mature here |
| 4. Shortlist | "I'm narrowing to a few I'd actually pursue." | Repeated visits to the same listings; favorites; CTA hovers without clicks. | `Add to Favorites`, `Add to Favorites on PLP`, `Add to Likes`, `similar_listing_cta_clicked`, `view_properties_cta_clicked` | Already tracked |
| 5. Contact | "I'm ready to talk to a human." | Phone / WhatsApp / email click; form submit; viewing booked. | `Attempt Contact Agent`, `Book a Viewing`, `Request a Viewing`, the 50+ `Enquiry` events | Tracked at the *attempt* level, **not always at the success/submit level** |
| 6. Conversion | "I picked Betterhomes." | Signed deal — lead → tenancy / sale closed. | `lead_submitted` + CRM deal-closed status | **Not yet tracked** — gap #1 |

The job of the framework is to make this journey **measurable end-to-end, predictable mid-funnel, and intervenable in real time.**

---

## 3. The stack — what's connected, what's broken, what's missing

### Connected and healthy

- **PostHog (project Websites, id 198002).** Full session-level event stream for bhomes.com, eos.bhomes.com, mobile app (Amplitude events imported). HogQL available for any custom query. 34 dashboards live. Already running funnels, paths, retention.
- **Motion (org "Bhomes", 4 workspaces).** Meta ads creative analytics. Useful for Stages 1–2 (Awareness, Discovery) on paid social. Meta-only — no TikTok / YouTube / LinkedIn coverage here.
- **Semrush.** Full domain research, keyword research, backlink, traffic analytics, competitive intel for the UAE database (and globally).

### Connected but broken / unused

- **Supermetrics — license expired 2026-04-24.** Until renewed, we cannot auto-pull Google Ads, GA4, Meta Ads (independent of Motion), TikTok Ads, LinkedIn Ads, Microsoft Ads, Bing, Search Console, HubSpot, Shopify, etc. into one queryable place. 169 platforms are supported once active.

### Missing / not connected

| Missing data | Why it matters | Suggested source |
|---|---|---|
| **CRM sales-outcome data** (lead → viewing → offer → close) | Without it: no pCLV, no real attribution to revenue, no closed-loop on which journeys actually pay. | Salesforce / HubSpot / Bayut / Property Finder feed → PostHog data warehouse via webhook or Supermetrics |
| **Google Analytics 4 + Google Search Console** | GSC keywords by landing page, GA4 cross-device, paid + organic union. Semrush is third-party estimates; GSC is ground truth. | Supermetrics (once renewed) or PostHog Marketing Analytics product (already shown intent) |
| **Reviews / sentiment text** (Google Business, Trustpilot, Property Finder agent reviews, support transcripts) | Required for Transformer sentiment/intent analysis. Right now we have zero brand-sentiment time series. | Trustpilot API, Google Business Profile API, in-app surveys |
| **ML compute environment** | Markov chains we can run in HogQL. LSTM / GNN / Transformer training cannot run inside PostHog. | One of: Databricks, AWS SageMaker, Google Vertex AI, or a single GPU instance + Python notebooks for prototyping |
| **TikTok, YouTube, LinkedIn organic + paid** | Multi-touch attribution needs every touchpoint. Right now we are blind to anything outside Meta and Google. | Supermetrics covers all three — single highest-leverage unlock once renewed |
| **Email engagement** (open/click → on-site behavior) | Newsletters drive Stages 1–3; we need them linked to the same `distinct_id`. | Mailchimp / Klaviyo / HubSpot tracking pixel firing PostHog `$identify` |

---

## 4. The ML layer, translated for bhomes

The LinkedIn post listed four model families. Here is what each one actually means for a Dubai brokerage, what data it requires, and whether we can do it today.

### 4.1 Markov Chains + Hidden Markov Models — Multi-Touch Attribution

**What it does.** Given a sequence of touchpoints per user (e.g. `Instagram ad → bhomes.com area guide → property page → WhatsApp click → form submit`), it calculates the **removal effect** of each touchpoint — what fraction of conversions would disappear if that touchpoint did not exist. This is how you replace last-click attribution.

**Why it matters for bhomes.** A typical Dubai buyer journey touches 6–12 surfaces before they call an agent: a Property Finder listing, an Instagram reel, a Google search, a market report download, a returning visit, a WhatsApp tap. Last-click attribution credits whichever one happened to fire last — usually direct or branded search — and starves the channels that actually built the case.

**Data needed.** Ordered event sequence per `distinct_id`, with the conversion endpoint defined (right now: any `Enquiry` event; eventually: `lead_submitted` and then CRM-closed-deal).

**Where we are.** PostHog already has all the sequence data. We can run a basic Markov attribution **today** with HogQL + a Python script that pulls the event log via the PostHog API. Sketch:

```sql
SELECT distinct_id,
       arraySort(x -> x.2, groupArray((event, timestamp))) AS touchpoints,
       max(if(event LIKE '%Enquiry%' OR event = 'lead_submitted', 1, 0)) AS converted
FROM events
WHERE timestamp >= now() - INTERVAL 90 DAY
GROUP BY distinct_id
HAVING length(touchpoints) >= 2
```

Export → Python (`pandas` + `pychattr` or a hand-rolled transition matrix) → removal-effect per channel.

**Gap.** HMM (hidden states — "curious", "comparing", "ready") requires latent-state modeling that's beyond HogQL. Build in a notebook environment.

### 4.2 RNNs and LSTMs — Next-Action Prediction

**What it does.** Given the last N events for a user, predicts the next event and its probability. Trained on historical sequences where we know the outcome.

**Why it matters for bhomes.** Real-time intervention. If the model predicts "this user is 80% likely to drop off in the next 2 minutes," we can fire an overlay (we already have `overlay_displayed` events — the infrastructure is there), trigger a WhatsApp ping, or surface a different listing.

**Data needed.** Tokenized event sequences with timestamps. PostHog has this.

**Where we are.** Data is ready. Model training is not — we need a Python environment, GPUs (or just CPU for a small LSTM), and a way to score live sessions back into PostHog as a person property (`predicted_next_action`, `dropoff_risk_score`) so we can target overlays / feature flags / workflows on it.

**Realistic first build.** A binary classifier — "will this session end in an Enquiry event?" — using a simple GRU/LSTM on the first 5 events of each session. We can ship this in 2–3 weeks once we have a Python env.

### 4.3 Graph Neural Networks — Hidden touchpoint relationships

**What it does.** Treats users, touchpoints, content, and agents as nodes in a graph; edges are interactions. Learns which combinations of touchpoints (not individual ones) drive conversion.

**Why it matters for bhomes.** "A user who reads the Q1 market report AND visits an area guide AND looks at 3+ off-plan listings converts at 8x the base rate" — that's the kind of insight last-click and even Markov cannot give you. GNNs find these multi-hop patterns automatically.

**Data needed.** The same event stream, but transformed into a graph. Plus content metadata (which area guide, which listing, which agent, which area).

**Where we are.** This is the most ambitious of the four. **Don't build this until phases 1–3 are working and we have the lead → close loop closed.** Without `lead_submitted` and CRM outcomes, the graph has nothing to optimize against.

### 4.4 Transformer Models — Sentiment and Intent

**What it does.** Reads text (reviews, support transcripts, comments, survey responses) and outputs sentiment, intent, and topic. Modern variants (e.g. an off-the-shelf BERT-family model, or just calling Claude / GPT via API) do this well out of the box.

**Why it matters for bhomes.** Are people telling Google Reviews that our agents are slow to respond? Are off-plan buyers complaining about a specific developer in support tickets? Is the brand story landing emotionally, or is it landing as "another brokerage"?

**Data needed.** **Text we do not currently have in one place.** Sources to assemble:
- Google Business Profile reviews (per branch / per agent if linked)
- Property Finder / Bayut agent reviews
- Trustpilot
- Support ticket transcripts (if Zendesk / Intercom / Freshdesk is in use)
- In-app survey responses (PostHog Surveys is enabled but underused)
- Social comments on Meta posts (Motion does not pull these — would need a separate listener)

**Where we are.** Easiest of the four to start once text is available. We do not need to train anything — we can call Claude or OpenAI via API and get sentiment + intent labels in one prompt. The work is the data plumbing, not the model.

---

## 5. The dashboards we actually need

### 5.1 Sankey — story-flow visualization

PostHog Paths insight already does Sankey diagrams natively. We have not used them for the full journey yet. Build one with these waypoints:

```
Entry channel → Stage 2 page type → Stage 3 engagement event → Stage 4 shortlist signal → Stage 5 contact action → Stage 6 outcome
```

This single visual tells us where the narrative breaks. Status: build-ready in PostHog UI, no new infrastructure needed.

### 5.2 Predictive cohort grids

PostHog Cohorts is enabled. Today we group by signup date. The framework asks us to group by **journey type**:

- "Read market report first" cohort
- "Came via ChatGPT" cohort (Copilot/Bing is actually our largest AI source per Zahra's dashboard — interesting)
- "Off-plan-only researcher" cohort
- "High-frequency returner, no contact" cohort (these are the cold leads worth re-engaging)

Each cohort gets compared on time-to-Enquiry, Enquiry rate, and (once we have it) deal-close rate.

### 5.3 Real-time velocity tracking

PostHog supports live event streams. The dashboard we want shows, in real time:
- Active sessions by stage (how many users currently in Discovery vs. Research vs. Shortlist)
- Avg minutes spent per stage across the last hour
- Stage-to-stage transition rate
- Friction signals: `$rageclick`, `$exception`, time-on-page > 90s without scroll

Status: build-ready, but requires designing the stage-classification logic (HogQL view that labels each session's current stage).

---

## 6. The four metrics that matter

These are the framework's headline metrics, instantiated for bhomes.

### 6.1 Time-to-Value (TTV)

**Definition for bhomes.** Seconds between first `$pageview` and the user's first meaningful "Aha" event — typically the first time they view a property page that matches their later-stated criteria (location + price + bedrooms).

**Why it's hard.** We don't yet capture "criteria" cleanly. The closest proxy is the first listing they click on after a search, treating its attributes as revealed preference.

**Quick win.** Measure TTV as "seconds from first `$pageview` to first `View Photos` event" — this is rough but actionable today.

### 6.2 Micro-Conversion Velocity

**Definition for bhomes.** Cumulative count of small "yes" events per session per stage:
- Stage 2 micros: scroll past hero, view top nav
- Stage 3 micros: `View Photos`, `View Map`, `similar_listing_hover`, area guide scroll-depth
- Stage 4 micros: `Add to Favorites`, `view_properties_cta_hover`
- Stage 5 micros: hover over phone/WhatsApp/email CTAs, expand contact agent card

Plot velocity (micros per minute) per stage. A user pulling 8 micros/min through Stage 3 is hot; one pulling 1/min is browsing.

**Status.** All events exist. Need a HogQL view + dashboard tile.

### 6.3 Path-to-Purchase Complexity

**Definition for bhomes.** Median count of distinct event types (deduplicated) between first `$pageview` and first `Enquiry` event, by channel.

**What good looks like.** Falling complexity means the story is getting clearer. Rising complexity means we're adding noise (or attracting less-qualified traffic).

**Status.** Computable in HogQL today. This is the single best leading indicator of whether the framework is working.

### 6.4 Predictive Customer Lifetime Value (pCLV)

**Definition for bhomes.** Forecast revenue from a user given (a) journey type, (b) channel, (c) property category interest (rent vs. sale vs. off-plan), (d) engagement depth. Trained on historical lead → closed-deal outcomes.

**Status.** **Blocked on CRM data.** Until lead-to-deal status is in PostHog (or PostHog's data warehouse), this is a wish, not a metric. It is the prize at the end of phase 3.

---

## 7. Implementation roadmap (phased)

Sequenced so each phase produces working value before the next starts. Don't skip phase 0.

### Phase 0 — Fix the foundation (weeks 1–2)

Goal: every other phase is wasted effort if conversion isn't measured.

- **0.1** Instrument `lead_submitted` event everywhere a real lead is created — every form on bhomes.com, blog enquiry forms, WhatsApp click-through that fires a CRM record, off-plan flows, viewings, valuation. This is the team's own flagged gap (Zahra's funnel) and it should not wait.
- **0.2** Decide: renew Supermetrics, or move marketing-data ingestion into PostHog Marketing Analytics (already shown product intent — id 198002 has `marketing_analytics` intent timestamp Jan 2026). Don't run both. Recommend renewing Supermetrics short-term while PostHog Marketing Analytics matures, because Supermetrics covers more sources today.
- **0.3** Choose a single CRM-of-record for sales outcomes and confirm we can webhook deal-stage changes into PostHog. Surface the question to whoever owns CRM at Betterhomes.

**Exit criteria.** A working end-to-end funnel from first `$pageview` → `lead_submitted` → CRM `deal_closed`, queryable in one HogQL query.

### Phase 1 — PostHog-native quick wins (weeks 3–6)

No new tools. Pure leverage of what's already there.

- **1.1** Build a Sankey path insight covering Stage 2 → Stage 6 (now possible because Stage 6 exists).
- **1.2** Define cohorts by journey type, not signup date. Start with 4: "AI-referred", "Market-report-first", "Off-plan-only", "High-frequency-returner".
- **1.3** Add the four metrics (TTV, Micro-Conversion Velocity, Path-to-Purchase Complexity, pCLV-proxy) as a single dashboard. The first three are HogQL queries; pCLV uses average historical deal size by cohort until ML lands.
- **1.4** Stage-classification HogQL view: a derived column that tags every session with its current stage. Reused by every downstream dashboard.

**Exit criteria.** Director Overview dashboard (1312369) gains a "Journey health" section that reports the four metrics weekly.

### Phase 2 — HogQL attribution + journey scoring (weeks 7–10)

Still no external ML — just better SQL.

- **2.1** Build a basic Markov attribution table in HogQL (transition matrix, removal effect estimates). Output: revenue / leads credited per channel under Markov vs. last-click. Expect Meta and AIO to be undercounted; expect branded search to be overcounted.
- **2.2** Compute a "journey-quality score" per session: weighted sum of micro-conversion velocity, stage progression, and friction signals. Use it to rank live sessions for the agent For-You page.

**Exit criteria.** First "Markov vs. last-click" attribution report shared in a director meeting.

### Phase 3 — ML layer (weeks 11–20)

Now we need a Python environment. Decision needed in Phase 0 on which one.

- **3.1** Train LSTM next-event predictor; deploy as a PostHog person-property reverse-ETL.
- **3.2** Train HMM with hidden states ("curious", "comparing", "ready"). Surface state on Director Overview.
- **3.3** Transformer sentiment pipeline on whichever text source we managed to assemble (recommend starting with Google Business Profile reviews — easiest API).

**Exit criteria.** Real-time `dropoff_risk_score` per session usable as a feature-flag input to trigger overlays / agent outreach.

### Phase 4 — Graph + advanced (quarter 4+)

GNN on the touchpoint graph. Only if Phases 0–3 are paying off and we have CRM outcomes flowing. This is the "next year" frontier, not a current commitment.

---

## 8. Tools and connectors to request

Concrete asks, ranked by leverage.

| Priority | Ask | Unlocks | Effort to get |
|---|---|---|---|
| P0 | `lead_submitted` instrumentation across all forms | Real Stage 6 measurement, real attribution, pCLV becomes possible | Engineering ticket; small |
| P0 | CRM webhook → PostHog data warehouse | Closed-loop attribution, revenue per channel, pCLV training data | Depends on CRM owner; medium |
| P1 | Supermetrics renewal **or** PostHog Marketing Analytics activation | Google Ads, GA4, GSC, TikTok, LinkedIn data joined to web behavior | Procurement / config; small |
| P1 | Python ML environment (recommend a single Databricks workspace or AWS SageMaker Studio) | Markov, LSTM, HMM, Transformer training | Procurement + IT; medium |
| P2 | Google Business Profile API access | Transformer sentiment on real reviews | Small; just credentials |
| P2 | Trustpilot + Property Finder + Bayut review feeds | Multi-source brand sentiment | Per-vendor API agreements |
| P2 | TikTok + LinkedIn ad accounts connected (via Supermetrics) | Full multi-touch attribution beyond Meta + Google | Comes free once Supermetrics is renewed |
| P3 | Email platform (Mailchimp / Klaviyo / HubSpot) firing `$identify` on click | Newsletter touch in the journey graph | Engineering ticket; small |
| P3 | GPU instance OR API budget for Claude / OpenAI for transformer scoring | If we don't want to train our own | Small if API-only |

---

## 9. What I need from you to move this forward

To turn this guide into shipped work, I need decisions or info from you on:

1. **Who owns `lead_submitted`?** Front-end team? CMS team? Each form might be in a different codebase (main site, blog, off-plan microsites, Unbounce). Need a single owner who can audit and instrument all of them.
2. **Which CRM is the source of truth for closed deals?** And does Betterhomes already webhook events out of it? If not, who can build that webhook?
3. **What's the budget posture for Supermetrics renewal vs. PostHog Marketing Analytics?** Both cost money. Picking one unblocks Phase 0.
4. **Do we have a Python / ML environment already (e.g. someone using Colab, a Databricks instance, a data-science seat anywhere in the company)?** If yes, we use it. If no, we need to request one before Phase 3.
5. **Brand-voice constraints for any public output.** If we eventually publish a Betterhomes-version of the LinkedIn post (see Appendix), what we can and can't say about the underlying methodology and numbers.
6. **Confirmation of which agents / regions / property categories matter most.** The framework is generic; we should weight metrics by where Betterhomes actually makes money (rental commissions vs. sale commissions vs. off-plan vs. property management).

Send me answers to these and I'll start scoping Phase 0 tickets and the first version of the journey-health dashboard.

---

## 10. Appendix: LinkedIn post derivatives

When we're ready to put a Betterhomes flavor of the original post out — internally or externally — these are the angles that hold up:

- **"Why we stopped trusting last-click."** Concrete: a Markov attribution comparison showing X% of credit shifts from branded search to AI + content. Lands well with peers and property-tech operators.
- **"Six story beats of a Dubai property buyer."** The 6-stage journey above, with anonymized numbers. Lands well with prospective clients (sellers/landlords listing with us) — it signals sophistication.
- **"What an AI buyer's journey actually looks like."** Because Copilot/Bing leads our AI traffic, not ChatGPT — that's a genuinely non-obvious finding worth a post on its own.
- **"The metrics that replaced the funnel."** TTV, Micro-Conversion Velocity, Path-to-Purchase Complexity, pCLV — framed as "this is how we now run digital at Betterhomes."

Each of these is one short post, not a thesis. Keep the methodology high-level in public; keep the numbers internal until Phase 0 closes the conversion-tracking gap.

---

*Maintained by Digital Marketing. Branch: `claude/storytelling-marketing-framework-VEC1F`.*
