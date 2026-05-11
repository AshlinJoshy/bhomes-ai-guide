# Betterhomes — Canonical Event Spec

A reference document for the bhomes.com frontend engineering team.

This is the **canonical, minimal set of events** the digital marketing team needs fired from the bhomes.com client to make the storytelling marketing framework (see `README.md`) work end-to-end.

Status as of 2026-05-11: every custom event in PostHog stopped firing months ago (last Enquiry event 2025-11-03; last `similar_listing_cta_clicked` 2026-02-03). Only PostHog system events (`$pageview`, `$autocapture`, `$pageleave`, `$rageclick`, `$web_vitals`, `$exception`) are still flowing.

To bridge this, we've created 11 PostHog Actions that synthesize "events" from `$pageview` and `$autocapture` data (see Section 4 below). Those work today but are **proxies** — they catch button clicks, not form-success states; they catch page reaches, not actual engagement signals. For the journey framework to work properly, the events in Section 2 need to be code-fired from bhomes.com.

---

## 1. Conventions

- **Names** — `snake_case`. No prefixes. No spaces.
- **Properties** — `snake_case`.
- **Required global properties** for every event:
  - `language` — `"en"` or `"ar"` based on URL prefix
  - `page_type` — one of: `home | sale_listing | rent_listing | off_plan_listing | property_detail | area_guide | blog | market_report | contact | agent | other`
  - `property_id` — when on a property detail page
  - `area_slug` — when on an area guide or area-filtered listing page
- **UTM properties** — PostHog already captures `$utm_source`, `$utm_medium`, `$utm_campaign`, `$utm_content`, `$utm_term` automatically. Don't duplicate.
- **Fire on success, not attempt** — for forms, fire after the API call resolves successfully, not on submit click. (Existing `Attempt *` events double-count; we want a single truthful event per outcome.)

---

## 2. The canonical event list

Grouped by buyer journey stage.

### Stage 2 — Discovery

Stage 2 needs no new events. We rely on `$pageview` for first-touch + the AI/Google referrer Actions (Section 4).

### Stage 3 — Research

| Event | When it fires | Required properties | Optional properties |
|---|---|---|---|
| `property_search_run` | User runs a search on the property listing page (filter applied or new search initiated). | `category` (sale/rent/off_plan), `location_count`, `bedroom_min`, `bedroom_max`, `price_min`, `price_max` | `area_slugs[]`, `developer`, `furnished` |
| `property_filter_changed` | User changes a single filter without re-running search. | `filter_name`, `filter_value` | — |
| `property_photo_viewed` | Photo gallery opened on a property detail page. | `property_id`, `photo_index` | — |
| `property_map_viewed` | Map tab opened on a property detail page. | `property_id` | — |
| `area_guide_section_read` | User scrolls past 50% of an area guide page. | `area_slug` | `scroll_depth_pct` |
| `market_report_section_read` | User scrolls past 50% of a market report. | `report_slug` | `scroll_depth_pct` |

### Stage 4 — Shortlist

| Event | When it fires | Required properties | Optional properties |
|---|---|---|---|
| `property_favorited` | User favorites a property (logged in or session). | `property_id`, `category` | `from_page` |
| `property_unfavorited` | User unfavorites. | `property_id` | — |
| `property_compared_added` | User adds property to compare list. | `property_id` | — |
| `search_alert_created` | User saves a search and asks to be alerted. | `category`, `location_count`, `email_provided` | `search_id` |
| `property_shared` | User shares a property (WhatsApp / email / copy link). | `property_id`, `share_channel` | — |

### Stage 5 — Contact

| Event | When it fires | Required properties | Optional properties |
|---|---|---|---|
| `agent_contact_clicked` | User clicks phone / WhatsApp / email CTA on a property or agent page. | `property_id`, `agent_id`, `cta_channel` (`phone`/`whatsapp`/`email`) | `from_page` |
| `viewing_requested` | User submits a viewing request form (form success, not click). | `property_id`, `agent_id`, `viewing_date_requested`, `mode` (`in_person`/`virtual`) | — |
| `valuation_requested` | User submits a property valuation request form (form success). | `property_type`, `area_slug` | `bedrooms`, `condition` |

### Stage 6 — Conversion

| Event | When it fires | Required properties | Optional properties |
|---|---|---|---|
| **`lead_submitted`** | **Any form on the site submitted successfully and a lead record is created.** This is the unified conversion event. | `form_type` (one of: `contact`, `property_enquiry`, `off_plan_enquiry`, `valuation`, `viewing`, `mortgage`, `pmgt`, `careers`, `currency`, `conveyancing`, `newsletter`, `download_guide`, `list_property`), `lead_id` (from CRM if available), `email_provided` (bool), `phone_provided` (bool) | `property_id`, `area_slug`, `agent_id`, `enquiry_type`, `service_type` |

`lead_submitted` is the single highest-priority event. **Every form on bhomes.com that creates a CRM record must fire it**, in addition to whatever form-specific event it also fires.

---

## 3. JavaScript firing pattern

```javascript
// Standard pattern — fire after the API success callback, not on submit
async function submitContactForm(formData) {
  try {
    const response = await api.createLead(formData);

    posthog.capture('lead_submitted', {
      form_type: 'contact',
      lead_id: response.lead_id,
      email_provided: !!formData.email,
      phone_provided: !!formData.phone,
      property_id: formData.property_id ?? null,
      area_slug: formData.area_slug ?? null,
      agent_id: formData.agent_id ?? null,
      // form-specific event below
    });

    posthog.capture('viewing_requested', {
      property_id: formData.property_id,
      agent_id: formData.agent_id,
      viewing_date_requested: formData.date,
      mode: formData.mode,
    });
  } catch (e) {
    // Don't fire success events on error
    posthog.capture('lead_submission_failed', {
      form_type: 'contact',
      error: e.message,
    });
  }
}
```

