# [PFS] Driving Set

The Feature Store manages and serves features to machine-learning models. The
Product Feature Store holds the features describing **a product/item sold at a
location/store on a day**, and different streams use those data points, or
features, to produce predictions.

To enable this, the Feature Store assembles every possible item-location
combination that could be used across the different machine-learning projects.
That set of item-location combinations is called the **driving set**, and it is
built once to serve many use cases. In the Product Feature Store a **past** and a
**future** driving set are both available: models are trained on the past driving
set, and predictions are made on the future driving set. Each project filters the
driving set down to the rows it needs using `DRIVINGSET_SOURCE_CDE`.

---

## Driving set Sources

**Daily count of item_loc combinations in the driving set: ~27M**

The driving set is built in the order below, and each priority is a **top-up** of
the codes before it: a step only inserts the item-location-day rows that no
earlier step has already claimed.

Three things to know before reading the table:

1. **`SOURCE_CDE` records the first bucket to claim a row, not every bucket the
   row qualifies for.** An item that both sold last week and has a purchase order
   due next month is stamped `SALES`, never `PURCH_ORD`. So counting `PURCH_ORD`
   rows does not tell you how many items have a future PO — it tells you how many
   have one *and nothing else*. The buckets are a partition, not a set of
   independent flags.
2. **Store 7323 is excluded from all seven steps** (`LOC_IDNT != 7323`).
3. **Priorities 4, 6 and 7 are leading-edge only.** They are gated by
   `conditional_run` and stamp a single day (`DAY_IDNT = @run_to_day_idnt`) rather
   than iterating over the date range, so they never appear on historical PFS days.

### The codes are stored short and published long

The driving-set table stores short codes; three are renamed when the ILD/IZD
feature tables are built. **Query the driving-set table with the short codes and
the feature tables with the long ones** — filtering the raw table for `'SALES'`
returns nothing.

| Stored in `ftr_pfs_feature_drivingset.SOURCE_CDE` | Published as `DRIVINGSET_SOURCE_CDE` |
|---|---|
| `SLS_AGG` | `SALES` |
| `COL_SLS` | `COL_ORDERED_NOT_PICKED` |
| `PEND_RANGE` | `PENDING_RANGE` |
| `FP_RANGE`, `NON_FP`, `PURCH_ORD`, `SOLD_LY` | unchanged |

### Sources

Schemas resolve as `@source_schema` = `biw`, `@target_schema` = `aa_smkt_pfs_v3`,
`@golden_db` = `uc_0004_prd_prd_catalog.aa_smkt_pfs_gold`,
`@source_schema_bigwave` = `bigwave_source`.

