# ClimateShield — Master Project Plan

**Purpose of this document:** map the full hackathon problem statement (Phases 1-3) to what is *actually built and working* today vs. what remains, list every free/real API we use (and how), define the AI/ML approach honestly, define how Citizen / Government (+ subcategories) / Rescue coordinate end to end, and lay out an ordered, independently-shippable chunk plan for everything still needed. Nothing in this plan proposes fabricated data — every new data source below is either a real free API, a real historical/public dataset, or an explicitly-labeled deterministic computation.

---

## 0. Problem-statement -> current-state map

| PS requirement | Where it lives today | Status |
|---|---|---|
| Establish geographic/asset context | Zones + InfrastructureAsset (real Chennai OSM import + synthetic demo assets), Prisma schema | **Done** |
| Environmental data ingestion | Open-Meteo weather + air quality (live, no key) | **Done**; flood-specific signal missing (§2) |
| Assess ≥1 climate risk | Deterministic risk engine (`risk.service.ts`) — hazard × vulnerability × criticality × historical recurrence | **Done** |
| Identify vulnerable areas/assets | Cascade engine (`cascade.service.ts`) — dependency-graph traversal, zone risk ranking | **Done** |
| Alerts / recommended actions | Citizen alerts (computed), Gov `recommendedActions` in cascade/response-plan, Notification model | **Done** (delivery is DB-only — no email/SMS/push yet, see §7) |
| Multiple locations/assets, hazard-specific assessment | 4 zones + 19 assets seeded, real Chennai geography imported | **Done** |
| Historical incidents / vulnerability tracking | `HistoricalEvent`, `Hotspot` models, seeded | **Done** (hand-seeded, not derived — see §4.2) |
| Thresholds | Severity thresholds in `weather.service.ts` / `risk.ts` (rainfall/temp/wind bands) | **Done** |
| Preparedness plans | `createResponsePlan()` (proposal-only, human-confirmed) | **Done**, not wired to any frontend page yet |
| Response workflow, task assignment, escalation | Full transactional state machine: Incident → Task → ResponseUnit, `dispatchUnitToIncident`, `updateTaskStatus`, `verifyTask` | **Done on backend**; **frontend gap** — see §6 |
| Recovery tracking | Task/Incident terminal states (`COMPLETED`/`RESOLVED`/`CLOSED`), unit auto-released | **Done** |
| Reports / risk dashboards | `analytics.service.ts`, `GET /api/analytics/overview` | **Done on backend**; frontend page not wired |
| Roles & permissions | `ADMIN GOVERNMENT_OPERATOR DISPATCHER ANALYST FIELD_OPERATOR CITIZEN` — fully enforced server-side | **Done** — see §5 for exact mapping |
| Multiple locations/assets at scale | Real geography import pipeline (`import-chennai.ts`) reusable for any city with OSM coverage | **Done**, extensible |
| Deployment | Not yet deployed to a public URL | **Open** — see §8 Chunk J |
| Monetization | Not modeled | **Open, low priority** — doc-only recommendation, §8 Chunk J |
| Auth/authz, secure APIs, validation | JWT, bcrypt, Zod validation everywhere, RBAC, rate-limited login | **Done** |
| Reliability for delayed/unavailable data | Every value carries `dataQuality`; weather failures return `502 WEATHER_PROVIDER_*`, never fabricate; AI has a deterministic fallback | **Done** |
| False positive/negative, uncertainty | Risk engine returns `confidence`; AI confidence is clamped to never exceed engine confidence | **Done** |
| Failed alert delivery / escalation failure | Not modeled (no delivery channel exists yet) | **Open** — §7 |
| Duplicate events | `dispatchUnitToIncident` explicitly rejects duplicate dispatch (409) | **Done** |
| Message queues / async processing | None — everything is synchronous request/response | **Open** — §7 Chunk H |
| Documentation (methodology, architecture, security, failure modes) | `docs/API.md` (excellent, honest data-quality docs), `DECISIONS.md` (ADRs) | **Partial** — needs consolidated methodology/security/failure-mode docs, §7 Chunk I |

