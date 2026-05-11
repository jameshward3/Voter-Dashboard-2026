# Vote-by-Mail Operations Dashboard — Design Document

**Jurisdiction:** Orange, NJ — South Ward (2026 Municipal)
**Source file analyzed:** `South ward mailins - 2026.xlsx` (638 records, 28 columns, mailed 3/19/2026; 252 returned / 386 outstanding as of 5/11/2026; 39.5% return rate)

This document covers the recommended schema, dashboard layout, cleaning and aggregation pseudocode, React component structure, chart/map logic, the privacy-safe display approach, and the auto-generated insights the dashboard should produce.

---

## 1. Guiding privacy and ethics principles

The dashboard is an **operations and transparency** tool — not a political intelligence tool. The following constraints are enforced at the **data layer**, not at the UI layer, so they cannot be circumvented by toggling views.

1. The dashboard never infers or displays how an individual voted. Party affiliation is preserved in the admin directory for follow-up operations (e.g., reminder mailings) but is **excluded from every aggregate query and every public view**.
2. Names and addresses appear only in the private/admin Directory tab, which is gated behind authenticated admin role.
3. Public-facing map and chart views always aggregate to precinct, ward, census block group, neighborhood, or grid cell.
4. **Minimum cell size = 10.** Any geography with fewer than 10 records has its counts suppressed (rendered as a hashed/gray tile labeled "< 10 — suppressed for privacy"). The threshold is configurable in one place.
5. No predictive scoring, vote-likelihood model, or partisan inference is computed anywhere in the pipeline.
6. Exports (CSV, PDF) honor the same suppression rules. The admin-only record-level CSV requires elevated role.

---

## 2. Recommended data schema

Two logical tables. The pipeline ingests the state extract, normalizes it to `voter_record`, and emits `area_aggregate` for the public surfaces.

### 2.1 `voter_record` (admin/private — never published)

| Field | Type | Source column | Notes |
| --- | --- | --- | --- |
| `voter_id` | string PK | `ID` | State-issued voter ID |
| `last_name`, `first_name`, `middle`, `suffix` | string | as-is | Admin Directory only |
| `party` | enum | `Party` | DEM/REP/UNA/etc. **Admin only.** Never aggregated |
| `residence_street_no`, `residence_street`, `apt` | string | `Street No.`, `Street Name`, `APT/UNIT` | Street is canonicalized (uppercase, AVENUE→AVE, etc.) |
| `residence_city`, `residence_state`, `residence_zip` | string | as-is | |
| `mailing_address`, `mailing_city`, `mailing_state`, `mailing_zip`, `mailing_country` | string | as-is | Detects ballots mailed out of jurisdiction |
| `ward` | string | derived/joined | South in this file |
| `precinct` | string | joined from state voter file | Not present in this extract — join required |
| `census_block_group` | string | derived from geocode | For ACS denominators if needed |
| `lat`, `lon` | float | geocoded | Census Geocoder primary, Mapbox fallback |
| `ballot_request_type` | enum | `Ballot Req Type` | AFE = Apply For Every election; SNG = Single election |
| `ballot_voter_type` | enum | `Ballot Req Voter Type` | Regular Civilian / UOCAVA / etc. |
| `ballot_transmission` | enum | `Ballot Req Trans Type` | ML = mail, EM = email, etc. |
| `request_received_date` | date | `Ballot Req Rcvd Date` | When the county received the application |
| `ballot_sent_date` | date | `Ballot Mailed Date` | When the county mailed the ballot |
| `ballot_received_date` | date | `Ballot Received Date` | When the county received the returned ballot |
| `status` | enum | `Ballot Counted Status` | `Received`, `Outstanding`, `Rejected`, `Cured` |
| `days_to_return` | int | derived | `received_date - sent_date` |
| `election` | string | `Election` | Allows multi-election dashboards |

### 2.2 `area_aggregate` (publishable, privacy-safe)

One row per (`area_key`, `geography_level`, `as_of_date`). `area_key` is the precinct/ward/block-group/grid-cell/neighborhood identifier.

| Field | Type | Notes |
| --- | --- | --- |
| `area_key` | string | e.g., `precinct:S-04`, `street:LINCOLN AVE`, `grid:lat0_lon4` |
| `geography_level` | enum | `precinct` \| `ward` \| `block_group` \| `neighborhood` \| `grid` \| `street` |
| `as_of_date` | date | Snapshot date — supports change-over-time charts |
| `n_voters_in_program` | int | All AFE/SNG records with a request on file |
| `ballots_requested`, `ballots_sent`, `ballots_received`, `ballots_outstanding` | int | NULL when `n_voters_in_program < 10` |
| `return_rate_pct` | float | NULL when suppressed |
| `median_days_to_return` | float | NULL when suppressed |
| `suppressed` | bool | TRUE when n < 10 |
| `centroid_lat`, `centroid_lon` | float | For map rendering |

