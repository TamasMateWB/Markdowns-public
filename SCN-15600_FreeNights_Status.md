# SCN-15600 – ALPP (Accor) FreeNights Promotions: status, tests and risks

_Status as of 19 Nov 2026. Sources: Jira epic SCN-15600, its child tasks SCN-15601 to SCN-15607, the analysis in SCN-15183, and the six open GitHub PRs listed below._

## 1. Summary

- **Goal:** OBC search prices for ALPP FreeNights promotions (ALPP API 1.1.3) must match what ALPP returns at CheckAvailability, Prebook and Book.
- **Where things stand:** the whole chain is deployed on **stage as prereleases** (Svalbard step 1 + step 2, obc-cache, Searcher, Consumer, ALPP Producer). **Nothing is merged.**
- **Dev results on stage (19/11):** all 8 ALPP scenarios match live ALPP CheckAvailability to the cent (rates APFLRA1/APFLRA3/APFLMRB1, rooms DBC/SAD, 1 and 2 adults). The APFLRA3 Prebook (2 adults) returns the same price in all 8 cases. Both checks were run again with Svalbard step 2 and gave the same results. Dev says it is **"Ready for QA"**.
- **Still open:** independent QA (SCN-15607, assigned to Tamás). This covers Book → Cancel, regression on MinMaxStay and HBSI, and freshness/invalidation. The production DB ALTER (SCN-15601) has not been run. One PR is still waiting for review (ALPP Producer #37). The NuGet still points to a prerelease.

## 2. How it works (final design)

ALPP sends free-night promotions as conditioned prices with `freeNightIndex`, a 0-based list that is already expanded. The conditioned price is the price of a free night only (in practice, the taxes). All other nights use the base price. The promotion applies only when the whole stay falls inside the promotion period, and it cannot be combined with another promotion: the cheapest whole-stay option wins.

```
ALPP push ─▶ Producer (reads freeNightIndex, marks require_all_days_stay)
          ─▶ Consumer (stores hotel_promotion.free_night_indexes, sends hotel_promotion invalidation)
          ─▶ DB ─▶ Svalbard (stores hotel_promotion_id on each price/breakdown row)
          ─▶ obc-cache (in-memory hotel_promotion table by id; resolves rows when a blob arrives;
                        lists promotions once per hotel; rows point to them with PromotionIndex)
          ─▶ Searcher (prices the whole stay: free nights at the promo price, other nights at base; cheapest total wins)
```

This design was agreed with Andrés Soliño in obc-cache#226 on 4 Nov. It replaced two earlier designs (indexes in static data, then a list by code in ARI).

## 3. Changes per task and repo

| Task | Component / PR | Change | Jira status | PR status | On stage |
|---|---|---|---|---|---|
| SCN-15601 | DB, all hotel shards (manual script) | `hotel_promotion.free_night_indexes smallint[] NULL`. Stage column recreated as `smallint[]`. Reference DDLs in obc-cache and obc-api updated | In Progress | n/a | ✅ applied. **Production pending** |
| SCN-15602 | webbeds-obc **#395** (Common NuGet + Consumer + CopyData) | `HotelPromotion.FreeNightIndexes` (`[Key(3)] short[]`). Consumer selects, inserts and updates the column. A promotion is updated when the indexes change (ordered compare, null = empty). Sends a `hotel_promotion` invalidation on insert/update | QA in progress | Open. Approved (reyesandrea, alejandro-navarro). Copilot left one open note (insert + id lookup not atomic, already existed before this PR). Case-sensitive update by code moved to **SCN-15614** | `p1.0.302-…FreeNightIndexes.1` |
| SCN-15603 | webbeds-obc-alpp **#37** (ALPP Producer) | Reads `freeNightIndex` (empty → null). Free-night promotion ⇒ `require_all_days_stay`. One promotion per code. Fixes a null access on a conditioned price with no promotion (now skipped and no longer stored as a base price) | QA in progress | Open, **no approval yet**. Depends on the **prerelease** NuGet `1.0.237-SCN-15602…`: must be switched to the final 1.0.237 after #395 is released | `p0.0.33-…FreeNightIndex.0` |
| SCN-15604 step 1 | svalbard **#98** | `PromotionId` on price and price breakdown rows (v1: fields 17/19; v2: new last column 21/23, presence bit 9). Still resolves promotions as today. Fix: LOS prices built from the amount JSON keep their promotion (before, they went out as base prices). Goldens unchanged | QA in progress | Open, approved (Andrés) | `p0.2.11-…PromotionIdInPriceRows.0` |
| SCN-15604 step 2 | svalbard **#99** | Stops resolving promotions: rows carry the id only, rows with an unknown promotion are no longer dropped, the 30-minute promotions map is removed | QA in progress | **Draft**, based on #98. Do not merge until obc-cache #226 runs on every production shard | `p0.2.11-…StopResolvingPromotions.0` |
| SCN-15605 | obc-cache **#226** | `HotelPromotionsData` engine table (LoadOnStart + Terraform). Refreshed by the invalidation, which expires rows at once. One shared read per hotel (an error is never treated as "no promotions"). BlobProcessor fills code and flag at ingest and drops rows with an unknown id. `AriHotelV2.Promotions`, `AriPriceRowV2.PromotionIndex`, `HotelDetailsV2.Promotions`, `FeeDataV2.PromotionIndex`. Rows keep `PromotionCode`/`AllDaysStay` for one release | QA in progress | Open, approved (Andrés ×2). +4,767/−486, 55 files | `p0.2.9-…WholeStayPromotionsAriV2.1` |
| SCN-15606 | obc-searcher **#129** | Reads promotions by index (a row with no index keeps its own code). Free-night rows are excluded from the cheapest price per night. Whole-stay comparison per occupancy. Applies only with a row on every night plus MinStay. Fees follow the plan. On a free night, taxes come from the promotion only, and a 0 charges nothing | QA in progress | Open, approved (Andrés), main merged in | `p0.5.0-…FreeNightsWholeStayPricing.0` |
| SCN-15607 | Stage QA, OBC + WBC | See section 4 | QA in progress (Tamás) | n/a | n/a |

**Production release order:** DB ALTER (all shards) → Consumer #395 (then release the NuGet) → Svalbard #98 (successor preload, shard by shard) → obc-cache #226 (all shards, Terraform for LoadOnStart) → Searcher #129 → ALPP Producer #37 → tell ALPP to start sending free nights → Svalbard #99.

**Optional, not started:** OBC Exporter impact analysis; showing the free night indexes in the OBC Viewer Raw viewer.

## 4. Tests and expected results

### 4.1 Test data

- Hotel **9174** (OBC id 400, hub 519595), Market 724 (ES). Natasha ran an on-demand push on stage for **02/01/2027 to 06/04/2027**.
- Rates: **APFLRA3** (main), APFLRA1 (no promotion), APFLMRB1 (in ALPP availability, but not in the ARI loaded for these scenarios). Rooms DBC and SAD.
- Searcher request: `POST http://<searcher>/v1/availability` with `{"ProviderCode":62,"ChainCode":"ACCOR","CheckInDate":"…","CheckOutDate":"…","HotelIds":["9174"],"Rooms":[{"PaxAge":[30]}]}`. `totalPrice` is the net price.
- WBC: send each CheckAvailability twice, with `IsObcRequest: true` (OBC) and `false` (live ALPP). In Prebook, add `"RatePlanId": "APFLRA3"`.

### 4.2 Scenarios: 2 adults, APFLRA3, DBC (reference values from dev, 19/11)

| # | Stay (in → out) | Checks | Expected promotion | Expected net (OBC = ALPP CheckAvail = Prebook) | Gross at Prebook |
|---|---|---|---|---|---|
| 1 | 02–07 Jan 2027 | 1 free night inside the promo (MinStay 5) | Promo_FN_Case1_Min5_D2 | **452.40** | 520.00 |
| 2 | 15–21 Jan | Stay starts outside the promo: full-stay rule, no promo | none | **678.60** | 780.00 |
| 3 | 25 Jan–01 Feb | Repeated free nights | Promo_FN_Case3_Min6_D2_Rep | **452.40** | 520.00 |
| 4 | 10–17 Feb | Free night beats the 10% discount | Promo_FN_Case4_Min4_D2 | **678.60** | 780.00 |
| 5 | 22 Feb–01 Mar | Discount beats the free night | Promo_Disc_Case5_Min6_10p | **475.02** | 546.00 |
| 6 | 05–12 Mar | Free night not eligible, discount applies | Promo_Disc_Case6_Min6_10p | **911.82** | 1037.61 |
| 7 | 20–25 Mar | Free nights with MaxStay 5 vs 15% | Promo_FN_Case7_Min2_Max5_D2Rep | **453.77** | 514.10 |
| 8 | 01–06 Apr | Repeated D1 free nights vs 20% Min3/Max6 | Promo_FN_Case8_Min2_D1Rep | **453.77** | 514.10 |

Notes:
- The "Total = 410€ / 612€ …" values in Natasha's scenario descriptions are illustrative. **Compare against ALPP's live response**, not those values.
- Nightly check example, case 7 APFLRA3: 139.59 / **5.00** / 139.59 / **5.00** / 139.59 (free nights at indexes 1 and 3).
- The ALPP spec descriptions do not always match the messages (for example "3rd night" when the index is 1). Use `freeNightIndex` as the reference, never the promotion name.

### 4.3 QA checklist (SCN-15607)

| # | Area | Test | Expected result | Dev status |
|---|---|---|---|---|
| 1 | DB | `hotel_promotion` rows for 9174 | Free-night promos have `free_night_indexes` set and `require_all_days_stay = true`. Discount promos have NULL indexes | ✅ (dev, 09/11) |
| 2 | DB | Column check (SCN-15601 verification query) on each shard | `ARRAY / _int2 / YES` | Stage ✅, prod ⏳ |
| 3 | Svalbard | `GET /debug/p_<hotel>_<day>_v2` | Step 2: rows carry `PromotionId` only, with no code or all-days flag | ✅ |
| 4 | obc-cache | `/v2/ari/daily`, `/v2/ari/los`, `/v2/hotel-details` for 9174 | `Promotions[]` listed once per hotel with indexes. Each promoted row or fee has `PromotionIndex` | ✅ daily + hotel-details. **LOS not reported** |
| 5 | obc-cache | `/stats` → `HotelPromotionsData` | Table loaded on start and populated | ⏳ |
| 6 | Searcher | Cases 1–8, 1 and 2 adults, every rate/room | Net = live ALPP CheckAvailability to the cent | ✅ |
| 7 | WBC | Search → CheckAvailability | No price change | ✅ |
| 8 | WBC | Prebook (2 adults, `RatePlanId` APFLRA3) | Same price as CheckAvailability | ✅ |
| 9 | WBC | **Book** for a sample of cases (at least 1, 3, 5, 7) | Booking created with the expected total | ⏳ **not done** |
| 10 | WBC | **Cancel**, plus cancellation policy and penalties on free-night stays | Policies and penalties correct (penalty based on the promoted total) | ⏳ **not done** |
| 11 | Regression | ALPP MinMaxStay promotion hotel | Same price as before / as production | ⏳ **not reported** |
| 12 | Regression | HBSI hotel with free nights | Unchanged | ⏳ **not reported** |
| 13 | Regression | Non-ALPP hotel without promotions (ARI v2 + hotel-details) | Identical response to production | ⏳ |
| 14 | Freshness | Create or change a promotion on stage | Consumer sends the `hotel_promotion` invalidation. Change visible in search after the obc-cache blob TTL (~108–132 min), without waiting for Svalbard's 30-minute map | ⏳ |
| 15 | Edge | Stay longer than the listed indexes / stay shorter than MinStay / stay partly outside the promo | Free nights only at the listed indexes. Promo not applied below MinStay or outside the period | Covered by unit tests. Case 2/6 on stage |
| 16 | Taxes | Free night with a percentage or amount tax | On a free night, only the promotion's taxes apply (0 charges nothing) | Unit test (9174: 400.90, 0.45 per tax) |

### 4.4 Automated tests in the PRs (all reported passing)

- **#395:** MessagePack/JSON round trip with and without indexes and from the old version. Update and no-update rules. Invalidation on insert, update and failure. A manual run against real PostgreSQL (an empty array is stored as `{}`, NULL as NULL).
- **#37:** new snapshot `price_message_with_free_nights` built from the real 9174 ARI. Producer 19/19, StaticImport 23/23.
- **#98 / #99:** v1 ↔ v2 round trip with the id (including 0), no column when there are no promotions, ErrLayout on a wrong-length column, LOS copies keep the promotion. #99 tests the four load paths with a fake SQL driver. Goldens unchanged.
- **#226:** table, invalidation and shared-read error tests (fake driver). Ingest fill/drop/untouched for v1 and v2. ARI daily/LOS and hotel-details index tests. Mutation testing. Not covered: the Postgres cycle and handlers end to end (stage only).
- **#129:** ALPP scenarios, per-occupancy decisions, a 0-priced free night, fee plan, reading by index (0, stale code, missing, out of range), MinMaxStay unchanged, mutation testing.

## 5. Risks

| # | Risk | Impact | Mitigation / owner |
|---|---|---|---|
| R1 | **Release order broken.** Svalbard #99 deployed before obc-cache #226 is on every shard, or the Producer deployed before obc-cache and the Searcher | Promoted prices served as **base prices** (underpricing), or free-night prices used every night | Strict order (section 3). #99 stays draft until then |
| R2 | **No easy rollback after step 2.** obc-cache cannot go back to a pre-#226 version while id-only blobs exist. Rolling Svalbard back to step 1 does not fix blobs already stored | A bad obc-cache release after step 2 can only be fixed forward, or by flushing blobs | Keep a soak period between obc-cache and step 2. Prepare a blob-flush runbook |
| R3 | **Production DB ALTER not done** (manual, per shard, no migration tool) | Consumer and obc-cache queries fail if any shard is missing the column | Run and verify on **every** shard before the Consumer deploy (Andrea) |
| R4 | **Prerelease NuGet in #37** | Production built against an unreleased package | Switch to the final 1.0.237 after #395 is released. #37 still needs a review |
| R5 | **Terraform for `LoadOnStart`** in obc-cache | Without it, the table loads lazily: a cold-start latency spike or a load error could drop promoted rows | Deploy obc-cache through the Terraform workflow and check `/stats` |
| R6 | **Freshness / lost invalidation.** New or changed promotions show after the blob TTL (~2 h). Up to ~5 h if an invalidation is lost. A brief race between `price_rule` and `hotel_promotion` in the same batch | Search shows the old price → price change at CheckAvailability (no wrong booking: ALPP decides the price live) | Accepted by design. Tell ALPP and the business |
| R7 | **Search ≠ ALPP when OBC and ALPP choose different promotions** (stale data, or ties) | Price change between Search and CheckAvailability | Covered by the 8 scenarios. Watch price-change metrics after go-live |
| R8 | **ALPP QA environment issues:** Prebook with 1 adult fails (883 / 28). APFLRA1 Prebook fails (402 / 28). APFLMRB1 is in availability but not in the ARI, so it is picked as the cheapest when `RatePlanId` is missing | Blocks part of the E2E testing; could hide real issues | With ALPP via Natasha. Test with 2 adults and `RatePlanId`. Re-test 1 adult once fixed |
| R9 | **Regression not yet proven on stage:** MinMaxStay, HBSI free nights, hotels with no promotions, LOS endpoint | Silent price change for existing content | QA items 4, 11–13 |
| R10 | **Percentage fees counted twice** when a fee group splits between base and promotion | Not ALPP (taxes are sent as amounts). Would affect a future provider that sends percentages | Known limit, documented in #129 |
| R11 | **Promotion update by code is case-sensitive** (existing behaviour) | Indexes not updated if the code case changes | Follow-up **SCN-15614** |
| R12 | **Consumer insert + id lookup is not atomic** (existing pattern) | A failure after insert means no invalidation until the cache entry expires | Accepted. Would need its own card |
| R13 | **Exact price ties** after step 2: promoted rows sort by id instead of code | Could change which promotion is chosen on an exact tie | Low; noted in #99 |
| R14 | **Spec ambiguity:** a free night combined with an *unconditional* discount (non-free nights at the discounted or the undiscounted base?) was never tested; ALPP sends every promo as conditioned | Possible mismatch if ALPP sends unconditional promos in production | Open question from SCN-15183. Monitor |
| R15 | **Large change set** (obc-cache alone is +4.7k lines), and part of it is only tested with fakes | Edge cases in production | Stage QA plus a gradual shard rollout |

## 6. Next steps

1. QA: run section 4.3 items 4 (LOS), 5, 9–14 on stage. Record the results in SCN-15607.
2. Get #37 reviewed. After #395 merges, release the NuGet and switch #37 to the final 1.0.237.
3. Run the SCN-15601 ALTER on all production shards and verify it.
4. Plan the production rollout in the order above, with a soak period before Svalbard step 2. Notify ALPP only after the Producer is live.
5. Before go-live, fill in the epic's Acceptance Criteria (still the template text) and decide on the optional items (Exporter, Viewer).

## Sources

- Jira: [SCN-15600](https://webbeds.atlassian.net/browse/SCN-15600) · [SCN-15601](https://webbeds.atlassian.net/browse/SCN-15601) · [SCN-15602](https://webbeds.atlassian.net/browse/SCN-15602) · [SCN-15603](https://webbeds.atlassian.net/browse/SCN-15603) · [SCN-15604](https://webbeds.atlassian.net/browse/SCN-15604) · [SCN-15605](https://webbeds.atlassian.net/browse/SCN-15605) · [SCN-15606](https://webbeds.atlassian.net/browse/SCN-15606) · [SCN-15607](https://webbeds.atlassian.net/browse/SCN-15607) · [SCN-15183](https://webbeds.atlassian.net/browse/SCN-15183)
- PRs: [webbeds-obc#395](https://github.com/WebBeds/webbeds-obc/pull/395) · [webbeds-obc-alpp#37](https://github.com/WebBeds/webbeds-obc-alpp/pull/37) · [svalbard#98](https://github.com/WebBeds/svalbard/pull/98) · [svalbard#99](https://github.com/WebBeds/svalbard/pull/99) · [obc-cache#226](https://github.com/WebBeds/obc-cache/pull/226) · [obc-searcher#129](https://github.com/WebBeds/obc-searcher/pull/129)
- Spec: [ALPP - Versions - FreeNights in Promotion 1.1.3](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/4983324688/ALPP+-+Versions+-+FreeNights+in+Promotion+1.1.3)