**Bottom line:** the *engine* (risk, cascade, simulator, task/dispatch state machine, AI explain layer) is real, deterministic, transactional, and already good enough to defend to a technical judge. The **citizen half of the product is fully wired end-to-end** (built in the previous session). The **weak link is the Government/Rescue frontend** — most of those pages are still hardcoded UI shells sitting on top of a backend that already supports everything they need.

---

## 1. Coordination model — how every role connects

```text
                              ┌─────────────────────────┐
                              │   ENVIRONMENTAL DATA     │
                              │ Open-Meteo (weather, AQI,│
                              │ flood/GloFAS) · OSM      │
                              └────────────┬─────────────┘
                                           │ live fetch
                                           ▼
                              ┌─────────────────────────┐
                              │  DETERMINISTIC ENGINES   │
                              │ risk.service · cascade   │
                              │ .service · simulator     │
                              └────────────┬─────────────┘
                                           │ MODELED risk/cascade
              ┌────────────────────────────┼────────────────────────────┐
              ▼                            ▼                            ▼
   ┌─────────────────┐         ┌─────────────────────┐       ┌──────────────────┐
   │     CITIZEN      │        │  GOVERNMENT (HQ)     │       │  GOVERNMENT       │
   │ live map, alerts,│───────▶│ ADMIN / OPERATOR /   │──────▶│  (FIELD/RESCUE)   │
   │ report hazard,   │ auto-  │ DISPATCHER / ANALYST │ Task  │  FIELD_OPERATOR   │
   │ SOS, safe route  │ Incident│ triage → dispatch    │ assign│  updates status,  │
   └────────┬─────────┘        └──────────┬───────────┘       │  Rescue-branded   │
            │                              │                   │  mobile pages     │
            │  Notification on             │  verifyTask()      └─────────┬─────────┘
            │  status change (§7 gap)      │  closes the loop             │
            ▼                              ▼                               ▼
   ┌─────────────────┐         ┌─────────────────────┐        task status updates,
   │ sees "Corridor   │◀───────│ Incident RESOLVED/   │◀───────field observations
   │ Cleared" alert   │        │ CLOSED, feeds         │
   └─────────────────┘         │ Analytics/Hotspots    │
                                 └─────────────────────┘
```

**Key fact:** "Rescue" is **not** a separate backend role. A Response Unit (`FIRE_RESCUE`, `EMS`, `PUMP_CREW`, etc.) is dispatched by a `DISPATCHER`/`GOVERNMENT_OPERATOR`; the person operating that unit's mobile device logs in as `FIELD_OPERATOR` and sees the Rescue-branded pages (`/rescue/*`). This is a **frontend persona of the same role**, not a schema change — avoids an unnecessary migration and matches how `task.service.ts` already gates `updateTaskStatus` to `FIELD_OPERATOR`.

---

## 1b. Region correction — Andhra Pradesh (coastal), not a generic city