| Priority | Driving Set Source code | PFS / PFuS | Source system | Source schema tables | High-level filters | ~Record counts | Logic / Notes |
|---|---|---|---|---|---|---|---|
| **1** | `SALES`<br>*(stored as `SLS_AGG`)* | Used in both PFS & PFuS | BIW sales aggregate | `biw.SLS_ITEM_LD_AGG`<br>`biw.time_day_dm`<br>`biw.cml_asis_prod_level2_dm` | `CHAIN_IDNT = 1`<br>`CML_CHANNEL_TYPE_CDE IN ('BM','CO')`<br>sold within a rolling **2 calendar months** (`add_months(day_dt, -2)`)<br>liquor (`DIV_IDNT = '10'`) excluded **only where `MIN_CHAN_TYPE = 'CO'`**<br>`LOC_IDNT != 7323` | 19M~ | Indicates that an item has been sold from the associated store within the last **2 calendar months** (59–62 days, not a fixed 60). Includes both BM and COL. Liquor is excluded from **Coles Online only** — an in-store liquor sale still enters the driving set. This is the **only** bucket that populates `LATEST_SALE_DAY_IDNT`. Reads the ASIS product table for performance, because a handful of items were flipped Liquor→Overhead in error and corrected the next day. |
| **2** | `COL_ORDERED_NOT_PICKED`<br>*(stored as `COL_SLS`)* | Used in both PFS & PFuS | Coles Online transactions | `biw.col_sls_transaction_ild_atm`<br>`biw.time_day_dm`<br>`biw.cml_asis_prod_level2_dm` | `ORDER_COUNT_FLAG = 'Y'`<br>`F_ORDER_QTY > 0`<br>`F_SLS_PICKED_QTY = 0`<br>same rolling 2-month window<br>liquor excluded **unconditionally**<br>`LOC_IDNT != 7323` | 48K~ | Indicates that an item has been ordered via the Coles Online channel but not picked. The reasons may be lack of availability, etc. Excludes liquor items. Anything not picked as part of the `SALES` bucket (Priority 1) is topped up with this category. This data lets AA track the **original demand** rather than going only by the sales records. `LATEST_SALE_DAY_IDNT` is NULL here — there was no sale. |
| **3** | `FP_RANGE` | **Used in both PFS & PFuS** *(corrected — see notes)* | **Lettuce** for warehouse-supplied lines; **ITEM_LOC_TRAITS (RMS)** for direct-to-store lines | Via `aa_smkt_pfs_v3.ftr_pfs_feature_fp_range`.<br>**Lettuce arm:** `biw.FP_ORD_ITEM_LOC_TRAITS_A_REF`, `biw.CML_FP_ORDER_ITEM_LEVEL_DM`, `biw.PROD_ITEM_DM`, `biw.ORG_LOC_ATTR_REF`, `biw.ORG_LOC_DM`, `biw.time_day_dm`<br>**Direct-to-store arm:** `biw.PROD_ITEM_LOC_TRAITS_CURR_REF`, `biw.PROD_ITEM_LOC_TRAITS_HIST_REF`, `biw.CML_PROD_LEVEL2_DM` | **At this step:** `FP_RANGED_IND = 'Y'`; `DAY_IDNT` in run window; **`EXISTS`** — the location must already be in the driving set for that day; `LOC_IDNT != 7323`<br>**Direct-to-store arm:** `DIV_IDNT = '6'`, `SOURCE_METHOD_CDE = 'S'`, `PACK_SELLABLE_CDE <> 'N'`, `ITEM_DESC NOT LIKE '%NON SCAN SALES%'`, `SBCLASS_DESC <> 'FINANCE'`, `ITEM_STATUS_CDE IN ('A','C')`<br>**Lettuce arm:** `DM_RECD_CURR_FLAG = 'Y'`, `ITEM_LEVEL = TRAN_LEVEL`, `PACK_SELLABLE_CDE != 'N'`, `DIV_IDNT = '6'`, `LOC_TYPE_CDE = 'R'`; availability from `ON_SALE_START_DT` / `ON_SALE_END_DT` | 9K~ | Includes only **fresh** items that are part of the active range per the Lettuce source system (for items coming from Warehouse). In addition, items on direct-to-store lines are sourced from ITEM_LOC_TRAITS. Tops up the driving set for item-loc-day combinations not yet covered by `SALES` or `COL_ORDERED_NOT_PICKED`. **This data is available only from 14-Jan-2020.**<br>Two corrections to the earlier draft: (a) this step has no `conditional_run`, so it runs for every day in the window and lands in **PFS as well as PFuS** — consistent with the feature-table matrix below, which lists `FP_RANGE` under `FTR_PFS_ITEM_LOC_DAY`; (b) the filter list quoted for this bucket is the **direct-to-store arm only** — the Lettuce arm filters `PROD_ITEM_DM` instead, as shown in the filters column.<br>Note the `EXISTS` gate: a store with no rows yet for that day gets nothing topped up here. |
| **4** | `PENDING_RANGE`<br>*(stored as `PEND_RANGE`)* | PFuS (16 & 54 weeks) | **Lettuce** for fresh warehouse lines; **CKB Pending Layout** for non-fresh; **ITEM_LOC_TRAITS (RMS)** for fresh direct-to-store and non-fresh current range | **At this step:** `biw.cml_prod_level2_dm`, `biw.prod_item_mixed_pallet_dim`<br>Via `aa_smkt_pfs_v3.ftr_pfus_feature_pending_range`.<br>**Fresh:** `biw.FP_ORD_ITEM_LOC_TRAITS_A_REF`, `biw.CML_FP_ORDER_ITEM_LEVEL_DM`, `biw.PROD_ITEM_DM`, `biw.ORG_LOC_ATTR_REF`, `biw.ORG_LOC_DM`, `aa_smkt_pfs_gold.store_level_open_close_date`<br>**Non-fresh:** `biw.ITEM_LOC_LAYOUT_PENDING_TXN`, `biw.PROD_ITEM_LOC_TRAITS_CURR_REF`, `bigwave_source.ITEMATTRIBUTES` | **At this step:** sellable item via the `cml_prod_level2_dm` type-2 window; `NOT (PACK_SIMPLE_CDE='S' AND PACK_SELLABLE_CDE='N')`; exclude mixed-SKU pallet items; `div_idnt NOT IN ('10','-1')`; `LOC_IDNT != 7323`<br>**Upstream:** `ITEM_STATUS_CDE IN ('A','C')` on the ITEM_LOC_TRAITS arms; `LAYOUT_PENDING_STATUS_CDE IN ('A','Z')` on pending layout; final bound `ON_SUPPLY_DT <= run_to AND OFF_SUPPLY_DT >= run_from` | 370K~ | Items that will be ranged in and out in future for 16 & 54 weeks. Tops up the driving set for item-loc-day combinations not yet covered by `SALES`, `COL_ORDERED_NOT_PICKED` or `FP_RANGE`.<br>Items range **in** when the on-supply / off-supply dates fall in the range from today to 16 or 54 weeks, and range **out** when they do not.<br>**Fresh** — Lettuce is the master, with history from 14-Jan-2020, and covers non-direct-to-store lines; any future range change on FP non-direct lines is sourced from the Lettuce feed. FP items on direct-to-store lines are sourced from ITEM_LOC_TRAITS (RMS), and their future range changes come from there. `ON_SALE_START_DT` drives it. Covers current trading stores and new stores opening within 1 year.<br>**Non-fresh** — future range changes are read from **CKB Pending Layout and ITEM_LOC_TRAITS**, with `CML_ON_SUPPLY_DT` / `CML_OFF_SUPPLY_DT` driving it. Covers current trading stores and new stores opening within 1 year. Pending layout accumulates the latest record from last Sunday through Saturday: on Sunday ITEM_LOC_TRAITS is updated in BIW so the live range folds into ITEM_LOC_TRAITS, and when a layout goes live it drops out of pending range — those dropped records are still needed, and layouts go live two weeks ahead of the implementation date.<br>**Correction on precedence:** the earlier draft had ITEM_LOC_TRAITS as primary and PENDING LAYOUT as secondary. In the SQL it is the other way round where both exist — the ITEM_LOC_TRAITS current-range arm carries `NOT EXISTS (... pending layout ...)`, so a pending-layout record wins and is stamped `LOG_SOURCE = 'CKB PENDING RANGE'`. For fresh, Lettuce wins over both.<br>**International items get a widened window here** — see the International section below.<br>The "one day a week (Saturday)" cadence comes from the PFuS job schedule; the driving-set config itself only says `conditional_run` (leading edge), so it is not verifiable from this config. |
| **5** | `NON_FP` | PFS + PFuS | ITEM_LOC_TRAITS (RMS) — non-fresh, not sold for 2 months | `biw.PROD_ITEM_LOC_TRAITS_CURR_REF`<br>`biw.PROD_ITEM_LOC_TRAITS_HIST_REF`<br>`biw.time_day_dm`<br>`biw.cml_prod_level2_dm`<br>`biw.prod_item_mixed_pallet_dim` | `on_supply_dt >= day_dt − 2 months`<br>`day_dt BETWEEN on_supply_dt AND NVL(off_supply_dt,'9999-12-31')`<br>**`item_status_cde = 'A'`** (active only)<br>type-2 bounding `day_dt BETWEEN RECD_LOAD_DT AND RECD_CLOSE_DT`<br>`NOT (PACK_SIMPLE_CDE='S' AND PACK_SELLABLE_CDE='N')`<br>exclude mixed-SKU pallet items<br>`div_idnt NOT IN ('9','10','-1','6')`<br>**`EXISTS`** location gate<br>`LOC_IDNT != 7323` | 550K~ | **Non-FP items ONLY.** Items that are **on supply now but have not sold in the last two months** — the on-supply date falls inside the trailing 2-month window and the day sits inside the on-supply/off-supply range. Includes only Active items. Excludes No Division Desc, LIQUOR, OVERHEAD and FRESH.<br>Tops up the driving set for item-loc-day combinations not yet covered by `SALES`, `COL_ORDERED_NOT_PICKED`, `FP_RANGE` or `PENDING_RANGE`.<br>Note this bucket accepts `'A'` **only**, where the range buckets accept `('A','C')` — so a discontinued-but-still-on-supply non-fresh item is not picked up here. It also carries an `EXISTS` gate on the location, like `FP_RANGE`. |
| **6** | `PURCH_ORD` | PFuS (16 & 54 weeks) | Fresh Produce purchase orders | Via `aa_smkt_pfs_v3.ftr_pfus_feature_fp_purch_ord`:<br>`biw.CML_FP_PURCH_ORDER_ILD_DM`<br>`biw.CML_PURCH_ORDER_DM`<br>`biw.FP_ORD_ITEM_LOC_TRAITS_A_REF`<br>`biw.ORG_LOC_ATTR_REF`<br>`biw.ORG_LOC_DM`<br>`biw.TIME_DAY_DM` | **At this step:** `DISTINCT`; `LOC_IDNT != 7323`<br>**Upstream:** `ORDER_DUE_IDNT` between the run-from and run-to day — the SQL comment reads *"Pick only the POs that are due in PFuS 113 days"*; `PO_TYPE_CDE IN ('LTX','LET')` — warehouse and cross-dock only, exception POs excluded; `FROM_TO_LOC_TYPE_CDE IN ('SUPP_DC','DC_DC')`; `FP_PO_VERSION_STATUS_CDE <> 'C'` — cancelled POs excluded; latest PO version only, by `ROW_NUMBER` per item × DC | 44K~ | **FP items ONLY.** Identifies whether a Fresh Produce item has a Purchase Order due in the future, for 16 & 54 weeks — forward visibility for items that are in season and will be delivered to stores. Only the latest instance of an Item-DC record is considered. Excludes cancelled POs.<br>Tops up the driving set for item-loc-day combinations not yet covered by `SALES`, `COL_ORDERED_NOT_PICKED`, `FP_RANGE`, `PENDING_RANGE` or `NON_FP`.<br>**Correction:** the earlier draft said "forward visibility of 10 days". There is no 10-day constant in the SQL — POs are picked across the whole PFuS window, which the code comments describe as 113 days for the 16-week stream.<br>The PO is held at item × DC grain, then exploded to every store mapped to that source warehouse via `CML_SOURCE_WH_ID` and cross-joined to the day range. |
| **7** | `SOLD_LY` | PFuS (16 & 54 weeks) — **FRESH PROD only** | BIW sales aggregate, via the sales feature table | `aa_smkt_pfs_v3.ftr_pfs_feature_sales` (which itself reads `biw.SLS_ITEM_LD_AGG`)<br>`biw.cml_asis_prod_level2_dm`<br>`biw.TIME_DAY_DM` | `div_idnt = 6` — Fresh Produce only<br>sold between (run-to date − **13 months**) and the run-to date<br>`LOC_IDNT != 7323` | 6M~ | All item-loc combinations that have sold within the **13 months up to** the PFuS creation date and are in no other driving-set bucket. Tops up the driving set for item-loc-day combinations not yet covered by `SALES`, `COL_ORDERED_NOT_PICKED`, `FP_RANGE`, `PENDING_RANGE` or `NON_FP`.<br>**Correction:** this is a 13-month **window ending at the run date**, not "sold 13 months prior" as a point in time.<br>Inherits the sales feature table's own filters (`CHAIN_IDNT = 1`, BM/CO only, liquor excluded).<br>PFuS ILD carries `SALES_LY_30_DAYS_IND`, `SALES_LY_14_DAYS_IND` and `SALES_LY_7_DAYS_IND` to specify which band the row belongs to — whether a sale happened in the last 30, 14 or 7 days respectively. |

