# 🏡 Lincoln House Hunting

> 🔒 **Showcase repository.** The source code lives in a private repo; this page explains what the
> app does and how it is built. Code walkthrough available on request — [LinkedIn](https://www.linkedin.com/in/acsamiloglu/).

A personal, local-first web app for hunting **houses for sale** in and around Lincoln, UK. It pulls
listings from the big UK property portals into one place, merges the same house listed on several
portals, enriches each listing with official energy and sale-price data, and lets you filter,
shortlist and **ask questions about the listings in plain English**.

**Why:** the portals each show a slice of the market, repeat the same houses, and hide the numbers
that matter (floor area, £/m², energy rating, what the house last sold for). This app answers
"what's actually worth viewing?" in one screen.

---

## ✨ Features

### One feed from three portals
- Listings from **Rightmove, Zoopla and OnTheMarket**, fetched via Apify actors (never scraped directly).
- **Cross-portal de-duplication** — the same property on two portals becomes one listing with a link to each.
- Change tracking on every refresh: **NEW**, **price changed** (with price history) and **removed** badges.
- Alternative offline path: export a dataset as CSV / JSON and import it through a column mapper that
  auto-detects fields and remembers your mapping.

### Data the portals don't show
- **EPC enrichment** from the GOV.UK energy-performance register: energy band and **floor area (m²)**,
  matched by address within the postcode with a high / low match-confidence flag.
- **£/m²** computed for every listing with a known floor area.
- **Land Registry Price Paid** lookups — what the property last sold for.
- Listing data always wins; enrichment only fills gaps.

### Portal-style search
- **Location autocomplete**: type a town, village, outcode or postcode ("wash" → Washingborough,
  "LN6 7" → LN6 7AA …) and pick a radius, just like on the portals.
- 20+ combinable filters: price, beds / baths, property type, tenure, minimum EPC, m² range,
  max £/m², build period, council-tax band, garden, parking, chain-free, new build, listing age,
  newly added, price dropped, source, status and free-text search.
- Filters live in the URL — every search is bookmarkable; **saved searches** by name.
- **Table** (sortable by price, m², £/m²), **card** and **map** (Leaflet / OpenStreetMap) views.
- **Favourites** with private per-listing notes.

### Ask the listings — LLM with verified answers
Type a question like *"3-bed with a south-facing garden near a school, under £250k, no chain"*:

1. The LLM splits it into **hard filters** (applied in SQL) and **soft criteria** (judged from each description).
2. Candidates are evaluated in small batches.
3. Every match carries a **verbatim quote** from the listing that is **verified to exist in the text** —
   anything unverifiable is flagged, and criteria the listing doesn't mention are shown as such.
4. Results are cached per question + listing; a footer shows tokens used and estimated cost.
5. One click turns the objective part of the question into a normal, saved filter set.

---

## 🏗️ Architecture

```
            ┌──────────── Next.js (App Router) — UI + API routes ────────────┐
 Browser ── │  Filters · Table / Cards / Map · Ask box · Import dialog        │
            └───────┬───────────────┬────────────────┬───────────────┬───────┘
                    │               │                │               │
           Provider layer     Enrichment       LLM layer       Location search
         (one per portal)   (EPC + Land Reg.)  (Gemini/Claude)  (postcodes.io + Photon)
                    │               │                │
                Apify API     GOV.UK EPC API    structured JSON
                    │               │            output + quote
                    ▼               ▼            verification
            normalise → de-duplicate → diff (new / price / removed)
                                    │
                              SQLite (local)
```

- **Pluggable providers** — each portal implements one interface (`fetchListings` for live data,
  `parseImport` for files); adding a portal is one module plus a registry entry.
- **Graceful degradation** — every external service is optional. Missing keys disable just that
  feature (with a tooltip naming what's missing); outages are reported, never crash the app.
- **Cost controls** — per-run result caps on the paid scrapers, a warning if you refresh twice in an
  hour, cheap default LLM model, and per-question token / cost reporting.
- **Local-only** — the database, caches and keys stay on the machine.

## 🛠️ Tech stack

| Area | Tools |
|---|---|
| App | Next.js (App Router), React, TypeScript, Tailwind CSS |
| Data | SQLite (better-sqlite3) |
| Maps | Leaflet + OpenStreetMap |
| Listings | Apify Store actors for Rightmove / Zoopla / OnTheMarket |
| Enrichment | GOV.UK EPC API, HM Land Registry Price Paid (open data) |
| Geo | postcodes.io, Photon (OSM) — free, no key |
| LLM | Google Gemini (default) or Anthropic Claude, structured JSON output |

---

<sub>Personal project for my own house search. Respects the portals' and data providers' terms:
low refresh frequency, no redistribution of listing data.</sub>