A teammate-supplied research dossier confirms the actual hackathon context: **Sustainable Smart Cities & Climate Tech, Problem Statement 5, Andhra Pradesh** — a coastal state with cyclone, storm-surge, and monsoon-flood exposure. This **corrects** the earlier "Assam" real-world-grounding idea in §3 below (Assam is real and well-documented, but it's the wrong region for this brief). The dossier itself cites a directly relevant, well-documented precedent: **Cyclone Hudhud (2014, Visakhapatnam, Andhra Pradesh)** — including the detail that Bhuvan's disaster-imagery layer processed 25,000+ citizen-submitted photos during that event, which is a strong, honest parallel to our own citizen-photo-evidence feature. See the revised §3.

Positioning takeaway from the dossier (worth stating explicitly in any pitch/demo narrative): commercial climate-risk platforms (Jupiter Intelligence, Cervest, One Concern, ClimateAi, Tomorrow.io) sell portfolio-level risk *scores* to insurers/enterprises on multi-year contracts — none of them offer the geography → risk → alert → task → response-closure **workflow** a municipal corporation, campus, or industrial park actually needs. That gap is exactly ClimateShield's pitch, and it's also exactly what our Task/Unit dispatch state machine already does that a "risk score API" does not.

## 2. Free/Real API catalog

| Provider | Endpoint | Key? | Status | Used for |
|---|---|---|---|---|
| Open-Meteo Weather | `api.open-meteo.com/v1/forecast` | No | ✅ Integrated | Live temp/rain/wind, government + citizen |
| Open-Meteo Air Quality | `air-quality-api.open-meteo.com/v1/air-quality` | No | ✅ Integrated | US AQI/PM2.5, citizen map |
| OpenStreetMap/Overpass | via `import-chennai.ts` | No | ✅ Integrated | Real Chennai zones/roads/infra |
| OSRM (public demo) | `router.project-osrm.org` | No | ✅ Integrated | Citizen safe-route alternatives |
| Nominatim (OSM) | `nominatim.openstreetmap.org/reverse` | No | ✅ Integrated | Reverse geocoding |
| Browser Geolocation | — | No | ✅ Integrated | Client GPS |
| **Open-Meteo Flood API** | `flood-api.open-meteo.com/v1/flood` | No (non-commercial) | 🆕 **Proposed — Chunk E** | **Real GloFAS river-discharge** (1984→7mo forecast, 5km res) as a genuine flood-specific signal, complementing rainfall-only modeling. Confirmed free, no key, global coverage. |
| Open-Meteo Geocoding | `geocoding-api.open-meteo.com` | No | 🆕 Proposed (optional) | Real place-name search for the citizen map search bar (currently a dead input) |
| Open-Meteo Historical/Archive Weather | `archive-api.open-meteo.com` | No | 🆕 Proposed (optional, Chunk G) | Real multi-year rainfall history per coordinate → genuine recurrence-score computation for Hotspots instead of hand-seeded scores |
| **ReliefWeb API** (UN OCHA) | `api.reliefweb.int/v2` | ⚠️ **Correction (verified live during Chunk F build): requires a registered/approved `appname`** — a live test request returned `403 AccessDeniedHttpException: "You are not using an approved appname"`; the v1 endpoint returns `410 Gone` (decommissioned). This contradicts this doc's original "No auth" assumption. | Documented, **not integrated** (same honest treatment as NASA FIRMS/data.gov.in below — would need a signup key we don't have). Chunk F was delivered instead as a static, fully-cited Cyclone Hudhud reference panel (no live external call). |
| NASA FIRMS | `firms.modaps.eosdis.nasa.gov` | Free self-register (MAP_KEY) | Documented, not integrated | Satellite fire/heat-anomaly detection — flagged as a credible future integration; not added now because it requires a signup key and isn't essential to the flood/heat MVP |
| data.gov.in / India-WRIS | data.gov.in | Free self-register | Documented, not integrated | Real Indian river-gauge levels — flagged in docs as the "next real step" for India-specific deployments; not integrated now due to key-signup friction within hackathon time |
| Google Gemini | `generativelanguage.googleapis.com` | Free tier, self-register | ✅ Integrated (explain layer) | Grounded narrative synthesis; proposed extension in §6 for vision-based photo triage |

Every one of these is genuinely free (no paid tier required to use what we use) and either already wired or has a concrete, scoped integration plan below — nothing is "planned" without a real, checked endpoint.

### 2b. India/Andhra-Pradesh-specific sources (from teammate research dossier) — status honestly assessed