Window lengths are config-driven, in `config/ftr_pfs_feature_drivingset_config.yaml`:

```yaml
drivingSetRollingWindowMonths: 2    # priorities 1, 2
drivingSetNonFreshRangeMonths: 2    # priority 5
drivingSetSeasonalMonths:      13   # priority 7
```

---

## Feature tables vs Driving set

| Table Name | Driving Set Sources |
|---|---|
| `FTR_PFS_ITEM_LOC_DAY` | `SALES`, `COL_ORDERED_NOT_PICKED`, `FP_RANGE`, `NON_FP` |
| `FTR_PFS_ITEM_ZONE_DAY` | `SALES`, `COL_ORDERED_NOT_PICKED` |
| `FTR_PFS_ITEM_ZONE_DAY_NSTORES` | `SALES`, `COL_ORDERED_NOT_PICKED` |
| `FTR_PFUS_ITEM_LOC_DAY` | `SALES`, `COL_ORDERED_NOT_PICKED`, `FP_RANGE`, `PENDING_RANGE`, `NON_FP`, `PURCH_ORD`, `SOLD_LY` |
| `FTR_PFUS_ITEM_ZONE_DAY` | `SALES`, `COL_ORDERED_NOT_PICKED`, `FP_RANGE`, `PENDING_RANGE`, `NON_FP`, `PURCH_ORD`, `SOLD_LY` |
| `FTR_PFUS_ITEM_ZONE_DAY_NSTORES` | `SALES`, `COL_ORDERED_NOT_PICKED`, `FP_RANGE`, `PENDING_RANGE`, `NON_FP`, `PURCH_ORD`, `SOLD_LY` |

