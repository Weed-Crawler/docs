<!-- Published copy, generated from WeedCrawler's private repository. Do not edit here: changes are overwritten on the next publish. -->
# BBFYB Data Feed — Data Dictionary

*Best Bang For Your Bud (BBFYB) retail cannabis dataset — reference guide for data consumers.*
*Version 2.1 — October 2026 (changelog at the bottom). Questions: hi@weedcrawler.ca*

## What this dataset is

BBFYB continuously observes Canadian recreational cannabis retail — store menus,
prices, inventory levels, and daily sales movements — across thousands of stores,
and consolidates everything into a clean, analysis-ready **star schema**: three
fact tables (what happened), six dimension tables (who/what/where), and one
convenience rollup. Beside it sit Statistics Canada's official monthly market
totals and population (`FCT_STATCAN_RETAIL_MONTHLY`, `DIM_GEO_POPULATION`, since
v1.5): market context by province, never a denominator for the facts above.

```
                    DIM_BRAND_PRODUCER
                           |
                      DIM_PRODUCT ──── DIM_PRODUCT_VARIANT ─── DIM_VARIANT_IDENTIFIER
                                              |  └──── DIM_PRODUCT_SUPPLIER (by province)
                                              |
       FCT_SALES_DAILY ───────────────────────┤
       FCT_INVENTORY_DAILY ───────────────────┤        FCT_REGULATOR_MONTHLY_SALES
       RPT_AVAILABILITY_WEEKLY ───────────────┘             (joins via PCV_ID)
              |
       DIM_MASTER_STORE

       V_SALES_DAILY_ENRICHED = FCT_SALES_DAILY ⨝ all dims (+ EOD_QTY),
       pre-joined for no-join querying (internal view — not in the Fabric feed)

       FCT_STATCAN_RETAIL_MONTHLY ─── DIM_GEO_POPULATION      (official market
              (joins by GEO_CODE; PROVINCE_STATE matches DIM_MASTER_STORE)   context)
```

**Join keys:** facts carry `MASTER_STORE_ID` (→ DIM_MASTER_STORE) and `PCV_ID`
(→ DIM_PRODUCT_VARIANT → DIM_PRODUCT → DIM_BRAND_PRODUCER). Who supplies a
product in a given province: `PCV_ID` + province → DIM_PRODUCT_SUPPLIER.

## Coverage: two sources, one catalog

Since v1.4 the daily facts cover **every province, Quebec included**, from two
sources that behave differently. Read this before comparing provinces.

| | Rest of Canada | Quebec |
|---|---|---|
| What is observed | a **covered panel**: the retailers BBFYB can read (~2,600 stores a day out of every licensed store) | a **census**: every SQDC store (~115, plus the SQDC online store), every day |
| Daily history starts | **2025-01-01** | **2020-06-23** |
| Modeled days (`IS_SYNTHETIC`) | yes, 18–20% of dollars in 2025–2026 | never |
| Stores reporting only "available" (no quantity) | ~14% of stores | none: every Quebec store reports quantities |
| Product mapping (`PCV_ID` NULL) | ≈10% of dollars unmapped | ≈0% since 2025 |

Both sources share **one product catalog and one taxonomy**: a product sold in
Quebec and in Ontario is the same `PC_ID` / `PCV_ID`, with the same `CATEGORY`,
`SUBCATEGORY`, brand and producer. There is nothing to translate or re-map.

> **Before 2025-01-01, every "national" figure is Quebec only.** A chart of total
> sales from 2020 shows a Quebec-shaped market that quadruples in January 2025,
> when the other provinces enter the data. That jump is coverage, not the market.
> For national trends start at 2025-01-01; for Quebec alone you can use the full
> history. Quebec's share of the sales in this dataset:
>
> | Year | Quebec (M$) | Rest of Canada (M$) | Quebec share |
> |---|---|---|---|
> | 2020 (from Jun 23) | 297.9 | — | 100% |
> | 2021 | 659.2 | — | 100% |
> | 2022 | 664.0 | — | 100% |
> | 2023 | 715.0 | — | 100% |
> | 2024 | 816.4 | — | 100% |
> | 2025 | 908.2 | 2,583.6 | 26% |
> | 2026 to Sep 24 | 715.3 | 2,250.0 | 24% |

**A census and a panel do not add up to "the market".** Quebec figures describe
the whole legal market of that province; rest-of-Canada figures describe the
stores BBFYB covers. A national total is therefore a *covered-market* total.
Compare provinces on shares and ratios, not on raw dollars or store counts.

**Reproducing pre-v1.4 numbers.** Anything you built before September 25, 2026
was rest of Canada only. Add `m.PROVINCE_STATE <> 'QC'` (join
`DIM_MASTER_STORE m`) to get the same totals back.

### Known gaps

Two collection outages left holes in the history. The data for those days
cannot be recovered, so the rows stay as they are: **exclude these periods**
from trends, year-over-year comparisons and averages for these provinces,
rather than reading them as market moves. They affect `FCT_SALES_DAILY`,
`FCT_INVENTORY_DAILY` and `RPT_AVAILABILITY_WEEKLY` alike.