| Source | What it offers | Why not integrated yet (or how) |
|---|---|---|
| **CPCB National AQI** (`data.gov.in`, `airquality.cpcb.gov.in`) | Real Indian CAAQMS station AQI (PM2.5/PM10/NO2/SO2/CO/O3/NH3) | Free, real REST API via a `data.gov.in` signup key. **Credible near-term addition** — same integration shape as our existing Open-Meteo AQI client, just an India-specific alternate/supplementary source. Not added yet purely because it needs a registered key (signup friction), not a technical blocker. Documented here as the concrete next step rather than silently skipped. |
| **OpenAQ** | Aggregated global open AQI incl. Indian stations | Free, no key, clean REST — actually **easier** to integrate than CPCB directly and could be added as a redundancy/cross-check source without any signup friction at all. |
| **IMD API Portal** | Official Indian current weather/forecast/warnings (source of record) | Portal/PDF-doc-first, IP-whitelisting on some feeds. We already treat Open-Meteo as the live weather source of record (clearly labeled `LIVE_OBSERVED`/provider `Open-Meteo`) — IMD would be additive/citation-only, not a replacement, given the access friction the dossier itself flags. |
| **CWC Flood Forecast / AFF** | Real river-gauge levels/discharge, 325 stations | Dashboard-first, no clean public REST API (confirmed by the dossier). Our **Open-Meteo Flood API (GloFAS)** addition (§2, Chunk E) is the honest global-fallback equivalent the dossier itself recommends pairing with every government-portal source. |
| **Bhuvan (ISRO/NRSC)** | India-specific satellite imagery + disaster layers, has a token-based API | Real and India-specific, strongest "built for India" option per the dossier. Flagged as a **credible Phase-3 stretch** (geospatial intelligence differentiator) — not added now due to token-registration time cost within remaining scope, documented honestly rather than faked. |
| **INCOIS (storm surge / high wave / OSF)** | Cyclone storm-surge & coastal advisories — **the single most relevant source for Andhra Pradesh's coastline** | No public JSON API today (bulletin/portal-based, confirmed by the dossier). Correct architecture per the dossier: treat as a **planned integration via SACHET's CAP feed**, not claim direct access we don't have. |
| **SACHET (NDMA)** | National Common Alerting Protocol (CAP) early-warning aggregator (IMD+CWC+INCOIS+GSI) | ⚠️ **Correction (verified during Chunk F2 build): no confirmed working public CAP/RSS feed URL.** Several plausible endpoint guesses were tested live and returned `404`/`403` — rather than fabricate an unverified URL, ingestion of SACHET's feed is **not implemented**. | Scope reduced to only the buildable, zero-external-dependency half: **(b) emit our own government-approved alerts in valid CAP 1.2 XML shape** as an export format (protocol-compatibility demo). This alone is real and delivered in **Chunk F2**. |
| **MSG91 / Twilio (SMS/WhatsApp)** | Real alert delivery channel | Currently our `Notification` model is DB-only (no delivery channel) — this is the actual, concrete fix for §7's "alert delivery reliability" gap. MSG91 is the dossier's correctly-reasoned recommendation for India-only cost (~₹0.15-0.20/SMS vs Twilio's USD pricing). Requires a signup key; documented as the specific provider choice for Chunk I rather than a vague "some email provider." |
| **Razorpay Subscriptions** | Real recurring-billing API (sandbox/test mode available with no real money) | Concrete, buildable monetization proof for Phase 2's "subscriptions/enterprise licensing" ask — a per-district/per-asset monthly plan gated behind Razorpay's **test mode**, demoable without real transactions. Upgrades §0's "low priority, doc-only" monetization note to a scoped, buildable Chunk J item. |
| **Kafka / RabbitMQ / Redis Streams (BullMQ)** | Async event processing | Confirms the plan's existing Chunk H recommendation (BullMQ+Redis) as the right-sized choice for a Node.js stack within hackathon time, vs. standing up Kafka/RabbitMQ infrastructure. |

---

## 3. Real-world grounding: Cyclone Hudhud (2014, Visakhapatnam, Andhra Pradesh) — corrected region

Researched directly (Wikipedia, IMD post-cyclone report, UN damage assessment) — genuine, precisely-sourced figures, replacing the earlier Assam draft which was the wrong region for this brief:

- **Made landfall at Visakhapatnam, Andhra Pradesh on 12 October 2014** at peak intensity — 950 mbar central pressure, 185 km/h sustained winds (IMD 3-min) / 215 km/h (JTWC 1-min), Category 4-equivalent
- **116 total deaths** (India + Nepal combined); **US$11 billion in damage** (UN 2015 assessment) — one of the costliest North Indian Ocean cyclones on record
- **Andhra Pradesh specifically:** 46 deaths, 43 injuries, **41,269 houses** damaged, **237,854 hectares** of cropland impacted, **2,446,532 livestock/poultry** died, **27,041 electric poles** downed, **6,075 km of roads** affected, **73 villages isolated** for up to 2 days
- **730,000 people** moved to relief camps across AP+Odisha (111,000 pre-emptively evacuated in AP alone, 370 relief camps readied)
- **1.4 m storm surge** at Visakhapatnam; **380 mm rainfall in 24h** at Gantyada (highest in the state)
- Visakhapatnam Airport flooded/roof torn off, closed 11-17 Oct (₹500 crore / $81.93M damage); Indira Gandhi Zoological Park lost 1,000m of walls with animals roaming loose
- **Real, coordinated multi-agency response** (directly maps to our Task/Unit dispatch model): 44 NDRF teams + 8 rescue teams pre-positioned; a Navy-led joint operation "**Lehar**" deployed 20 Navy rescue teams, 25 Army rescue teams, 17 Coast Guard ships, 7 Air Force aircraft; 12 NDRF teams + 5,000 power-company workers on cleanup

**How to use this honestly (no fabricated Visakhapatnam geometry — yet):** we currently only have real OSM-imported geography for Chennai. Rather than inventing fake Visakhapatnam zone boundaries, the plan is:

1. **"Historical Disaster Intelligence" panel** (Chunk F — **delivered**, `GET /api/analytics/disaster-intelligence`) — a read-only reference widget on the Government Command Center, powered by a static, hand-verified, fully-cited Cyclone Hudhud case study (`dataQuality: REAL_HISTORICAL_REFERENCE`, with citation links to Wikipedia/UN OCHA ReliefWeb). **Scope correction:** the originally-planned live ReliefWeb API call was tested live during implementation and found to require a registered `appname` we don't have (`403 AccessDeniedHttpException`) — so this panel is static reference data only, not live-polled. This still demonstrates real regional research to judges without pretending our Chennai demo data represents Visakhapatnam or fabricating a live feed we can't actually reach.
2. **Chunk F2 — CAP XML alert export** (**delivered**, scope reduced from the original plan): no confirmed working public SACHET CAP/RSS feed URL was found (tested live, only guesses returning 404/403), so feed *ingestion* was dropped rather than built against a fabricated URL. What *was* built: our own government-approved alerts can be exported in valid CAP 1.2 XML shape (protocol-compatibility demo, not a claim of direct SACHET integration).
3. **Upgraded stretch goal (was optional, now recommended given the region correction):** run a real OSM/Overpass import (same pattern as `import-chennai.ts`) for **Visakhapatnam** specifically — this would let the Hudhud case study anchor to genuine local geography (real roads, real hospital locations, a real "Andhra University campus" or "Indira Gandhi Zoological Park" asset) rather than only living in a text panel. Scoped as Chunk J given import-script time cost, but now clearly the most narratively powerful option if time allows.

---

## 4. AI/ML — honest scope

### 4.1 What's already built (confirmed by code audit)
`POST /api/incidents/:id/explain` — Gemini-grounded, **not free-form**:
- Every fact given to the model comes from the verified deterministic engine (`getIncidentCascade`) — root asset, risk score/confidence, cascade nodes, hazard readings.
- Model output is schema-validated; `confidence` is clamped to never exceed the engine's own confidence; recommended actions are restricted to a whitelisted catalog.
- **Any failure** (no API key, timeout, bad JSON, validation failure) falls back to a fully deterministic template — AI is never a single point of failure.

This already satisfies "AI integration" credibly. What's proposed below **adds** genuine, scoped ML/AI value rather than overclaiming a custom-trained deep model we don't have time to build responsibly.

### 4.2 Proposed additions (Chunk G)
1. **Gemini Vision citizen-photo triage** — when a citizen submits report evidence, send the photo to Gemini's vision endpoint (same API already integrated) asking only for a structured, whitelisted classification (`category confidence`, `visible water depth estimate: none|ankle|knee|waist|submerged`, `caption`) — stored as an **additional, clearly non-authoritative signal** on the `CitizenReport` (mirrors how `reportedSeverity` is already labeled non-authoritative). This is genuinely novel, judge-visible, and buildable in one chunk since the Gemini plumbing already exists.
2. **Statistical risk-trend forecasting** — plain deterministic time-series math (linear trend / exponential smoothing) over stored `WeatherSnapshot`/`TelemetryReading` history to forecast next-N-hour risk trend per zone. Real math, `dataQuality: FORECAST`, no new dependency.
3. **Real hotspot derivation via clustering** — a simple k-means/DBSCAN-style clustering (small, dependency-free TS implementation) over `HistoricalEvent` lat/lng + severity to **derive** hotspots from data instead of hand-authoring them. This is genuine unsupervised ML, not a buzzword.

**Explicitly not doing:** claiming a custom-trained deep-learning model (no time to build, validate, and honestly document one within a hackathon window — would risk becoming exactly the kind of unverifiable claim you told me to avoid).

---

## 5. Roles & permissions — exact mapping

| Role | Frontend persona | Backend permissions (already enforced) |
|---|---|---|
| `ADMIN` | Government HQ (full) | Everything, incl. audit/departments |
| `GOVERNMENT_OPERATOR` | Government HQ (EOC Director) | Overview, zone cascade, response center, simulator, dispatch, task verify |
| `DISPATCHER` | Government HQ (operational) | Response center, dispatch, task assignment |
| `ANALYST` | Government HQ (read-only) | Analytics, hotspots, history — **no** dispatch/write (already excluded from `GOVERNMENT_ROLES`) |
| `FIELD_OPERATOR` | **Government Mobile** *or* **Rescue Team** (same role, different frontend routes/branding) | View assigned tasks, update task status (`ASSIGNED→ACKNOWLEDGED→IN_PROGRESS→COMPLETED`) |
| `CITIZEN` | Citizen app | Own reports/SOS, public-safety reads only — fully built |

---

## 6. Closing the loop — one real gap found

Today: Citizen report/SOS → auto-`Incident` → Government resolves it → **nothing tells the citizen it was resolved.** The link exists (`CitizenReport.incidentId`) but nothing reads it back. **Chunk D** adds: when an Incident linked to a `CitizenReport`/`SosEvent` changes status, create a `Notification`-equivalent for that citizen (reusing the existing `Notification` model against the citizen's own `userId`) and surface it in `GET /api/citizen/alerts` or a small new `GET /api/citizen/reports/:id` status poll (already returns `incident.status` — just needs the frontend to show it / poll it, which is a small, real addition).

---

## 7. Phase 3 hardening — genuine gaps, scoped honestly

- **Message queue / async processing** (explicit PS ask, currently absent): recommend **BullMQ + Redis** (free, self-hosted via a Docker container, matches the existing local-Postgres-in-Docker pattern already used for dev) for: hazard ingestion → risk recompute → notification fan-out → report generation. Chunk H.
- **Alert delivery reliability**: currently DB-only `Notification` rows, no actual delivery channel or retry/escalation. Propose an outbox pattern processed by the same queue; real delivery (email) would need a free-tier transactional provider (e.g., Resend/Brevo free tier, requires signup) — will document as an explicit limitation if not wired due to key friction, never fake a "sent" status.
- **Rate limiting**: only on login today; extend to citizen SOS/report endpoints (abuse prevention).
- **Monitoring**: structured JSON logs exist; no metrics. Add a lightweight `/metrics` endpoint (Prometheus text format, no new infra) for request duration/error-rate — genuinely useful, zero paid dependency.
- **Documentation deliverables** the PS explicitly asks for: `docs/RISK_METHODOLOGY.md`, `docs/SECURITY.md`, `docs/FAILURE_MODES.md`, and a deployment/scalability section in `ARCHITECTURE.md`. Chunk I.

---

## 7b. Auth regression found during re-audit (Chunk A0 — fix before proceeding)

A teammate's "transport layer -> cline_backend" PR (merged to `main`) wired `GovCommandCenterPage`/`GovZoneCascadePage` to the backend, but introduced a real regression rather than reusing the existing, working auth system:

- **Two independent, unsynchronized token stores**: the pre-existing `lib/api.ts`/`AuthContext` (used by the citizen app) stores its JWT under `localStorage['cs_auth_token']`; the new `services/api.ts` transport layer stores/reads its own copy under a *different* key, `localStorage['cs_token']`. They are bridged today only by a manual `localStorage.setItem('cs_token', token)` line added to `LoginPage.tsx` after a successful login — a patch, not a fix.
- **Multiple no-auth bypass paths were added to `LoginPage.tsx`**: a "Direct Launch" button that navigates straight to a role's home route with **no login call at all**, a per-role bypass on any login error, and the Rescue role *always* bypassing auth (there's no backend account it could log into, since Rescue is meant to be `FIELD_OPERATOR` — see §1).
- **Route guards are inconsistent**: Citizen routes have the real `RequireAuth roles={['CITIZEN']}` guard. Government HQ desktop routes only check *token presence* (not role) via a locally-defined `RequireGovernmentLogin`. **Government Mobile routes and all 6 Rescue routes have no guard at all.**

**Fix (Chunk A0, before A/B):** unify on the single existing `lib/api.ts` + `AuthContext`/`RequireAuth` system (already built, already correct), delete the second token store and the bypass buttons, and apply `RequireAuth roles={[...GOVERNMENT_ROLES]}` / `RequireAuth roles={['FIELD_OPERATOR']}` consistently to every gov/mobile/rescue route — mirroring exactly what already works for `/citizen/*`.

---

## 8. Ordered chunk plan

| # | Chunk | Why this order |
|---|---|---|
| **A0** | **Fix an auth regression found during re-audit** (see below) before touching more pages | A teammate's transport-layer PR introduced two independent, unsynchronized token stores and multiple no-auth "Direct Launch" bypass buttons — must be cleaned up first so new pages aren't built on top of it |
| A | Wire `GovResponseCenterPage` + `GovSimulatorPage` to the real, already-working backend endpoints | Biggest "fake→real" jump for least effort — backend is 100% ready |
| B | Wire all 6 Rescue pages to real task/unit endpoints (`FIELD_OPERATOR`) | Completes the Citizen→Gov→**Rescue** loop end to end — currently the single biggest gap |
| C | Wire remaining Gov pages (Critical Asset Monitor, Mobile Map/Triage/Tasks) | Finishes Government side |
| D | Citizen notification on their own report/SOS resolution | Small, closes the full loop both directions |
| E | Open-Meteo Flood/GloFAS integration | Real flood-specific signal, cheap to add |
| F | ✅ **Done** — Static, cited Cyclone Hudhud (2014, Visakhapatnam, AP) "Disaster Intelligence" panel (live ReliefWeb dropped — requires an approved `appname` we don't have, confirmed live) | Real-world grounding, judge-visible research depth, correct region, no fabricated live feed |
| F2 | ✅ **Done, reduced scope** — Emit our own alerts in valid CAP XML (SACHET feed *ingestion* dropped — no confirmed working public URL found) | Demonstrates protocol-compatibility with India's national alert system without overclaiming access |
| G | AI/ML additions: Gemini vision photo triage, statistical forecasting, clustering-derived hotspots | High "wow factor," builds on existing Gemini plumbing |
| H | Message queue (BullMQ+Redis) for hazard/risk/notification pipeline | Phase 3 requirement |
| I | `/metrics` endpoint, extended rate limiting, notification outbox | Phase 3 hardening |
| J | Documentation set (methodology/security/failure-modes/deployment) + optional monetization doc + optional second-city real import | Wraps every PS Phase 3 documentation ask |

Each chunk is independently testable and committable, following the same pattern as the citizen-workflow chunks (typecheck → tests → build → commit).