The zone-level tables filter with
`drivingset_source_cde IN ('SALES','COL_ORDERED_NOT_PICKED')`. A `CASE`
expression enumerating every code also appears in those files — that is a shared
priority-ranking helper, not evidence that the other codes reach the table.

---

## Fresh vs Non-Fresh

### "Fresh" means division 6

There is no fresh flag column anywhere in the Feature Store. Throughout the
driving set, **fresh means `DIV_IDNT = '6'` — Fresh Produce**, read from
`cml_prod_level2_dm` / `cml_asis_prod_level2_dm` / `PROD_ITEM_DM.DIV_IDNT`.

This is narrower than the commercial sense of the word: **bakery, deli and meat
are not division 6** and are covered by the non-fresh paths.

| `DIV_IDNT` | Meaning | Treatment in the driving set |
|---|---|---|
| `'6'` | Fresh Produce | included or excluded explicitly, per bucket |
| `'10'` | LIQUOR | excluded wherever named |
| `'9'` | OVERHEAD | excluded (Priority 5) |
| `'-1'` | No Division Desc | excluded (Priorities 4, 5) |
| `'1'`–`'8'` | trading divisions | the accepted set |

### Which buckets a fresh or non-fresh item can reach

Four of the seven buckets are division-scoped, so fresh and non-fresh items reach
the driving set through largely separate routes.

