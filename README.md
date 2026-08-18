# 📊 Executive Market Intelligence Dashboard — Power BI

> A star-schema Power BI dashboard tracking competitor pricing, risk exposure, and stock signals — designed with AI-assisted architecture, built and validated by hand.

This is the third build in a small competitor-intelligence series (following an automated scraping pipeline and a Looker Studio dashboard). This version focuses on executive-level reporting: a proper star schema, risk-tiered SKU flagging, and an AI-generated narrative summary layer — while being explicit about where AI assistance actually started and stopped.

## 🧭 A note on "AI-built"

The full data model, DAX measures, and Deneb/Vega-Lite chart spec in this project were designed through conversations with **Claude and Gemini** — not Power BI's in-app Copilot, which requires a paid Microsoft Fabric capacity (F2+, ~€260/month) and wasn't part of this build. Formulas and structure were suggested in chat, then implemented, tested, and debugged manually in free Power BI Desktop.

This distinction matters beyond licensing: AI-suggested DAX and thresholds still needed human validation against the actual data before being trusted — see [Where a human was still required](#-where-a-human-was-still-required) below.

## 📐 Data model — Star Schema

```
Dim_Calendar (1) ──┐
                    ├──► price_compare_log (fact)
Dim_Product  (1) ──┘
```

- `Dim_Calendar` and `Dim_Product` are dimension tables filtering the fact table.
- `price_compare_log` is the single fact table driving all measures — fact-to-fact relationships were deliberately avoided to prevent ambiguous/broken filter paths, a common star-schema pitfall.
- `market_intelligence_log` (raw daily scrape log) is merged into the model via Power Query rather than a live relationship, using a composite key of `product` + `optical_condition` + `date`. All three fields are required — `product` + `date` alone isn't unique, since a single product has multiple rows per day (one per condition tier), which would cause an incorrect many-to-one match:

```powerquery
let
    Source = price_compare_log,
    #"Merged Queries" = Table.NestedJoin(
        Source, {"product", "optical_condition", "report_date"},
        market_intelligence_log, {"product", "optical_condition", "scraped_date"},
        "market_intelligence_log", JoinKind.LeftOuter
    ),
    #"Expanded market_intelligence_log" = Table.ExpandTableColumn(
        #"Merged Queries", "market_intelligence_log", {"inStock"}, {"inStock"}
    )
in
    #"Expanded market_intelligence_log"
```

## 🧮 Key DAX measures

**Core pricing & trend**
```dax
Avg Price Gap % =
DIVIDE(
    AVERAGE('price_compare_log'[OurPrice]) - AVERAGE('price_compare_log'[CompetitorPrice]),
    AVERAGE('price_compare_log'[CompetitorPrice])
)

Comp Price MA3 =
CALCULATE(
    AVERAGE('price_compare_log'[CompetitorPrice]),
    DATESINPERIOD('Dim_Calendar'[Date], MAX('Dim_Calendar'[Date]), -3, DAY)
)

Expected Upper Band = [Comp Price MA3] + (2 * [Price StdDev 8D])
Expected Lower Band = MAX(400, [Comp Price MA3] - (2 * [Price StdDev 8D]))
```

**Risk & alerting**
```dax
Risk SKU Count =
CALCULATE(
    DISTINCTCOUNT('price_compare_log'[product]),
    FILTER(
        'price_compare_log',
        ABS([Avg Price Gap %]) > 0.10 &&
        LOWER('price_compare_log'[inStock]) = "available"
    )
)

Total Price Alerts =
CALCULATE(
    COUNTROWS('price_compare_log'),
    'price_compare_log'[price_difference] >= 0,
    LOWER('price_compare_log'[inStock]) = "available"
)
```

**Smart Narrative** — a DAX measure that auto-generates an executive summary string (with dynamic status icons) from the current filter context. Full implementation in this repo's `dax_measures.dax`.

> Full measure list (SKU status icons, OOS rate, max-drop SKU, etc.) is documented in the accompanying `.dax` file in this repo.

## 📈 Layout

┌─────────────────────────────────────────────────────────────────────────────────┐
│ [LEFT PANEL - SLICERS]        │ [HEADER BANNER] Executive Market Intelligence   │
│ 🔘 Product Name               ├─────────────────────────────────────────────────┤
│ 🔘 Optical Condition          │ [KPI CARDS]                                     │
│ 🔘 Date Range (3/8 - 10/8)    │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐│
│                               │ │GAP %     │ │SKU RISK  │ │ALERTS    │ │OOS RATE││
│ 💡 DATA NOTE:                 │ │ +3.1%    │ │ 3 SKUs   │ │ 42 Alerts│ │ 18.5%  ││
│ "Window: 3-day MA due to 8-day│ └──────────┘ └──────────┘ └──────────┘ └────────┘│
│ tracking history."            ├─────────────────────────────────────────────────┤
│                               │ [TIME SERIES - PRICE TREND & EXPECTED RANGE]    │
│                               │ Visual: Line & Clustered Column Chart           │
│                               │  - Line 1: Our Price MA3                        │
│                               │  - Line 2: Comp Price MA3                       │
│                               │  - Error Bands: Expected Upper/Lower Bounds     │
├───────────────────────────────┼─────────────────────────────────────────────────┤
│ [EXECUTIVE FOOTER]            │ [RANKED TABLE - LARGEST PRICE MOVES]            │
│ Smart Narrative: AI-generated │ Columns: Product | Condition | Our | Comp | Gap │
│ summary of key price shifts.  │ Visual Features: Data Bars on Gap %, Status Icon│
└───────────────────────────────┴─────────────────────────────────────────────────┘

- **Header**: title, data freshness note, last-scraped-date card
- **Left panel**: slicers (date, product group, condition, stock status) + AI-generated Smart Narrative box
- **KPI row**: Price Gap %, SKUs at Risk, Price Alerts, Competitor OOS Rate
- **Main chart**: Deneb/Vega-Lite custom visual — 3-day moving average trendline with a ±2 standard-deviation confidence band
- **Ranked table**: largest price moves, with data bars and colour-coded risk-status icons

## 🎨 Custom visual (Deneb / Vega-Lite)

The trend chart is a custom Deneb spec layering a shaded confidence band (`Expected Lower/Upper Band`) under the moving-average price line — full JSON spec in `deneb_price_trend.json` in this repo.

## 💡 Key insight

The dashboard's own AI-generated narrative reported average pricing as **"aligned with market pricing (+4.1%)"** — a reassuring, technically accurate headline. Sitting directly beneath it: **8 SKUs individually breaching the ±10% risk threshold**, invisible in the averaged number. This is a textbook case of an aggregate metric smoothing over real, actionable outliers — exactly the kind of gap a dashboard's narrative text can't be trusted to catch on its own, and exactly what a human reviewing the underlying table is still needed for.

## 🧠 Where a human was still required

AI produced strong first drafts of the schema, measures, and narrative logic — but several decisions and fixes still required manual judgment:

- **Threshold calibration**: the ±10% "risk" and ±15% "underpriced" bands are business judgment calls specific to this margin structure, not something derivable from the data alone.
- **Reading past the headline metric**: as above — the AI narrative's "aligned" summary needed a human to notice the SKU-level risk count told a different story.
- **Star-schema correctness**: validating that dimension-to-fact relationships didn't create ambiguous filter paths required manually testing slicer behavior, not just accepting the suggested model.
- **Window sizing**: using a 3-day (not 7-day) moving average was a deliberate adjustment for the current ~7–8 day tracking history — a decision based on data volume, not something the AI defaulted to correctly without being told the constraint.
- **Data quality**: as with the earlier project in this series, AI-suggested formulas were tested against real data and iterated on where the first version didn't hold up (mismatched types, filter context issues) — the working version here is the result of that back-and-forth, not a first draft used as-is.
- **Join key correctness**: an early version of the Power Query merge used only `product` + `date` as the composite key. That's not actually unique in this dataset — each product has one row per condition tier per day — so it silently produced incorrect many-to-one matches until caught and fixed by adding `optical_condition` to the key.

## 🚀 Setup

1. Connect Power BI Desktop (free) to the source Google Sheets pipeline (or export as CSV).
2. Build `Dim_Calendar` and `Dim_Product` dimension tables; merge `market_intelligence_log` into `price_compare_log` via the Power Query script above.
3. Add the DAX measures (see `dax_measures.dax`).
4. Install the Deneb custom visual from AppSource (free) and paste in `deneb_price_trend.json`.
5. Build the layout as described, or adapt thresholds to your own margin structure.

## 🏷️ Tags

`#PowerBI` `#DAX` `#DataModeling` `#CompetitorIntelligence` `#BusinessIntelligence` `#AI`