### 2.3 Important note about this specific dataset

The 2026 South Ward extract does **not** include precinct, census block group, or geocoded coordinates — those need to be added through a join against the state voter file and a geocoding pass. The pipeline below does both. It also normalizes the street-name casing inconsistencies present in the source (e.g., `LINCOLN AVE` vs `Lincoln Ave` are merged).

---

## 3. Dashboard layout

A single-page executive layout with a fixed left rail of filters, a top KPI strip, and three primary content panels. Admin role unlocks a fourth Directory tab.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CITY OF ORANGE · VOTE-BY-MAIL OPERATIONS         [Public ▼] [Admin]    │  ← role toggle
├──────────────────────────────────────────────────────────────────────────┤
│  KPI strip: voters | mailed | received | outstanding | return rate | ↑↓ │
├────────────┬──────────────────────────────────┬──────────────────────────┤
│            │  Map View                        │  Insights                │
│  Filters   │  ──────────────────────────────  │  ──────────────────────  │
│            │  Aggregation: [precinct ▼]       │  Highest return area     │
│  Ward      │  Layer: [return rate ▼]          │  Most outstanding area   │
│  Precinct  │  Choropleth + threshold suppress │  Growth last 3 days      │
│  Status    │                                  │  Areas below avg         │
│  Date range│                                  │                          │
│  Neigh.    ├──────────────────────────────────┴──────────────────────────┤
│            │  Trends                                                     │
│  [Reset]   │  Daily returns | Cumulative | Return rate by area           │
│  [Export]  │                                                             │
└────────────┴─────────────────────────────────────────────────────────────┘
                                  Directory (admin role only)
                          ┌────────────────────────────────────┐
                          │ Search… [____]  filters apply      │
                          │ Name | Address | Mailing | Status  │
                          └────────────────────────────────────┘
```

**Visual style.** Navy `#0B1E3F` background, gold `#C9A227` accent, white text, muted gray `#6B7280` secondary text. Cards have a 1px gold-tinted border on hover, no drop shadows. Charts use the same palette plus a single sequential ramp for the choropleth (`#1B3A6B → #C9A227`).

---

## 4. Cleaning and aggregation pseudocode

The actual implementation is provided in `clean_aggregate.py`. The pseudocode below is the contract.

```python
def clean(source_xlsx) -> list[VoterRecord]:
    rows = read_xlsx(source_xlsx)
    out = []
    for row in rows:
        rec = VoterRecord(
            voter_id    = row.ID,
            last_name   = row['Last Name'],
            first_name  = row['First Name'],
            party       = row.Party,            # admin only
            street      = canonicalize(row['Street Name']),
            req_date    = parse_date(row['Ballot Req Rcvd Date']),
            sent_date   = parse_date(row['Ballot Mailed Date']),
            recv_date   = parse_date(row['Ballot Received Date']),
            status      = row['Ballot Counted Status'] or 'Outstanding',
        )
        rec.lat, rec.lon = geocode(rec.full_address)        # Census → Mapbox fallback
        rec.precinct     = join_state_voter_file(rec.voter_id)
        rec.days_to_return = (rec.recv_date - rec.sent_date).days if both else None
        out.append(rec)
    return out

def canonicalize(street):
    s = upper(strip(collapse_whitespace(street)))
    return apply({' AVENUE':' AVE', ' STREET':' ST', ' ROAD':' RD', ...})

def aggregate(records, level, as_of):
    bucket = group_by(records, key=area_for(level))   # precinct | grid | street | bg
    out = []
    for area, recs in bucket.items():
        n  = len(recs)
        r  = count(recs, status='Received')
        suppressed = n < PRIVACY_THRESHOLD          # default 10
        out.append(AreaAggregate(
            area_key       = area,
            n_voters       = n,
            ballots_sent   = count(recs, has(sent_date)),
            ballots_received = None if suppressed else r,
            outstanding    = None if suppressed else n - r,
            return_rate    = None if suppressed else r / n,
            median_days    = None if suppressed else median(days_to_return for recs),
            suppressed     = suppressed,
            as_of_date     = as_of,
        ))
    return out

def auto_insights(aggregates, daily_timeline):
    visible = [a for a in aggregates if not a.suppressed]
    return {
        'highest_return':  max(visible, key=lambda a: a.return_rate),
        'most_outstanding':max(visible, key=lambda a: a.outstanding),
        'growth_3d':       sum(d.received for d in daily_timeline[-3:])
                           - sum(d.received for d in daily_timeline[-6:-3]),
        'below_avg':       [a for a in visible if a.return_rate < avg(visible)][:5],
    }
```