| Priority | Division rule | Effect |
|---|---|---|
| 1 `SALES` | liquor anti-join only, and only for Coles Online | both |
| 2 `COL_ORDERED_NOT_PICKED` | liquor anti-join | both |
| 3 `FP_RANGE` | upstream filters `DIV_IDNT = '6'` in **both** arms | **fresh only** |
| 4 `PENDING_RANGE` | `div_idnt NOT IN ('10','-1')`, then split into separate fresh / non-fresh arms | both, different logic per arm |
| 5 `NON_FP` | `div_idnt NOT IN ('9','10','-1','6')` | **non-fresh only** |
| 6 `PURCH_ORD` | no explicit filter; the source tables are Fresh Produce feeds by construction | **fresh only** |
| 7 `SOLD_LY` | `div_idnt = 6` | **fresh only** |

So `FP_RANGE`, `PURCH_ORD` and `SOLD_LY` rows are fresh by definition, and
`NON_FP` rows are non-fresh by definition.

### Range comes from a different master depending on the class

| Item class | Master | Tables | Dates used |
|---|---|---|---|
| Fresh, warehouse-supplied | **Lettuce** | `biw.FP_ORD_ITEM_LOC_TRAITS_A_REF` + `biw.CML_FP_ORDER_ITEM_LEVEL_DM` | `ON_SALE_START_DT` / `ON_SALE_END_DT` — the business-confirmed availability dates |
| Fresh, direct-to-store (`SOURCE_METHOD_CDE = 'S'`) | **ITEM_LOC_TRAITS (RMS)** | `biw.PROD_ITEM_LOC_TRAITS_CURR_REF` | `ON_SUPPLY_DT` / `OFF_SUPPLY_DT` |
| Non-fresh | **CKB Pending Layout**, then ITEM_LOC_TRAITS (RMS) | `biw.ITEM_LOC_LAYOUT_PENDING_TXN`, then `biw.PROD_ITEM_LOC_TRAITS_CURR_REF` / `_HIST_REF` | `CML_ON_SUPPLY_DT` / `CML_OFF_SUPPLY_DT` |