Note: also fire `lead_submission_failed` when the API call errors. That's the error-tracking signal — we currently can't tell apart "users dropped off because they didn't fill the form" vs. "users hit submit but the form 500'd". This is its own data-quality unlock.

---

## 4. PostHog Actions — proxies that work today

These are server-side rules in PostHog itself that synthesize "events" from data still flowing (`$pageview` and `$autocapture`). They were created on 2026-05-11 and start working immediately. Use them in funnels and insights while engineering implements the code-fired events above.

| Action | PostHog ID | Synthesized from | Replaces (when code-fired event lands) |
|---|---:|---|---|
| `[Stage 2] AI-Referred Session` | 270067 | `$pageview` with referrer matching ChatGPT/OpenAI/Perplexity/Gemini/Claude/Copilot/Bing | n/a — stays useful |
| `[Stage 2] Google Organic Session` | 270068 | `$pageview` with referrer containing "google" but not "googleads" | n/a — stays useful |
| `[Stage 3] Property Detail Viewed` | 270069 | `$pageview` URL contains `/property/` | n/a — stays useful |
| `[Stage 3] Area Guide Viewed` | 270070 | `$pageview` URL contains `/area-guide` | n/a — stays useful |
| `[Stage 3] Market Report Viewed` | 270071 | `$pageview` URL contains `/market-report` | n/a — stays useful |
| `[Stage 3] Off-Plan Project Viewed` | 270072 | `$pageview` URL contains `/off-plan` or `/offplan` | n/a — stays useful |
| `[Stage 3] Sale Listing Viewed` | 270073 | `$pageview` URL contains `/for-sale` or `/buy` | n/a — stays useful |
| `[Stage 3] Rent Listing Viewed` | 270074 | `$pageview` URL contains `/for-rent` or `/rent` | n/a — stays useful |
| `[Stage 3] Blog Post Viewed` | 270075 | `$pageview` URL contains `/blog/` | n/a — stays useful |
| `[Stage 5] Contact Page Viewed` | 270076 | `$pageview` URL contains `/contact` | n/a — stays useful |
| `[Stage 5/6] Lead Submitted (proxy)` | 270077 | `$autocapture` clicks on Submit / Send / Register Now / Enquire Now / Get in Touch / Request a Callback buttons, or any `button[type="submit"]` | **`lead_submitted` (Section 2 Stage 6)** — once code-fired, this Action becomes the fallback / sanity check |

The existing useful Actions (preserve, don't touch):
- `All CTA Clicks - Phone, WhatsApp & Email` (id 255650) — sitewide via href matching (tel:/wa.me:/mailto:)
- `CTA - Phone Button Click` (id 255612), `CTA - WhatsApp Button Click` (id 255613), `CTA - Email Button Click` (id 255614) — property-page-scoped
- `Off-Plan Form - Register Now Submission` (id 255625)
- `Blog Submit Button Click` (id 260597)

To deprecate (fragile auto-generated selectors, redundant with the cleaner aria-label-based ones above):
- `CTA - Email 1`, `CTA - Email 2`, `CTA - Phone 1`, `CTA - Phone 2`, `CTA - Whatsapp 1`, `CTA - Whatsapp 2`
- `Subscribe`, `tests`, `test`
- `clicked Search`, `Homepage searches`

---

## 5. Migration plan

The legacy taxonomy (50+ `* Enquiry` events, viewing events, similar-listing events) doesn't need a new name — but **the events themselves need to be re-fired from current bhomes.com code**.

Two options:

**Option A (recommended) — clean start.** Engineer the events in Section 2 of this doc as the canonical list. Don't worry about restoring the legacy event names. Old dashboards referencing dead events get rebuilt against the new taxonomy. Cleanest long-term, ~2–3 sprints of work.

**Option B — restore legacy.** Audit which `posthog.capture('...')` calls existed in the old codebase, restore them as-is. Faster short-term, but we keep all the warts (e.g. duplicate `Apply for a job Enquiry` and `Attempt Apply for a job` events; `Download Our Investment Guide Enquiry` casing inconsistency).

Recommend Option A. The legacy data is not worth preserving — it stopped six months ago and the schemas were inconsistent.

---

## 6. Bot filtering (related, urgent)

Independent of the event spec, the PostHog project's `test_account_filters` currently only excludes `localhost`. This means dashboards include heavy bot traffic from:
- US Ashburn, VA (AWS datacenter cluster) — 211k pageviews / 108k "users" in 90 days
- Bare-China-no-city traffic — 60k pageviews / 60k 1:1-ratio "users"

Recommend adding to `test_account_filters` in the project settings:
- `$geoip_city_name` exact `Ashburn`
- `$geoip_country_name` exact `China` AND `$geoip_city_name` `is_not_set`
- Optionally: PostHog's built-in bot filter (account settings → Filter out internal and test users)

This is a 5-minute change in PostHog UI and immediately cleans up every dashboard.

---

*Owned by Digital Marketing. Implementation by Engineering. Branch: `claude/storytelling-marketing-framework-VEC1F`.*