| Province | Affected | What is missing | Coverage after |
|---|---|---|---|
| **New Brunswick** | **2025-05-13 to 2026-04-26** | only 4 to 6 of the ~29 stores were collected (about 3% of the province's usual dollars) | 34 stores from 2026-04-27, 39-41 since July 2026 |
| **Nova Scotia** | **2026-02-16 to 2026-04-22** | no stores at all; February and April 2026 are partial months and March 2026 has no rows | 84 stores from 2026-04-23 (51 before) |

In both provinces more stores are collected after the outage than before it, so
store counts, doors and province totals **step up on the recovery date**. That
step is coverage, not growth: compare like-for-like stores across it.

We found both by reconciling every province against Statistics Canada's monthly
retail sales (see `FCT_STATCAN_RETAIL_MONTHLY`), which we now do routinely; any
new gap will be listed here.

## Freshness & revisions

- Data refreshes **daily around 09:00 Eastern**; new data is visible within minutes.
- **Two independent pipelines feed each daily fact**: the rest of Canada (a covered
  panel of scraped retailers) and Quebec (a daily census of every SQDC store). Each
  is checked before it replaces anything: if a source arrives late, unreadable or
  visibly incomplete (fewer stores or rows than it already had), that source keeps
  its previous complete version for the day and the other source updates as normal.
  Nothing is ever replaced with a partial day. **`REFRESH_STATUS`** (also the view
  `V_REFRESH_STATUS`) says, per fact and per source, the last decision, when it
  was published (Eastern time) and the day the published data runs through
  (`DATA_THROUGH`); `COMMON_THROUGH` is the day
  both sources are published through, which is the safe end date for national
  totals. It is refreshed at the end of every run and reaches Fabric like the other
  tables.
- Figures for the trailing **7 days are provisional** and may be revised as
  late-arriving store data lands. From day 8 onward, numbers are stable, with two
  exceptions below.
- **Quebec rectifications.** The SQDC occasionally freezes its public inventory
  for a few days. The sales of the frozen days are corrected once it thaws,
  sometimes more than 8 days later. The weekly re-map (every Sunday,
  trailing 6 months of `FCT_SALES_DAILY`) picks those corrections up, so a Quebec
  week can move once more after day 8. `FCT_INVENTORY_DAILY` is not re-covered
  weekly: inventory keeps what was observed at the time.
- **Catalog re-mapping.** The same weekly re-map moves sales onto a product when
  it gets matched or merged in the catalog, over the trailing 6 months.
- Daily history begins **2025-01-01** for the rest of Canada and **2020-06-23**
  for Quebec (sales & inventory; see *Coverage*). Regulator monthly data begins
  **January 2024**.
- **Statistics Canada tables** (`FCT_STATCAN_RETAIL_MONTHLY`, `DIM_GEO_POPULATION`)
  are rewritten daily around 10:00 Eastern from a weekly fetch of StatCan's full
  history, from **October 2018**. StatCan publishes a month about **7-8 weeks**
  after it ends and **revises up to about 3 years back**, so any month can change;
  each row carries StatCan's release time (`SOURCE_RELEASED_AT`). If StatCan's
  answer is incomplete, the tables keep their previous version for the day.

## Conventions

- All money is in **Canadian dollars** (never cents). All dates are calendar dates
  (store close-of-day).
- **Sales tax.** Sales dollars are shelf price × units, so they follow each
  province's menu pricing: **Quebec (SQDC), Nova Scotia (NSLC) and New Brunswick
  (Cannabis NB) prices include sales tax** (QST+GST 14.975%; HST 14%, 15% before
  April 2025; HST 15%); the other provinces' store menus list prices before tax.
  Compare provinces on ratios and shares, or remove the tax first with
  `DIM_SALES_TAX` (since v1.6.1): `SALES_DOLLARS / (1 + INCLUDED_TAX_RATE)`. Multi-buy
  deals (e.g. "2 for $72") are not visible on menus, so provinces that run them
  (notably New Brunswick) read slightly high.
- **Sales units are fractional** by design: when a store's feed is briefly
  unavailable, BBFYB estimates the missing days (see `IS_SYNTHETIC`), and
  estimates carry decimals. Sum first, round last.
- Product taxonomy is BBFYB's standardized ladder:
  `CATEGORY` (flower / extracts / vapes / edibles / topicals / accessories) →
  `SUBCATEGORY` (dried flower, pre-rolls, oils, beverages, …) →
  `SUB_SUBCATEGORY` (whole flower, milled flower, single-strain pre-rolls, …).
  Quebec products use the same ladder. Some subcategories are simply **not sold
  in Quebec**: soft chews and disposable pens show no Quebec sales at all, so an
  empty Quebec cell there is the assortment, not missing data. Infused pre-rolls
  are `extracts / infused pre-rolls` in every province.
- **One barcode, one product, the latest name.** A product is identified by its
  barcode over time. When the same barcode was sold under different names in
  different provinces or years (a rebrand, or a product renamed for Quebec), the
  whole history sits on one product carrying its **latest** official name. A
  2022 Quebec sale can therefore show a product name that did not exist yet in
  2022.

---

## Fact tables

### FCT_SALES_DAILY — daily sales by store × product variant

One row per store × product variant × day **where something sold** (sparse: days
with no movement have no row). ~273M rows: 2025-01-01 onward for the rest of
Canada, 2020-06-23 onward for Quebec.

| Column | Type | Description |
|---|---|---|
| MASTER_STORE_ID | int | Store — join DIM_MASTER_STORE |
| PCV_ID | int | Product variant — join DIM_PRODUCT_VARIANT. **NULL ≈ 10% of dollars**: products not yet in the consolidated catalog. Kept for honest market totals; see UNMAPPED_CATEGORY for what they are. |
| CLOSING_ON | date | Store close-of-day the sales belong to |
| SALES_UNITS | decimal | Units sold that day (fractional — see conventions) |
| SALES_DOLLARS | decimal | Revenue that day, CAD |
| UNIT_PRICE | decimal | SALES_DOLLARS / SALES_UNITS for the day |
| IS_SYNTHETIC | boolean | TRUE = modeled estimate filling a gap in the store's feed, not an observed transaction. Rest of Canada only (18–20% of its dollars in 2025–2026); **always FALSE in Quebec**, which is observed every day. Filter out with `WHERE NOT IS_SYNTHETIC` if you want observed-only. |
| UNMAPPED_CATEGORY | text | **Only on NULL-PCV rows** (always NULL on mapped rows): what the unmapped product is, in the same taxonomy as DIM_PRODUCT — `flower`, `vapes`, `edibles`, `extracts`, `topicals`, `accessories`, or `unknown`. Derived from the source listing's raw category/name. |
| UNMAPPED_SUBCATEGORY | text | Best-effort subcategory for NULL-PCV rows (`pre-rolls`, `dried flower`, `battery`, …). Two values exist only here: `nicotine` and `apparel` (under `accessories`). NULL when there's no signal. |

> **Cannabis-only market totals** (excludes accessories, apparel & nicotine sold
> in cannabis stores): `WHERE COALESCE(f.UNMAPPED_CATEGORY,'x') <> 'accessories'`
> — plus the usual `DIM_PRODUCT.CATEGORY <> 'accessories'` on mapped rows.
> For **complete category shares** including the unmapped tail:
> `COALESCE(p.CATEGORY, f.UNMAPPED_CATEGORY)`.
>
> Grain note: mapped rows are one per store × variant × day; unmapped rows are
> one per store × day × (UNMAPPED_CATEGORY, UNMAPPED_SUBCATEGORY).
>
> **Known understatement, Quebec 2020–2022.** About 6,200 Quebec rows from those
> years (≈33,000 units) carry units but no dollars: the sale was recorded before
> the product's first price was known, and the raw data needed to price them no
> longer exists. That is roughly $0.9M against $1.62B for the three years
> (0.05%). `SALES_UNITS` is complete; `SALES_DOLLARS` reads very slightly light.
> From 2023 onward every such row has been priced.

### FCT_INVENTORY_DAILY — daily end-of-day inventory position

One row per store × product variant × **day the product appeared on the store's
menu** (dense — includes zero-stock days). ~2.7B rows (Quebec from 2020-06-22);
always filter by `CLOSING_ON`, and prefer RPT_AVAILABILITY_WEEKLY for
distribution questions.

**This is the table for units left on shelves.** `FCT_SALES_DAILY` cannot
rebuild them: it records units sold on days something sold, while stock also
moves with deliveries, returns and adjustments. `EOD_QTY` is what the store
reported on hand at close, summed across the store's menu entries for that
variant. A row exists only on days the store's menu was read; days with no
reading have no row, and the last quantity is never carried forward. For "how
much is on the shelf now", take each store's latest row and check its
`CLOSING_ON` (see *Worked examples*).

Some retailers' menus only say whether a product is *available*, never how many
units are on hand (about 14% of stores, concentrated in BC). For those stores,
an available day is a row with `EOD_QTY` NULL and `IS_AVAILABLE` TRUE. **Use
`IS_AVAILABLE` for "could a shopper buy it that day"; use `IS_IN_STOCK` only
when you need a quantity.** `IS_IN_STOCK` implies `IS_AVAILABLE`.

Menus keep showing products long after the store stopped carrying them, so a row
here does not mean the store carries the product. **Use `IS_LISTED` for "does
this store carry it"**: TRUE while the product was available or sold at that
store at least once in its last 21 observed days. `IS_AVAILABLE` implies
`IS_LISTED`.

| Column | Type | Description |
|---|---|---|
| MASTER_STORE_ID | int | Store |
| PCV_ID | int | Product variant (NULL = unconsolidated) |
| CLOSING_ON | date | Day of the reading |
| EOD_QTY | decimal | Units on hand at end of day (capped at 100,000 — a handful of upstream feed glitches are sanitized). **NULL** when the store reports availability but not a quantity |
| IS_IN_STOCK | boolean | EOD_QTY > 0 (FALSE when the quantity is unknown) |
| IS_AVAILABLE | boolean | Purchasable that day: in stock, **or** reported available by a store that never reports quantities |
| IS_LISTED | boolean | The store carries the product that day: it was available, or sold a real unit, at least once in the pair's last 21 **observed** days (days with no row for the pair do not count). A listing ends 21 observed days after the last stock or sale and comes back with a restock. **The column to use for distribution.** NULL on rows with no `PCV_ID` |

### FCT_REGULATOR_MONTHLY_SALES — Ontario (OCS) market data

Official monthly wholesale **sell-through** by SKU, region and channel.
~680K rows, January 2024 onward.

> **Important:** these figures are *sell-through* (product sold onward), **not**
> "units distributed/shipped". They will not match OCS reports that count
> distribution — the gap between shipped and sold is real information.

| Column | Type | Description |
|---|---|---|
| REGULATOR_CODE | text | Data source (`OCS`) |
| YEAR_MONTH | date | First day of the sales month |
| PCV_ID | int | Product variant (NULL = SKU not yet mapped; ~0% of dollars) |
| GTIN | text | Barcode as reported by the regulator |
| PROVINCIAL_SKU | text | Regulator's SKU identifier |
| REGION | text | OCS sales region (e.g. "GTA Region (Excludes Toronto)") |
| SALES_CHANNEL | text | e.g. WHOLESALE |
| SALES_UNITS | int | Units sold in the month |
| SALES_DOLLARS | decimal | Sales value, CAD |
| LIST_PRICE | decimal | Listed unit price, CAD |
| SUPPLIER | text | Licensed producer as reported by the regulator (authoritative) |
| BRAND / CATEGORY | text | Source-reported conveniences; prefer joining the DIM tables via PCV_ID |
| MATCH_STATUS | text | Whether the SKU is mapped to the BBFYB catalog |

### FCT_STATCAN_RETAIL_MONTHLY — official market size (Statistics Canada)

Statistics Canada's official **monthly retail-sales estimates for the cannabis-
retailer industry** (table 20-10-0056, NAICS 459993: stores whose main business
is selling cannabis; total retail sales, unadjusted). All brands and all
categories, **excluding sales taxes**, including those stores' own online
sales; not medical, not illicit, and not cannabis sold by other kinds of outlets.
One row per geography per month, **October 2018 onward**: Canada, the 13
provinces and territories, and 9 metropolitan areas. ~2,200 rows.

> **Important: market context, not a denominator.** Do not divide BBFYB sales by
> these figures to get a market share. BBFYB covers part of most provinces'
> stores, its Quebec/NS/NB dollars include sales tax while StatCan's do not, and
> its periods are days and weeks, not calendar months. In June 2026, BBFYB's
> Ontario dollars equalled about 78% of StatCan's, Alberta's about 36%.

| Column | Type | Description |
|---|---|---|
| GEO_CODE | text | `CA`, a province/territory code (`ON`, `QC`, …) or a metro slug (`toronto`, `montreal`, …) |
| GEO_NAME | text | Display name |
| GEO_TYPE | text | `country`, `province`, `territory` or `cma` (metropolitan area) |
| PROVINCE_STATE | text | Province of the geography (NULL for Canada); matches `DIM_MASTER_STORE.PROVINCE_STATE` |
| YEAR_MONTH | date | First day of the month |
| SALES_DOLLARS | decimal | Sales, CAD, excluding sales taxes. **NULL = StatCan did not publish the figure, never zero** |
| IS_SUPPRESSED | boolean | StatCan withheld the cell for confidentiality (`x`) |
| QUALITY_GRADE | text | StatCan's quality grade: A excellent, B very good, C good, D acceptable, E use with caution, F unreliable; NULL when not graded |
| STATCAN_SYMBOL | text | `p` preliminary, `r` revised, NULL final |
| SALES_DOLLARS_YEAR_AGO | decimal | The same month a year earlier |
| YOY_GROWTH | decimal | `SALES_DOLLARS / SALES_DOLLARS_YEAR_AGO − 1` (0.12 = +12%, nominal) |
| SHARE_OF_CANADA | decimal | `SALES_DOLLARS / Canada's SALES_DOLLARS` that month |
| POPULATION | int | StatCan population (all ages), latest quarter starting on or before the month; NULL for metros |
| SALES_PER_RESIDENT | decimal | Monthly `SALES_DOLLARS / POPULATION`: a geographic ratio, not spending per consumer |
| SOURCE_RELEASED_AT | timestamp | When StatCan released this value (UTC) |
| REFRESHED_AT | timestamp | Last rewrite of the table (UTC) |

`YOY_GROWTH`, `SHARE_OF_CANADA` and `SALES_PER_RESIDENT` are BBFYB calculations
from StatCan's inputs. Reading rules:

- **Never add rows of different `GEO_TYPE` together**: metros sit inside their
  province and provinces inside Canada. Filter `GEO_TYPE` first.
- `SHARE_OF_CANADA` divides by the Canada series itself, so provinces and
  territories add up to a little under 100% when StatCan withholds a cell. In
  the first two months (October and November 2018) StatCan withheld Manitoba, so
  they reach only 94% and 88%; from December 2018 on it is at least 98.5%.
- Monthly and not seasonally adjusted: compare a month with the same month a year
  earlier, not with the month before.
- Metropolitan areas are StatCan's census metropolitan areas as reported in this
  table (Ottawa and Gatineau appear separately, one per province), not
  municipalities.
- **British Columbia, October 2025**: StatCan published $31.3M, less than half the
  months around it (grade C), while BBFYB's BC shelf data shows no drop that month.
  We keep StatCan's value as published; treat that month, and October 2026's
  year-over-year change, with care until StatCan revises or explains it.

### DIM_GEO_POPULATION — population by geography and quarter (~450)

Statistics Canada population estimates (table 17-10-0009) for Canada and the 13
provinces and territories, quarterly, October 2018 onward. Revised by StatCan
like the sales table.

| Column | Type | Description |
|---|---|---|
| GEO_CODE | text | `CA` or a province/territory code |
| GEO_NAME | text | Display name |
| GEO_TYPE | text | `country`, `province` or `territory` |
| PROVINCE_STATE | text | NULL for Canada |
| QUARTER_START | date | Population as of this date (January, April, July, October 1) |
| POPULATION | int | Persons, all ages |
| STATCAN_SYMBOL | text | `p` preliminary, `r` revised, NULL final |
| SOURCE_RELEASED_AT | timestamp | When StatCan released this value (UTC) |
| REFRESHED_AT | timestamp | Last rewrite of the table (UTC) |

### FCT_SOCIAL_REVIEW_MENTION — what consumers say on Reddit, per product

One row per **Reddit post × catalogue product the author reviewed first-hand**,
from six Canadian cannabis communities: r/TheOCS, r/sqdc (French and English),
r/CanadianCannabisLPs, r/TheBCCS, r/RecPics and r/OCSreviews4TheCulture.
**July 2021 onward**, ~19,100 rows, ~17,800 posts, ~5,350 products, ~950 brands.
Refreshed daily around 10:30 Eastern.

How a row is made: a post is first classified as a review or not; a language
model then lists every product the post names and the role it plays
(reviewed, compared against, recommended, merely mentioned), with the author's
sentiment and a supporting quote. Only **reviewed** products are kept, and each
is matched to the catalogue by brand, product name and format (a flower review
never lands on the same strain's pre-roll). A product the model cannot match
with confidence is left out rather than guessed.

**Accuracy.** Graded by hand on a stratified sample of 2021–2026 posts: posts
reviewing one product 53/53 correct, posts naming several products 40/40
correct after the role step, sentiment 40/40. Our conservative estimate for the
whole table is **at least 93% of rows link the right product**.

> **Important: a signal, not a survey.** Reddit reviewers are enthusiasts,
> skew to Ontario (r/TheOCS is ~60% of rows) and to flower (~77% of rows;
> extracts and vapes ~11% each, edibles 2%), and post more about products
> that surprised them. Use this for direction and for
> what people say about a product, not for market share or satisfaction rates.
> Products with a handful of reviews swing on one post.

| Column | Type | Description |
|---|---|---|
| MENTION_ID | text | Unique row id (post × product) |
| PLATFORM | text | `reddit` |
| COMMUNITY | text | Subreddit name without `r/` |
| POST_URL | text | Link to the post on Reddit |
| POSTED_AT | timestamp | When the post was published on Reddit (UTC) |
| POST_TITLE | text | Post title as written |
| EVIDENCE_SNIPPET | text | Short quote from the post supporting the sentiment (≤ 500 characters) |
| PRODUCT_MENTION | text | Product as the author named it |
| BRAND_MENTION | text | Brand as the author named it; NULL if the post does not say |
| PC_ID | int | Catalogue product → `DIM_PRODUCT` |
| BRAND_ID | int | → `DIM_BRAND_PRODUCER` |
| SENTIMENT | text | `positive`, `neutral` or `negative`: the author's opinion **of this product** (a post can praise one product and pan another) |
| SENTIMENT_SCORE | int | +1 / 0 / −1 |
| UPVOTES | int | Reddit upvotes when collected; NULL for ~9% of rows (posts collected without vote counts, mostly 2024–2026) |
| UPVOTE_RATIO | decimal | Share of upvotes among votes (0–1) |
| MATCH_METHOD | text | `string` (exact catalogue name within the brand), `llm_pick` (chosen by the model among that brand's products) or `product_line` (the author named a product line as if it were the brand, e.g. "Gros Plaisirs - LA Kush Cake": linked to that product, Pure Laine's Gros Plaisirs) |
| MATCH_CONFIDENCE | decimal | 0–1; informational |
| REFRESHED_AT | timestamp | Last rewrite of the table (UTC) |

**`V_SOCIAL_REVIEW_MONTHLY`** (secure view, SQL only; not in the Fabric feed)
aggregates the table by product × month: `MENTIONS`, `POSITIVE` / `NEUTRAL` /
`NEGATIVE`, `PCT_POSITIVE`, `NET_SENTIMENT` (average score, −1 to +1),
`NET_SENTIMENT_UPVOTE_WEIGHTED` (each review weighted by upvotes + 1), and the
same month's **category benchmark** (`CATEGORY_MENTIONS`, `CATEGORY_NET_SENTIMENT`)
so a product can be read against its category.

Reading rules:

- **Known gap: June–September 2026.** Reddit collection was down from
  2026-05-28 to 2026-10-02. June and July have no rows, August and September a
  few dozen (only what Reddit still listed when collection resumed). Exclude
  those months from trends.
- Compare a product with its category in the same month (`CATEGORY_NET_SENTIMENT`),
  not with an absolute bar: Reddit reviews lean positive overall (net sentiment
  about +0.63).
- A post that reviews several products contributes one row per product; count
  posts with `COUNT(DISTINCT POST_URL)`.
- Products renamed between markets carry the latest catalogue name (see the
  barcode rule), so an older post may use the older name in `PRODUCT_MENTION`.

### FCT_PRODUCT_LAUNCH — every product launch, by province

One row per **product × province launch**, all brands, from **April 2025
onward**: Ontario, British Columbia, Quebec, New Brunswick, Nova Scotia and
Prince Edward Island. ~4,300 rows. Refreshed daily around 11:00 Eastern; every
row is recomputed, so the day-N columns fill in as a launch ages.

How a launch is dated: each province's cannabis wholesaler publishes a product
catalogue (OCS, BC Cannabis Stores, SQDC, Cannabis NB, NSLC, PEI Cannabis). A
launch is the day the product first appears in that catalogue, checked against
what store shelves show:

- Usually stores list the product on or after the catalogue date (Ontario about
  two weeks after, the other provinces within a few days). The launch is the
  catalogue date.
- When at least 3 stores in the province listed it 15 to 90 days **before** the
  catalogue entry, the catalogue was late: the launch is that shelf date
  (`LAUNCH_DATE_SOURCE` = `first_shelf`).
- When 3 stores carried it more than 90 days before, it is not a launch (a new
  barcode or size of an old product, or a re-listing) and has no row.

**Accuracy.** Checked on a sample of 297 catalogue entries (50 per province, all
25 in Prince Edward Island): 96–100% of sampled rows per province were real
launches, and every product the rule set aside as old was indeed old. This was an
internal, AI-assisted check, not an independent audit; small provinces have small
samples (Prince Edward Island: 10 decided rows). Manitoba is not covered.
Quebec covers the SQDC products already in this dataset's catalogue, about ten
launches a month.

**One product, several launches.** The province is the launch grain: a
product already sold in Ontario that reaches BC launches again in BC, with its
own date and rollout. For a Canada-wide count, keep `LAUNCH_SCOPE = 'new_to_canada'`.

| Column | Type | Description |
|---|---|---|
| PROVINCE_STATE | text | Province of the launch (ON, BC, QC, NB, NS, PE) |
| PC_ID | int | Product → `DIM_PRODUCT` |
| BRAND_ID | int | → `DIM_BRAND_PRODUCER` |
| CATALOG_ON | date | First day the product appeared in the province's wholesaler catalogue |
| LAUNCH_ON | date | Launch date (see above) |
| LAUNCH_DATE_SOURCE | text | `catalog` or `first_shelf` |
| FIRST_SHELF_ON | date | Day the product reached its 3rd store in the province (one or two stray early listings are ignored). NULL: not on a shelf yet |
| DAYS_TO_FIRST_SHELF | int | `FIRST_SHELF_ON − LAUNCH_ON`. A few days negative when stores were slightly ahead of the catalogue |
| LAUNCH_SCOPE | text | `new_to_canada`, or `provincial_expansion` when the product was listed in another province more than 30 days before |
| FIRST_SHELF_ELSEWHERE_ON | date | Week the product was first listed in another province (any province we observe, Manitoba stores included). NULL if never |
| LAUNCH_NOVELTY | text | `new_format` when the brand already sold the same base product more than 14 days before (a pre-roll of an existing flower, a multipack, a milled or hash version, an all-in-one of an existing cartridge...); else `new_product` |
| FORMAT_OF_PC_ID | int | For `new_format`: the earlier product it is a format of → `DIM_PRODUCT` |
| DAYS_OBSERVED | int | Days of rollout observed so far, capped at 90 |
| DOORS_LISTED_D7 … _D90 | int | Stores listing the product on day 7, 14, 28, 56, 90 after launch (see *Doors* below). NULL until that day is reached |
| DOORS_IN_STOCK_D7 … _D90 | int | Same, stores reporting a positive stock quantity. **Undercounts**: stores that report available / unavailable without quantities never count (about half of BC's stores, a third of Ontario's). Prefer DOORS_AVAILABLE_* |
| DOORS_AVAILABLE_D7 … _D90, DOORS_AVAILABLE_LATEST | int | Stores where the product was purchasable (in stock, or reported available) on day N, same 7-day rule. **The distribution number** (v1.9) |
| DOORS_LISTED_LATEST, DOORS_IN_STOCK_LATEST | int | Doors on the last observed day (≤ 90) |
| LEAD_PCV_ID | int | The pack size most stores carried in its first 28 days on shelf → `DIM_PRODUCT_VARIANT` |
| LAUNCH_UNIT_PRICE | decimal | Median observed unit price of that pack in the province over its first 28 days on shelf (dollars, observed sales only, tax basis of the province) |
| SUBCATEGORY_MEDIAN_UNIT_PRICE | decimal | Median observed unit price of the same subcategory and pack format in the province, launch month |
| PRICE_INDEX | decimal | `LAUNCH_UNIT_PRICE / SUBCATEGORY_MEDIAN_UNIT_PRICE`: 1.10 = launched 10% above its segment |
| FIRST_MOVER_BANNERS | text | The first three banners to list it in the province, in order. Chains with fewer than 3 **active** stores in the province read `Independent`; no store is named |
| REFRESHED_AT | timestamp | Last rewrite (UTC) |
| CATALOG_TO_SHELF_DAYS | int | Days from the catalogue entry to the 3rd store (negative when stores listed it first). Ontario ~16, Prince Edward Island ~51, others 0–3 |
| SHELF_DAYS_OBSERVED | int | Days observed since the 3rd store, capped at 90. NULL: not on shelves yet |
| DOORS_AVAILABLE_SHELF_D28 | int | Stores where it was available on any of days 22–28 **after its 3rd store**: the verdict's measure, free of wholesale timing |
| TYPICAL_SHELF_D28_P25, _MEDIAN, _P75, _LAUNCHES | decimal, int | The same measure over comparable launches: same province and subcategory (its category when fewer than 5), reached 3 stores in the trailing year, **this launch left out** |
| TYPICAL_SEGMENT, TYPICAL_LEVEL | text | What the typical launch covers (`dried flower` / `all flower`; `subcategory` / `category`) |
| LAUNCH_VERDICT | text | `well_ahead` (above p75), `ahead` (above the median), `in_line` (within 10% of the median, at least 3 stores), `behind`, `well_behind` (below p25); `too_early` (under 28 days on shelves), `not_on_shelves` (fewer than 3 stores), `no_benchmark` (Prince Edward Island, thin segments) |
| METHOD_VERSION | text | `v2-availability-shelf28` |

**`FCT_PRODUCT_LAUNCH_CURVE`** holds the rollout day by day: one row per launch
× `DAY_SINCE_LAUNCH` (0 to 90, observed days only) with `CLOSING_ON`,
`DOORS_LISTED`, `DOORS_IN_STOCK` and (v1.9) `DOORS_AVAILABLE`. ~335,000 rows.

**`V_PRODUCT_LAUNCH_BENCHMARK`** (secure view, SQL only; not in the Fabric feed)
is the typical rollout: per province × category × subcategory × day since
launch, the 25th percentile, median and 75th percentile of doors listed, in
stock and (v1.9) available, over the launches of the trailing year that reached a shelf and have 90
days observed. `BENCHMARK_LEVEL = 'category'` rows (SUBCATEGORY NULL) cover the
whole category for thin subcategories. Groups with fewer than 5 launches are
left out, and Prince Edward Island (5 stores observed) has no benchmark.

Reading rules:

- **Doors.** A store counts on day N when it listed (or had in stock) the
  product on at least one day of N−6 to N. Stores are not read every day, so a
  single-day count would jump around.
- Compare a launch with its province's benchmark, not across provinces: store
  counts differ (Quebec's ~110 SQDC stores all carry most launches; Ontario's
  2,400 private stores do not).
- `PRICE_INDEX` compares the lead pack with the same pack format. Unusual packs
  (party packs, sample sizes) can read far from 1; the 5th to 95th percentile
  runs about 0.76 to 1.66.
- Prices in Quebec, New Brunswick and Nova Scotia include sales tax, as
  elsewhere in this dataset. Within a province the index is unaffected.

### RPT_AVAILABILITY_WEEKLY — weekly distribution summary

The quickest answer to *"where is my product?"*: per product variant × province ×
ISO week (Monday start). ~4.3M rows. In Quebec every store reports quantities,
so `STORES_AVAILABLE` equals `STORES_IN_STOCK` there; the SQDC online store counts
as one store, like a physical one.

| Column | Type | Description |
|---|---|---|
| PCV_ID | int | Product variant |
| PROVINCE_STATE | text | Province (ON, AB, BC, …) |
| WEEK_START | date | Monday of the ISO week |
| STORES_LISTING | int | Stores that **carried** the product that week: the pair was `IS_LISTED` on at least one day of the week. Includes short sell-outs (up to 21 observed days); excludes products a store stopped carrying and menu entries that were never stocked (see below) |
| STORES_IN_STOCK | int | Stores with a known positive quantity at least one day that week |
| IN_STOCK_RATIO | decimal | STORES_IN_STOCK / STORES_LISTING. Healthy active products typically run 0.45–0.80 |
| STORES_AVAILABLE | int | Stores where the product was purchasable at least one day that week: in stock, or reported available by a store that never reports quantities. **The distribution number** — it is never lower than STORES_IN_STOCK |

**Which stores count as listing a product.** A store's menu can show a product it
has never carried (scraper residue, a placeholder page) or one it stopped carrying
months ago (sold out, discontinued, still on the page). Since v1.6 a (store,
product) pair counts as a listing only while it had stock or a real sale in its
last 21 observed days: it starts on its first day of evidence and ends 21 observed
days after its last one, and a restock brings it back. A week where no store
carried the product has no row. The rule looks only backwards, so a restock today
never changes weeks already published.

### V_SALES_DAILY_ENRICHED — pre-joined sales view (no joins needed)

`FCT_SALES_DAILY` with every dimension already joined, for quick ad-hoc queries
and BI tools that prefer one flat object. **View, not a table** — it does not
appear in the Fabric/Iceberg feed; query it directly in Snowflake. Same grain,
trust horizon and provisional-window semantics as `FCT_SALES_DAILY`; rows whose
product isn't in the consolidated catalog (NULL `PCV_ID`) are excluded.

| Column | Comes from |
|---|---|
| CLOSING_ON, MASTER_STORE_ID, SALES_UNITS, SALES_DOLLARS, UNIT_PRICE, IS_SYNTHETIC | FCT_SALES_DAILY |
| STORE_PROVINCE_STATE, STORE_NAME | DIM_MASTER_STORE |
| BRAND | DIM_BRAND_PRODUCER |
| PRODUCT_NAME, CATEGORY, SUBCATEGORY, SUB_SUBCATEGORY, IS_CRAFT, GROWING_PROVINCE | DIM_PRODUCT |
| FORMAT | DIM_PRODUCT_VARIANT |
| EOD_QTY | FCT_INVENTORY_DAILY (NULL on synthetic/estimate days) |

> `STORE_SALES_TIMELINE_DAILIES` is a **deprecated alias** of this view, kept
> temporarily for older queries — use `V_SALES_DAILY_ENRICHED`.

---

## Dimension tables

### DIM_MASTER_STORE — one row per retail store (~4,900 incl. historical)

| Column | Type | Description |
|---|---|---|
| MASTER_STORE_ID | int | Key. Stores are deduplicated: one row per physical location even when it appears on multiple platforms |
| STORE_NAME / CHAIN_NAME | text | Store and banner name |
| PROVINCE_STATE / CITY / ADDRESS / POSTAL_CODE / COUNTRY | text | Location |
| LATITUDE / LONGITUDE | decimal | Geocoordinates |
| IS_ACTIVE | boolean | FALSE = store closed (its provincial retail license is no longer active) or merged away upstream. Historical rows are kept so facts always find their store — filter `IS_ACTIVE` for the current store universe (~4,700) |

Quebec stores are the SQDC's: `CHAIN_NAME = 'SQDC'`, one row per store. The SQDC
online store (`SQDC - Z -En ligne`) is a row too, with no address or coordinates:
exclude it from maps and radius queries, keep it in sales and distribution.

### DIM_PRODUCT — one row per product (~21,800)

| Column | Type | Description |
|---|---|---|
| PC_ID | int | Key ("product consolidation" — the same product listed across many stores/platforms resolves to one row) |
| PRODUCT_NAME | text | Product name |
| BRAND_ID | int | Join DIM_BRAND_PRODUCER |
| PARENT_CATEGORY | text | `cannabis` or `accessories` |
| CATEGORY / SUBCATEGORY / SUB_SUBCATEGORY | text | Standardized taxonomy (see conventions) |
| THC_MIN/MAX, CBD_MIN/MAX | decimal | Potency range. **Units vary by category**: % for flower/vapes, mg for edibles/beverages |
| PLANT_TYPE | text | indica dominant / sativa dominant / hybrid / blend |
| STRAIN_NAME | text | Strain ("street name") where known |
| GROWING_PROVINCE | text | Where grown (lowercase province name), where reported |
| IS_CRAFT | boolean | Flagged craft in the OCS catalog |

### DIM_PRODUCT_VARIANT — one row per pack/size of a product (~27,500)

| Column | Type | Description |
|---|---|---|
| PCV_ID | int | Key — this is what the facts join on |
| PC_ID | int | Parent product |
| FORMAT | text | Pack/size as sold, e.g. `3.5g`, `10x0.35g`, `4 x 355ml` |
| GTIN / SKU / GTIN_CASE | text | Primary barcode, retailer SKU, case-level barcode (90% / 85% / ~20% coverage). Historical aliases live in DIM_VARIANT_IDENTIFIER |
| PACK_COUNT | int | Parsed from FORMAT (e.g. 10 for `10x0.35g`) |
| UNIT_SIZE | decimal | Parsed unit size; the unit (g/ml/mg) is in the FORMAT text |

### DIM_VARIANT_IDENTIFIER — barcode/SKU lookup bridge (~53,000)

One row per (variant, identifier), including **every historical alias**. Join your
own GTIN or SKU list on `IDENTIFIER_VALUE` to land on `PCV_ID`.

| Column | Type | Description |
|---|---|---|
| PCV_ID | int | Product variant |
| IDENTIFIER_TYPE | text | `GTIN`, `SKU`, or `GTIN_CASE` |
| IDENTIFIER_VALUE | text | The identifier |

### DIM_SALES_TAX — sales tax included in the dollars (14 provinces/territories)

The sales tax already inside `SALES_DOLLARS`, by province and period. Join it
to put every province on the same pre-tax basis (the basis Statistics Canada
uses):

```sql
SELECT m.PROVINCE_STATE, DATE_TRUNC('month', f.CLOSING_ON) AS month,
       SUM(f.SALES_DOLLARS)                             AS sales_dollars,
       SUM(f.SALES_DOLLARS / (1 + t.INCLUDED_TAX_RATE)) AS sales_dollars_ex_tax
FROM FCT_SALES_DAILY f
JOIN DIM_MASTER_STORE m USING (MASTER_STORE_ID)
JOIN DIM_SALES_TAX t
  ON t.PROVINCE_STATE = m.PROVINCE_STATE
 AND f.CLOSING_ON BETWEEN t.VALID_FROM AND t.VALID_TO
GROUP BY 1, 2;
```

| Column | Type | Description |
|---|---|---|
| PROVINCE_STATE | text | Province/territory code; joins `DIM_MASTER_STORE.PROVINCE_STATE` |
| VALID_FROM / VALID_TO | date | Period the row applies to (inclusive; `9999-12-31` = current) |
| TAX_INCLUDED | boolean | Menu prices, and so `SALES_DOLLARS`, include sales tax |
| INCLUDED_TAX_RATE | decimal | Tax rate inside `SALES_DOLLARS` (0.14975 = 14.975%); 0 where prices are pre-tax |
| TAX_NAME | text | e.g. `GST 5% + QST 9.975%`, `HST 14%` |
| RETAILER | text | Who sets the price display (SQDC, NSLC, Cannabis NB, private retail, …) |
| BASIS_VERIFIED | boolean | TRUE where the retailer's price display was confirmed (QC, NS, NB); FALSE where "pre-tax" is our reading of the menus |
| NOTE | text | e.g. Nova Scotia's HST cut from 15% to 14% on 2025-04-01 |

Quebec, Nova Scotia and New Brunswick include tax (Nova Scotia has two rows,
before and after its April 2025 HST cut); every other province and territory
is pre-tax. Multi-buy deals in New Brunswick are not visible on menus and are
not corrected here.

### DIM_BRAND_PRODUCER — one row per brand (~2,250)

| Column | Type | Description |
|---|---|---|
| BRAND_ID | int | Key |
| BRAND | text | Standardized brand name |
| BRAND_RAW | text | Brand name as scraped |
| PARENT_BRAND | text | Parent brand where applicable (sparse) |
| PRODUCER | text | Verified licensed producer behind the brand |
| PRODUCER_GROUP | text | Parent-company rollup (e.g. Tweed → Canopy Growth) |
| ATTRIBUTION_SOURCE / CONFIDENCE / VERIFIED_AT | text/date | Provenance of the brand→producer mapping |

### DIM_PRODUCT_SUPPLIER — who supplies each product, by province (~38,800)

*New in v2.1.* A brand's producer depends on the province. The SQDC buys Back Forty from
Origine Nature and General Admission from Rose Science Vie; the OCS lists Auxly and WestLeaf.
`DIM_BRAND_PRODUCER` gives one national answer per brand. This table gives, for each product
variant, the supplier **in each province where a source tells us**, plus the national answer.

| PROVINCE | SOURCE | Who it is |
|---|---|---|
| `QC` | `sqdc_list` | The producer the SQDC buys the product from, from the SQDC's own producer list, matched per product (no brand-name matching). Covers 99.99% of Québec dollars. |
| `ON` | `ocs` | The OCS supplier of record (the same field as `FCT_REGULATOR_MONTHLY_SALES.SUPPLIER`). When a product changed supplier, the current one. |
| `CA` | as in `DIM_BRAND_PRODUCER` | The national brand owner, through the product's brand. Use it for national views and for every other province (no other regulator reports a supplier). |

| Column | Type | Description |
|---|---|---|
| PCV_ID | int | → `DIM_PRODUCT_VARIANT` |
| PROVINCE | text | `QC`, `ON` or `CA` (national). One row per (PCV_ID, PROVINCE) |
| SUPPLIER | text | Display name. Québec names come lower case from the SQDC list and are title-cased here, which lowers acronyms ("Cbd" for "CBD"): restore them if you display names |
| SUPPLIER_RAW | text | The name exactly as the source spells it |
| SOURCE | text | `sqdc_list`, `ocs`, or the national row's attribution source |
| SHARE_OF_DOLLARS | number | The chosen supplier's share of the product's dollars (QC: 36 months; ON: 12 months). Below 1 when a product was split or changed supplier; NULL for `CA` |
| REFRESHED_AT | timestamp | Last rebuild |

**Supplier of record vs brand owner.** They differ on purpose for licensed and contract-made
brands: Sherbinskis is owned by Final Bell and supplied to the OCS by CannaPiece. For a
province's market (who you compete with on that shelf) use the province's row; for national
league tables use `CA`.

```sql
-- Québec sales by the producer the SQDC buys from, last 12 months
SELECT ps.SUPPLIER, ROUND(SUM(f.SALES_DOLLARS)) AS dollars
FROM FCT_SALES_DAILY f
JOIN DIM_MASTER_STORE m ON m.MASTER_STORE_ID = f.MASTER_STORE_ID AND m.PROVINCE_STATE = 'QC'
LEFT JOIN DIM_PRODUCT_SUPPLIER ps ON ps.PCV_ID = f.PCV_ID AND ps.PROVINCE = 'QC'
WHERE f.CLOSING_ON >= DATEADD(month, -12, CURRENT_DATE())
GROUP BY 1 ORDER BY 2 DESC;
```

Refreshed daily at 10:00 ET.

---

## Worked examples

**Brand share by province, last full month:**
```sql
SELECT m.PROVINCE_STATE, b.BRAND, SUM(f.SALES_DOLLARS) AS dollars
FROM FCT_SALES_DAILY f
JOIN DIM_MASTER_STORE m   ON m.MASTER_STORE_ID = f.MASTER_STORE_ID
JOIN DIM_PRODUCT_VARIANT v ON v.PCV_ID = f.PCV_ID
JOIN DIM_PRODUCT p         ON p.PC_ID = v.PC_ID
JOIN DIM_BRAND_PRODUCER b  ON b.BRAND_ID = p.BRAND_ID
WHERE f.CLOSING_ON >= '2026-06-01' AND f.CLOSING_ON < '2026-07-01'
GROUP BY 1, 2 ORDER BY 1, dollars DESC;
```

**Where is my product listed and in stock (by week)?**
```sql
SELECT r.WEEK_START, r.PROVINCE_STATE, r.STORES_LISTING, r.STORES_AVAILABLE, r.STORES_IN_STOCK, r.IN_STOCK_RATIO
FROM RPT_AVAILABILITY_WEEKLY r
JOIN DIM_VARIANT_IDENTIFIER i ON i.PCV_ID = r.PCV_ID
WHERE i.IDENTIFIER_TYPE = 'GTIN' AND i.IDENTIFIER_VALUE = '00628942536052'
ORDER BY r.WEEK_START DESC, r.PROVINCE_STATE;
```

**Units left on shelves, per store, latest reading:**
```sql
SELECT m.STORE_NAME, m.PROVINCE_STATE, f.CLOSING_ON, f.EOD_QTY, f.IS_AVAILABLE
FROM FCT_INVENTORY_DAILY f
JOIN DIM_MASTER_STORE m       ON m.MASTER_STORE_ID = f.MASTER_STORE_ID
JOIN DIM_VARIANT_IDENTIFIER i ON i.PCV_ID = f.PCV_ID
WHERE i.IDENTIFIER_TYPE = 'GTIN' AND i.IDENTIFIER_VALUE = '00628942536052'
  AND f.CLOSING_ON >= CURRENT_DATE - 14   -- always bound the date: the table is large
  AND f.IS_LISTED                         -- stores that still carry it
QUALIFY ROW_NUMBER() OVER (PARTITION BY f.MASTER_STORE_ID ORDER BY f.CLOSING_ON DESC) = 1
ORDER BY m.PROVINCE_STATE, f.EOD_QTY DESC NULLS LAST;
```
`EOD_QTY` NULL with `IS_AVAILABLE` TRUE is a store that reports availability
but not quantities. Drop the `QUALIFY` line to get the daily series instead.

**National sales by province, safe end date for both sources:**
```sql
SELECT m.PROVINCE_STATE, SUM(f.SALES_DOLLARS) AS dollars
FROM FCT_SALES_DAILY f
JOIN DIM_MASTER_STORE m ON m.MASTER_STORE_ID = f.MASTER_STORE_ID
WHERE f.CLOSING_ON >= '2025-01-01'   -- national trends start here (see Coverage)
  AND f.CLOSING_ON <= (SELECT MIN(COMMON_THROUGH) FROM REFRESH_STATUS WHERE FACT_TABLE = 'FCT_SALES_DAILY')
GROUP BY 1 ORDER BY dollars DESC;
```

**Rest of Canada only (the pre-v1.4 scope):** add `AND m.PROVINCE_STATE <> 'QC'`.
**Quebec only, full history:** `AND m.PROVINCE_STATE = 'QC'`, any date from 2020-06-23.

**My Ontario market share within a subcategory (regulator data):**
```sql
SELECT YEAR_MONTH,
       SUM(IFF(SUPPLIER = 'YOUR PRODUCER NAME', SALES_DOLLARS, 0)) / SUM(SALES_DOLLARS) AS share
FROM FCT_REGULATOR_MONTHLY_SALES
WHERE CATEGORY = 'flower'
GROUP BY 1 ORDER BY 1;
```

## FAQ

**Why did last week's numbers change?** The trailing 7 days are provisional —
stores report late and estimates get replaced by observations. Day 8+ is stable.

**Why are units fractional?** Short feed gaps are filled with modeled estimates
(marked `IS_SYNTHETIC`), which carry decimals. Round after aggregating.

**Why don't the OCS figures match OCS's own reports?** Our regulator table is
*sell-through* (sold onward); many OCS reports count *units distributed*
(shipped). Shipped ≠ sold.

**Why don't BBFYB's provincial totals match Statistics Canada's?** They measure
different things: StatCan estimates every cannabis store's sales excluding sales
tax, per calendar month; BBFYB estimates the stores it observes, at shelf prices
(tax included in Quebec, Nova Scotia and New Brunswick), per day. Where BBFYB
covers nearly every store (Quebec, Nova Scotia, New Brunswick, PEI) the two agree
within a few percent once tax is removed; elsewhere BBFYB sees part of the market.

**Can I compute a brand's market share with `FCT_STATCAN_RETAIL_MONTHLY`?** No.
Divide within BBFYB's own facts instead (a brand's dollars over all dollars in
the same stores and days): that is share of the observed market, which is what
the portal and the API report. StatCan's totals are for sizing the market around
it.

**How many units are left on the shelf at end of day? Can I work it out from
sales?** Not from sales: `FCT_SALES_DAILY` only has units sold, and deliveries
refill the shelf in between. Use `EOD_QTY` in `FCT_INVENTORY_DAILY`, the
quantity each store reported at close. About 14% of stores outside Quebec
report availability only; for them `EOD_QTY` is NULL and `IS_AVAILABLE` says
whether the product could be bought.

**Why does a product show more "listing" stores than stores in stock?**
`STORES_LISTING` counts stores that carry the product, including short
sell-outs (up to 21 observed days without stock); `STORES_AVAILABLE` is the ones
where a shopper could buy it that week, and `STORES_IN_STOCK` the subset that
also reports a positive quantity. Since v1.6, products a store stopped carrying
no longer count at all, so the gap between listing and available is recent
sell-outs, not old menu entries.

**Why did listing counts drop in October 2026?** v1.6 stopped counting products
stores no longer carry (see the changelog). Brand store counts barely moved for
brands that are still selling, because a store keeps counting as long as any one
of the brand's products is carried; per-product listing counts fell most for old
formats and discontinued products.

**Why did national totals jump in September 2026?** Quebec joined the daily
facts (v1.4). Rest-of-Canada numbers did not change; filter
`PROVINCE_STATE <> 'QC'` to reproduce what you had.

**Why is there Quebec data before 2025 but nothing else?** Quebec is observed
from 2020-06-23; the rest of Canada from 2025-01-01. National figures before
2025 are Quebec only.

**Why does Quebec have no gummies or disposable vapes?** The SQDC does not sell
them. The category is empty in Quebec because the assortment is.

**A product's name looks wrong for its date.** One barcode is one product
over time and carries its latest official name (see Conventions), so older
sales show today's name.

**Why do product-level totals differ slightly from market totals?** About 10% of
sales dollars belong to products not yet mapped into the consolidated catalog
(`PCV_ID` is NULL). They're included in the facts for honest totals but drop out
of product joins.

**What do THC/CBD numbers mean?** The range printed on the product. Units follow
the category: percentages for inhalables, milligrams for ingestibles.

---

## Access

Direct-SQL clients query with the `BBFYB_GOLD_READER` role: read-only on every
GOLD object (new objects are covered automatically), no access to anything
else in the account. Each client gets their own user and XS warehouse. Microsoft Fabric consumers receive the same tables via the feed
instead and don't need Snowflake credentials.

---

## Changelog

**v2.1 — 2026-10** — *one new table; nothing existing changes.*
- **New `DIM_PRODUCT_SUPPLIER`**: who supplies each product variant, by province. Québec from the SQDC's
  own producer list, Ontario from the OCS supplier of record, plus the national brand owner (`CA`). Use the
  province's row for that province's market, `CA` for national views.

**v1.9 — 2026-10** — *new columns; existing columns keep their meaning.*
- **`FCT_PRODUCT_LAUNCH` / `_CURVE`: availability.** New `DOORS_AVAILABLE_*` columns count stores where a
  launch was purchasable. The in-stock columns count only stores reporting a stock quantity and undercount
  BC and Ontario heavily; prefer availability for distribution.
- **A verdict measured from the shelf:** `DOORS_AVAILABLE_SHELF_D28` (28 days after the 3rd store), its typical
  range among comparable launches (`TYPICAL_SHELF_D28_*`, the launch itself left out) and `LAUNCH_VERDICT`.
- `CATALOG_TO_SHELF_DAYS`, `SHELF_DAYS_OBSERVED`, `METHOD_VERSION`. First-mover banners count active stores
  only; the subcategory price median is now the month the launch was priced (first shelf month).
- The accuracy note now says what the check was: internal and AI-assisted, not an independent audit.
- **`V_PRODUCT_LAUNCH_BENCHMARK`**: adds availability percentiles.

**v1.8 — 2026-10** — *new tables and view; nothing existing changed.*
- **`FCT_PRODUCT_LAUNCH`**: every product launch by province since April 2025,
  all brands, dated on the provincial wholesaler catalogue and checked against
  store shelves (graded 96–100% correct per province). Labels each launch as
  new to Canada or a provincial expansion, and as a new product or a new
  format of an existing one, with doors at day 7 to 90, launch price against
  its segment and the first banners to list it.
- **`FCT_PRODUCT_LAUNCH_CURVE`**: the day-by-day rollout of each launch.
- **`V_PRODUCT_LAUNCH_BENCHMARK`**: the typical rollout per province and
  subcategory, to read a launch against (SQL only).

**v1.7.1 — 2026-10** — *more reviews linked; no column changed.*
- **`FCT_SOCIAL_REVIEW_MENTION`**: 61 brand spellings reviewers use (MTL,
  PSF, B40, Fraser Valley, Wola...) now resolve to their brand, and reviews
  that name a rotating product line as the brand (Gros Plaisirs, Menu
  Cantine, Chopper's Pick) link to that product. New `MATCH_METHOD` value
  `product_line`. Past rows were re-linked: counts for those brands rise.

**v1.7 — 2026-10** — *new table and view; nothing existing changed.*
- **`FCT_SOCIAL_REVIEW_MENTION`**: Reddit reviews linked to catalogue
  products, one row per post × reviewed product, July 2021 onward, with the
  author's sentiment per product and a supporting quote. Graded ≥ 93% correct
  product links. A signal of what enthusiasts say, not a survey; see its
  section, including the June–September 2026 gap.
- **`V_SOCIAL_REVIEW_MONTHLY`**: product × month review counts and net
  sentiment with the category benchmark (SQL only).


**v1.6.1 — 2026-10** — *new table; nothing existing changed.*
- **`DIM_SALES_TAX`**: the sales tax included in `SALES_DOLLARS` by province
  and period (Quebec 14.975%; Nova Scotia 15%, then 14% from 2025-04-01; New
  Brunswick 15%; other provinces 0). Divide by `1 + INCLUDED_TAX_RATE` to put
  provinces on one pre-tax basis. See its section for the query.


**v1.6 — 2026-10** — *history restated; takes effect when announced.*
- **`FCT_INVENTORY_DAILY` joins the feed**: daily end-of-day quantity on hand
  per store × product variant, Quebec from 2020-06-22 and the rest of Canada
  from 2025-01-01. It answers "how many units are left on the shelf", which
  daily sales cannot. See its section and the new worked example.
- `FCT_INVENTORY_DAILY`: new **`IS_LISTED`** column: the store carries the
  product that day (available or sold at least once in the last 21 observed
  days). Every existing row keeps its values; the column is filled for the whole
  history.
- `RPT_AVAILABILITY_WEEKLY`: `STORES_LISTING` counts only stores that carry the
  product (listed on at least one day of the week), replacing v1.3's "from the
  first week it was available". Products a store stopped carrying, sold-out menu
  entries and discontinued products stop counting; a restock brings them back.
  Weeks with no carrying store have no row. `IN_STOCK_RATIO` rises because its
  denominator no longer includes stale listings.
- **Restatement:** the weekly table was rebuilt over its full history.
  Measured on 2026-09-28: store-product listings fall about 76 % in Quebec and
  44 % in the rest of Canada. For brands still selling, store counts move
  little (for most established brands under 10 %), because a store keeps
  counting while any of the brand's products is carried; brands no longer sold
  anywhere in a province leave that province's counts.

**v1.5.1 — 2026-09-30** — *documentation only; no data changed.*
- New *Known gaps* section: New Brunswick 2025-05-13 to 2026-04-26 (4-6 of ~29
  stores collected) and Nova Scotia 2026-02-16 to 2026-04-22 (no data). Exclude
  those periods from trends and year-over-year comparisons; store counts step
  up on each recovery date.


**v1.5 — 2026-09-29** — *new tables; nothing existing changed.*
- **`FCT_STATCAN_RETAIL_MONTHLY`**: Statistics Canada's official monthly
  cannabis-retailer sales for Canada, the provinces and territories and 9
  metropolitan areas, October 2018 onward, with year-over-year change, share of
  Canada and sales per resident. Market context only (see its section).
- **`DIM_GEO_POPULATION`**: StatCan quarterly population estimates.
- New *Sales tax* convention: Quebec, Nova Scotia and New Brunswick sales dollars
  include sales tax (they always have; now documented).
- New FAQ entries on comparing BBFYB with StatCan.


**v1.4.1 — 2026-09-28**
- `DIM_MASTER_STORE.IS_ACTIVE` now also turns FALSE when a store's provincial
  retail license is no longer active, not only when the store is removed.
  About 68 stores flip (66 in Ontario, one closed SQDC store); none had sold
  anything in the previous 30 days. Active-store counts drop by that much.
- `REFRESH_STATUS` / `V_REFRESH_STATUS` times are Eastern (they were Pacific).

**v1.4 — 2026-09-25** — *Quebec added; rest-of-Canada rows unchanged.*
- `FCT_SALES_DAILY`, `FCT_INVENTORY_DAILY`, `RPT_AVAILABILITY_WEEKLY` now include
  **Quebec**: every SQDC store and the SQDC online store, daily, from
  **2020-06-23** (inventory and weekly availability from 2020-06-22). +40.9M sales
  rows ($4.78B), +190M inventory rows, +401K weekly availability rows.
- `DIM_MASTER_STORE` gains the 116 SQDC master stores; Quebec products sit on the
  shared catalog (same `PC_ID` / `PCV_ID` / taxonomy as the rest of Canada).
- **National totals change**: they now include Quebec, and before 2025-01-01 they
  are Quebec only. Filter `PROVINCE_STATE <> 'QC'` to reproduce earlier numbers.
- New sections: *Coverage* (census vs panel, per-year Quebec share), Quebec
  freshness (SQDC rectifications), the barcode/latest-name rule, the Quebec
  2020–2022 understatement.
- Row counts refreshed throughout; the `IS_SYNTHETIC` share restated
  (rest of Canada 18–20% of dollars, not 3–8%).

**v1.3.1 — 2026-09** — *no data change.*
- Each daily fact is now refreshed per source (rest of Canada, Quebec) behind a
  readiness check; a late or incomplete source is held at its previous version
  instead of being written partially. New table **`REFRESH_STATUS`** (and view
  `V_REFRESH_STATUS`) publishes per-source freshness and a `COMMON_THROUGH` date.

**v1.3 — 2026-09** — *history restated; see the note at the end.*
- `FCT_INVENTORY_DAILY`: new **`IS_AVAILABLE`** column. Stores whose menu only
  reports available/unavailable (never a quantity; ~14% of stores, ~45% of BC)
  now have a row on every available day, with `EOD_QTY` NULL. About +18% rows.
  Use `IS_AVAILABLE` for distribution; `IS_IN_STOCK` still means a known
  positive quantity.
- `RPT_AVAILABILITY_WEEKLY`: new **`STORES_AVAILABLE`** column (the distribution
  number). A (store, product) pair now counts as a listing only from the week it
  was first available or first sold a real unit; never-stocked menu residue is
  excluded. `STORES_LISTING` falls for most products as a result, and
  `IN_STOCK_RATIO` rises mechanically because its denominator shrinks — that
  rise is arithmetic, not a market change.
- **Restatement:** both tables were rebuilt over their full history (2025-01-01
  onward), so weekly distribution numbers you exported before this date will not
  match a re-export. This is a one-time correction of what was counted, not a
  change to any store's observed data. The "day 8+ is stable" promise under
  *Freshness & revisions* applies to observations, not to a definition change
  like this one; we will announce any future one the same way.

**v1.2 — 2026-07-10**
- `FCT_SALES_DAILY`: new **`UNMAPPED_CATEGORY` / `UNMAPPED_SUBCATEGORY`**
  columns classify the ~10% of dollars whose product isn't in the consolidated
  catalog (std-taxonomy values; `accessories` covers hardware/apparel/nicotine
  and is the recommended exclusion for cannabis-only totals). Unmapped rows are
  now one per store × day × classification (previously one collapsed row per
  store × day). Mapped rows unchanged.

**v1.1 — 2026-07-10**
- `DIM_MASTER_STORE`: now includes closed/merged stores with **`IS_ACTIVE`**
  (new column) so historical sales always join; row count ~4,600 → ~9,400.
  Filter `IS_ACTIVE` for the current store universe.
- New **`V_SALES_DAILY_ENRICHED`** pre-joined sales view (internal);
  `STORE_SALES_TIMELINE_DAILIES` deprecated → alias, removed in Phase 4.
- `RPT_AVAILABILITY_WEEKLY` history rebuilt to include closed stores (+1.2% rows).

**v1.0 — 2026-07-09** — initial release.