**Lettuce always wins for fresh** — both non-Lettuce routes are guarded so they
cannot re-supply an item-location that Lettuce has already provided. Which master
supplied a row is recorded on `ftr_pfus_feature_pending_range.LOG_SOURCE`:

| `LOG_SOURCE` | Item class |
|---|---|
| `LETTUCE` | fresh, warehouse-supplied |
| `CKB PENDING RANGE` | non-fresh, pending layout |
| `RMS ILT - current range` | non-fresh, current range |
| `RMS ILT - current range for fresh direct-to-store` | fresh, direct-to-store |

---

## International vs Domestic

International items are the only class the driving set treats differently by item
**origin**, and Priority 4 `PENDING_RANGE` is the only bucket where that
treatment applies.

### How an international item is identified

An item is international when the CML forecast lead-time indicator is set:
`CML_FORECAST_LT_IND = 'Y'` on `bigwave_source.ITEMATTRIBUTES`. These are
long-lead-time imported lines, where range and supply decisions are made months
before the stock is available to sell.

The attribute sits at **item level** — there is no location or day dimension —
and only the latest snapshot of `ITEMATTRIBUTES` is read
(`BUSINESSEFFECTIVEDATE = max(BUSINESSEFFECTIVEDATE)`).

There is no separate domestic indicator. **Domestic is the default**: the flag is
resolved with `nvl(..., 'N')`, so an item with no `ITEMATTRIBUTES` record is
treated as domestic and follows the ordinary Priority 4 rules.

It is **not** derived from `ITEM_STATUS_CDE`, and it is neither an RMS nor a CKB
attribute — it comes from `ITEMATTRIBUTES` alone.

### How international items enter the driving set

Two routes supply Priority 4, in this precedence order. CKB Pending Layout is
evaluated first, and the ITEM_LOC_TRAITS route excludes any item-location that
pending layout has already supplied.

**1. Future range / pending changes — sourced from CKB Pending Layout**, when the
item has a pending range-in or range-out event within the 16- or 54-week future
window. Pending range-in (`LAYOUT_PENDING_STATUS_CDE = 'A'`) and pending range-out
(`'Z'`) are both captured, so items arriving into range and items leaving it are
equally visible.

This route is not international-specific — every non-fresh item with a pending
layout event is picked up the same way. It is listed first because it takes
precedence, so it is how an international item normally arrives whenever a range
change is genuinely pending.

**2. Current active range — sourced from ITEM_LOC_TRAITS (RMS)**, when the item is
actively ranged with no pending change at all: `ON_SUPPLY_DT` up to the run-to
date, `OFF_SUPPLY_DT` on or after the run-from date (a null off-supply date reads
as open-ended), and RMS status `ITEM_STATUS_CDE IN ('A','C')`.

**This branch is international-specific.** Domestic items reach the current-range
route only when a supply date actually falls *inside* the run window, or when the
store is a new store opening within the year. International items are admitted on
range **overlap** instead: it is enough that the range is open across the window.

### There is no sales-based restriction for international items

International items are not required to have sold. There is no sales-history
test, no recency window and no minimum volume on either route above:

> **If an international item is actively ranged at a store, it is included in the
> driving set for that store and day.**

This is the point of the rule. Priorities 1 and 2 are sales-driven over a
trailing two months, and Priority 5 requires the item to have come on supply
within the last two months. An imported line ranged nine months ago, selling
slowly or not yet selling at a given store, meets none of those. Without the
international branch it would drop out of the driving set entirely and the model
would have no row to forecast against — precisely the items where forward
visibility matters most, because replenishment must be committed a long way ahead.

### Boundaries

- **Non-fresh only.** The international branch sits on the non-fresh
  current-range route; the fresh direct-to-store route has no international
  branch, as Fresh Produce is short-lead by nature.
- **PFuS only.** Priority 4 lands in PFuS (16 and 54 week), not in historical PFS.
- **The flag widens; it never excludes.** No bucket drops an item for being
  international, and none drops one for being domestic.