---

## 5. React component structure

```
src/
├── App.tsx                       // routes + role context
├── lib/
│   ├── api.ts                    // fetch /aggregates, /records (admin), /timeline
│   ├── privacy.ts                // suppression helpers; threshold constant
│   ├── format.ts                 // date/percent helpers
│   └── colorScale.ts             // navy → gold sequential ramp
├── context/
│   ├── RoleContext.tsx           // Public | Admin (server-validated JWT)
│   └── FiltersContext.tsx        // ward, precinct, status, date range, neighborhood
├── components/
│   ├── layout/
│   │   ├── Shell.tsx             // top bar + left rail + main grid
│   │   ├── RoleBadge.tsx
│   │   └── ExportMenu.tsx        // CSV (aggregates), PDF summary
│   ├── filters/
│   │   ├── FilterRail.tsx
│   │   ├── DateRangePicker.tsx
│   │   └── ResetButton.tsx
│   ├── kpis/
│   │   └── KpiStrip.tsx          // voters | mailed | received | outstanding | rate
│   ├── map/
│   │   ├── MapView.tsx           // Leaflet/Mapbox container
│   │   ├── AggregationToggle.tsx // precinct | ward | block group | grid
│   │   ├── LayerToggle.tsx       // requested | sent | returned | rate | outstanding
│   │   ├── ChoroplethLayer.tsx   // GeoJSON + colorScale + suppression mask
│   │   └── PointLayer.tsx        // admin role ONLY; individual markers
│   ├── trends/
│   │   ├── DailyReturnsChart.tsx
│   │   ├── CumulativeChart.tsx
│   │   ├── ReturnRateByAreaChart.tsx
│   │   └── OutstandingByAreaChart.tsx
│   ├── insights/
│   │   ├── InsightCardGrid.tsx
│   │   └── InsightCard.tsx       // title + value + plain-language sentence
│   └── directory/                // ADMIN ROLE ONLY
│       ├── DirectoryTable.tsx    // virtualized
│       ├── DirectoryRow.tsx
│       └── VoterDetailDrawer.tsx
└── pages/
    ├── Overview.tsx              // KPIs + map + trends + insights
    └── Directory.tsx             // role-guarded
```

Role gating is enforced server-side (the `/records` endpoint requires admin JWT). The `PointLayer` component throws if it ever receives data while `role === 'public'` — defense in depth.

---

## 6. Chart and map logic

### 6.1 Choropleth (Leaflet + GeoJSON)

```ts
function ChoroplethLayer({ aggregates, layer, suppressionThreshold = 10 }) {
  const max = Math.max(...aggregates.filter(a => !a.suppressed).map(a => a[layer]));
  const scale = d3.scaleSequential([0, max], d3.interpolateRgb('#1B3A6B', '#C9A227'));
  return aggregates.map(a => (
    <GeoJSON
      key={a.area_key}
      data={a.geometry}
      style={() => a.suppressed
        ? { fillColor: '#2A3441', fillPattern: 'hatch', fillOpacity: 0.5,
            color: '#6B7280', weight: 1 }
        : { fillColor: scale(a[layer]), fillOpacity: 0.75,
            color: '#0B1E3F', weight: 1 }}
      onEachFeature={(_, lyr) => lyr.bindTooltip(a.suppressed
        ? `${a.area_key} — fewer than 10 voters; suppressed`
        : `${a.area_key} — ${a[layer]} (${a.return_rate}%)`)}
    />
  ));
}
```

The suppression branch must be the only branch that runs when `suppressed` — a single switch ensures no number leaks through `aria-label`, `title`, or tooltip.

### 6.2 Daily-returns line chart (Recharts)

```tsx
<LineChart data={daily}>
  <XAxis dataKey="date" />
  <YAxis />
  <Tooltip />
  <Line type="monotone" dataKey="mailed"   stroke="#6B7280" strokeDasharray="4 4" />
  <Line type="monotone" dataKey="received" stroke="#C9A227" strokeWidth={2} />
</LineChart>
```

### 6.3 Cumulative-returned area chart

```tsx
<AreaChart data={cumulative}>
  <XAxis dataKey="date" />
  <YAxis />
  <Area dataKey="cumulative" stroke="#C9A227" fill="#C9A227" fillOpacity={0.2} />
</AreaChart>
```

### 6.4 Return-rate-by-area horizontal bar

Sort by `return_rate` desc; suppressed rows render as a single gray bar with "<10 — suppressed" label and no numeric value.

### 6.5 Aggregation toggles