- **`SOURCE_CDE` still records the first claim.** An international item that also
  sold recently is stamped `SALES`, not `PENDING_RANGE`. The international rule
  changes *whether a row exists*, not which bucket labels it.
- **The universal Priority 4 exclusions still apply**: liquor, no-division items,
  non-sellable simple packs, mixed-SKU pallet items and store 7323.

`INTERNATIONAL_FLAG` is also published as a feature column on prod_attrib, ILD,
IZD and the channel tables, where it gates the PRMG planned-end columns. Those are
feature-table rules, not driving-set rules.

---

## Does the driving set use ITEM_STATUS_CDE, RMS or CKB?

### `ITEM_STATUS_CDE` — used, but not consistently

It is applied in the ITEM_LOC_TRAITS-based routes only, and **the accepted value
set differs by bucket**:

| Where | Rule | Reading |
|---|---|---|
| Priority 5 `NON_FP`, both current and history arms | `item_status_cde = 'A'` | active only |
| Priority 3 `FP_RANGE`, direct-to-store arm | `ITEM_STATUS_CDE IN ('A','C')` | active, or discontinued but still ranged |
| Priority 4, non-fresh current range | `item_status_cde IN ('A','C')` | same |
| Priority 4, fresh direct-to-store current range | `item_status_cde IN ('A','C')` | same |
| Priority 4, on/off-supply date resolution | `NVL(ITEM_STATUS_CDE,'D') = 'A'` vs `!= 'A'` | **not a filter** — decides whether the date is taken from ITEM_LOC_TRAITS or from pending range |
| All Lettuce arms (Priorities 3-fresh, 4-fresh, 6) | not used | availability comes from `ON_SALE_START_DT` / `ON_SALE_END_DT` instead |

Values: `'A'` active / ranged in, `'C'` discontinued, `'D'` deleted (the `NVL`
default, treated as ranged out), `'-1'` unknown — nulled out downstream.

Do not confuse it with `LAYOUT_PENDING_STATUS_CDE` on the pending-layout feed,
which is a different column with its own values: `'A'` pending range **in**, `'Z'`
pending range **out**.

**The asymmetry to know about:** `NON_FP` accepts `'A'` only, so a
discontinued-but-still-on-supply non-fresh item that has not sold for two months
is **not** picked up there, while the same item passes the `('A','C')` test in the
pending-range and fp-range buckets.

### RMS — yes

`PROD_ITEM_LOC_TRAITS_CURR_REF` and `_HIST_REF` are the RMS ITEM_LOC_TRAITS
extracts (the code calls them "RMS ILT"). RMS is the primary range master for
non-fresh items and for fresh direct-to-store items, and the on/off-supply
resolution logic is explicitly aligned to RMS status.

### CKB — as a data feed yes, as a schema no

- **CKB data does reach the driving set**, through Priority 4: the non-fresh
  pending-layout route reads `biw.ITEM_LOC_LAYOUT_PENDING_TXN` — CKB-originated
  planogram / layout data landed into BIW — and stamps
  `LOG_SOURCE = 'CKB PENDING RANGE'`.
- **No driving-set step reads the `ckb` schema.**
  `ckb.cgl_impl_pog_item_store_mv` is referenced by exactly two files,
  `ftr_pfs_fsi_ckb_wrk_01.sql` and `_02.sql` — the FSI shelf-capacity / planogram
  features.
- **Naming trap:** the pending-layout query aliases a `biw` table as
  `pending_ckb`. Grepping for `ckb` will hit it and suggest a dependency on the
  `ckb` schema that does not exist.

---

## Summary of corrections applied to the earlier version of this page

| Earlier wording | What the SQL does |
|---|---|
| Source codes are `SALES`, `COL_ORDERED_NOT_PICKED`, `PENDING_RANGE` | Those are the **published** names on the feature tables. The driving-set table itself stores `SLS_AGG`, `COL_SLS`, `PEND_RANGE`. |
| "within the last 60 days" | A 2 calendar month window (`add_months(..., -2)`), so 59–62 days depending on the month. |
| "Excludes liquor items" (Priority 1) | Priority 1's liquor anti-join fires **only** where `MIN_CHAN_TYPE = 'CO'`. Liquor sold in store is retained. Priority 2 excludes liquor unconditionally. |
| `FP_RANGE` marked PFuS-only | It has no `conditional_run`, so it runs for every day in the window and lands in PFS too — consistent with the feature-table matrix, which lists it under `FTR_PFS_ITEM_LOC_DAY`. |
| The fresh filter list (`DIV_IDNT='6'`, `SOURCE_METHOD_CDE='S'`, `PACK_SELLABLE_CDE<>'N'`, `ITEM_DESC NOT LIKE '%NON SCAN SALES%'`, `SBCLASS_DESC<>'FINANCE'`, `ITEM_STATUS_CDE IN ('A','C')`) presented as applying to all of `FP_RANGE` | That list is the **direct-to-store arm only**. The Lettuce arm filters `PROD_ITEM_DM` instead: `DM_RECD_CURR_FLAG='Y'`, `ITEM_LEVEL = TRAN_LEVEL`, `PACK_SELLABLE_CDE != 'N'`, `DIV_IDNT = '6'`, plus `LOC_TYPE_CDE = 'R'`. |
| Priority 4 non-fresh: "ITEM_LOC_TRAITS is the primary source, PENDING LAYOUT the secondary" | Precedence is the other way round where both exist. The ITEM_LOC_TRAITS current-range arm carries `NOT EXISTS (... pending layout ...)`, so a pending-layout record wins. |
| `PURCH_ORD` "forward visibility of 10 days" | No 10-day constant exists in the SQL. POs are picked across the whole PFuS window — the code comment says *"POs that are due in PFuS 113 days"*. |
| `SOLD_LY` "sold 13 months prior" | A 13-month **window ending at the run date**, not a point in time. |
| Priority 5 "items that will be ranged in the future but on supply date in past" | On supply **within the last 2 months**, with the day inside the on-supply/off-supply range. |
| No mention of store exclusions | `LOC_IDNT != 7323` in all seven steps. |
| `FP_RANGE` and `NON_FP` described as plain top-ups | Both also carry an `EXISTS` gate requiring the **location** to already be in the driving set for that day. |
| `PENDING_RNAGE` in the feature-table matrix | Typo — `PENDING_RANGE`. |

---

## Where this is implemented

| Component | Path in `aaAzureProductFeatureStore` |
|---|---|
| Config (steps, windows, increments) | `config/ftr_pfs_feature_drivingset_config.yaml` |
| The seven steps | `sql/pfs_feature_drivingset_01_sales.sql` … `_07_sold_ly.sql` |
| Fresh range input | `sql/pfs_feature_fp_range_01_lettus.sql`, `_02_direct_to_store.sql` |
| Pending range input | `sql/pfus_feature_pending_range_01_fresh_lettus.sql`, `_02_nonfresh_pendinglayout_and_ilt.sql` |
| Future PO input | `sql/pfus_feature_fp_purch_ord_01.sql` |
| Table DDL | `ddl_AA_PRODUCT_FEATURE_STORES/01tables/FTR_PFS_FEATURE_DRIVINGSET.ddl` |
| Short → published code renaming | `sql/pfs_item_loc_day_03_main.sql`, `pfs_item_loc_day_channel.sql`, `pfus_item_loc_day_wrk2_01.sql`, `pfus_item_loc_day_channel_02_main.sql` |

**Table facts:** `aa_smkt_pfs_v3.ftr_pfs_feature_drivingset`, Delta, partitioned by
`DAY_IDNT`, primary key `ITEM_IDNT, LOC_IDNT, DAY_IDNT`. Columns: `DAY_IDNT`,
`ITEM_IDNT`, `LOC_IDNT`, `DAY_DT`, `LATEST_SALE_DAY_IDNT`, `FUTURE_FP_PO_IND`,
`SOURCE_CDE`, `RECD_LOAD_DT`, `PROCESS_KEY`. Backfilled from 2015-01-01 in
180-day increments with 5 parallel jobs.

**Record counts** in the Sources table are carried over from the earlier version
and have not been re-measured. To refresh:

```sql
SELECT SOURCE_CDE, COUNT(*) AS rows, COUNT(DISTINCT ITEM_IDNT) AS items
FROM   aa_smkt_pfs_v3.ftr_pfs_feature_drivingset
WHERE  DAY_IDNT = <day_idnt>
GROUP BY SOURCE_CDE
ORDER BY rows DESC;
```