The map exposes an aggregation control (precinct / ward / block group / grid) and a layer control (requested / sent / returned / return-rate / outstanding / change-over-time). All combinations route to the same `area_aggregate` table with `geography_level` and `as_of_date` filters — no special-casing.

---

## 7. Privacy-safe display approach

| Surface | Identifiers visible? | Aggregation | Threshold | Notes |
| --- | --- | --- | --- | --- |
| KPI strip | No | Citywide / filter scope | Always passes | |
| Map (public) | No | Selected geography | n ≥ 10 enforced server-side | Suppressed tiles rendered with hatch pattern, no number |
| Map (admin) | Optional point layer | Points only when explicitly toggled | Points masked to nearest 50ft when zoomed in | |
| Trend charts | No | Filter scope | If filter yields n < 10, chart shows "Insufficient data" | |
| Insight cards | No | Visible (non-suppressed) areas only | Built from `visible = aggregates.filter(!suppressed)` | |
| Directory | Yes | Per voter | Admin role only, audit-logged | |
| CSV export | Aggregate CSV: no PII. Record CSV: admin only. | Honors current filters | Suppressed rows have blank numeric cells in aggregate CSV | |
| PDF summary | No | Aggregate only | Same rules as above | |

The suppression check lives in **one** function (`shouldSuppress(n)`) used by every aggregate constructor; UI components cannot opt out.

---

## 8. Suggested auto-generated insights

Computed on each refresh; render as cards in the right-side Insights panel. **The current South Ward dataset values are shown in parentheses.**

1. **Highest vote-by-mail return rate** — `BERKELEY AVE, 65.8% (48 of 73 returned)`
2. **Largest number of outstanding ballots** — `LINCOLN AVE, 78 outstanding of 112 mailed`
3. **Fastest growth over last 3 days** — Net change in daily returns vs the prior 3 days. *(Current trailing-3d total of 20 returns is 14 below the prior 3-day total, signaling a slowdown worth investigating.)*
4. **Areas below citywide average** — Up to 5 visible areas whose return rate is below the citywide average (currently 39.5%).
5. **Median days to return** — Useful for forecasting outstanding ballots that may still arrive before the deadline.
6. **Out-of-jurisdiction mailings** — Count of records whose mailing ZIP differs from residence ZIP. Operationally useful (forwarding addresses, college students).
7. **Same-day request-and-return** — Unusual pattern worth flagging for review.

Each card includes a one-sentence plain-language summary and a "View on map" link that highlights the area.

---

## 9. Suggested tech stack

| Layer | Choice | Why |
| --- | --- | --- |
| Front-end | React + TypeScript + Vite | Fast, typed, good ecosystem |
| Map | Leaflet + OpenStreetMap tiles | Free, no key required; swap to Mapbox if you want vector styling |
| Charts | Recharts | Pairs cleanly with React and the palette |
| Backend | Node/Express or FastAPI | Serves `/aggregates`, `/timeline`, `/records` (admin) |
| Cleaning | Python + pandas | `clean_aggregate.py` provided |
| Geocoding | U.S. Census Geocoder (free, batch) primary; Mapbox fallback for unmatched | Census API is appropriate for civic use |
| Storage | Postgres + PostGIS | Geographic joins, fast aggregation |
| Auth | OAuth (Google/Microsoft) → JWT with role claim | `public` vs `admin`; admin endpoints require admin claim |
| Export | CSV (aggregate; record CSV admin only); PDF via headless Chromium | |
| Hosting | Internal city hosting or Vercel (front) + Fly.io/AWS (API) | Match Orange IT policy |

---

## 10. Operational follow-up suggestions

Below are the dashboard's intended uses, written as plain-English questions an operator should be able to answer in under 30 seconds:

- "Where should canvass teams go this week?" → Outstanding-by-area bar chart, sorted descending.
- "Are returns slowing down?" → Daily-returns line + Cumulative area chart together.
- "Which areas are below average and at risk of low participation?" → Below-average insight card.
- "Are any ballots being mailed out-of-jurisdiction?" → Out-of-jurisdiction insight card.
- "Has the city seen an unusually long return-time recently?" → Median days-to-return card with sparkline.

---

## 11. What's in this delivery

| File | Purpose |
| --- | --- |
| `design.md` | This document |
| `dashboard.html` | Single-file working prototype (Directory / Map / Trends / Insights) preloaded with the cleaned South Ward data; role toggle simulates public vs admin |
| `clean_aggregate.py` | Reusable Python pipeline — ingests the xlsx, normalizes streets, aggregates with privacy threshold, writes JSON + CSV |
| `cleaned_voters.csv` | Admin-only record-level export |
| `aggregate_by_street.csv` | Privacy-safe aggregate export (suppressed counts blank) |
