# SCN-15600 – ALPP (Accor) FreeNights Promotions: status, tests and risks

_Status as of 19 Nov 2026 (DB check results added 15:30 UTC). Sources: Jira epic SCN-15600, its child tasks SCN-15601 to SCN-15607 (including the SCN-15607 Acceptance Criteria), the analysis in SCN-15183, the six open GitHub PRs listed below, the ALPP 1.1.3 spec wiki page (sections 2.1 and 4.2.1), and the stage DB exports in the `SQL` folder._

## 1. Summary

- **Goal:** OBC search prices for ALPP FreeNights promotions (ALPP API 1.1.3) must match what ALPP returns at CheckAvailability, Prebook and Book.
- **Where things stand:** the whole chain is deployed on **stage as prereleases** (Svalbard step 1 + step 2, obc-cache, Searcher, Consumer, ALPP Producer). **Nothing is merged.**
- **Dev results on stage (19/11):** all 8 ALPP scenarios match live ALPP CheckAvailability to the cent (rates APFLRA1/APFLRA3/APFLMRB1, rooms DBC/SAD, 1 and 2 adults). The APFLRA3 Prebook (2 adults) returns the same price in all 8 cases. Both checks were run again with Svalbard step 2 and gave the same results. Dev says it is **"Ready for QA"**.
- **QA on stage (19/11, Tamás):** WBC Search / CheckAvailability (OBC vs live) / Prebook for the 8 cases plus 12 variations: 0 failures. The DB checks (section 8) confirm the data behind them: free-night promotions stored with their indexes and the whole-stay flag, eligibility per case as expected, no orphan rows.
- **Still open:** Book → Cancel (not run), MinMaxStay regression (partly covered; candidates found), HBSI regression (**no HBSI free-night data on stage**, stage has a single shard), freshness/invalidation, obc-cache LOS and `/stats`. Three data issues to raise: an unplanned push to 9174 on 19/11 12:08, promo rows left over from the old-dates replay, and the Refundable (OBC) vs Nrf (live) policy difference. The production DB ALTER (SCN-15601) has not been run. One PR is still waiting for review (ALPP Producer #37). The NuGet still points to a prerelease. See section 9 for the acceptance criteria.

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

### 2.1 Spec rules (ALPP 1.1.3 wiki page) and where the implementation differs

| Spec says | Implementation / decision | Note |
|---|---|---|
| A free night is always tied to a MinStay. The page says MinStay = free night index + 1, with a minimum of 2 | MinStay is **read from the message**, never worked out from the index | The "+1" rule fails in 6 of the 8 test promos (for example Case 1: index 2, MinStay 5). The page was never corrected |
| The promotion cannot be applied partly: the whole stay must be inside the promotion period | Searcher requires the promo row on **every** night of the stay | ✅ |
| It cannot be combined with other promotions. ALPP returns the best price | Whole-stay comparison, cheapest total wins | ✅ |
| Free night: room amount = 0, **amount** taxes still apply, **percentage** taxes = 0. ALPP pushes the free-night price = the taxes (0.18 in the old test data; in the current stage data 0.00 in Case 1 and 5.00/10.00 in Case 7) | Free night = the promo's conditioned price. On a free night only the promotion's taxes apply, and a 0 charges nothing | ✅ (fixed in #129 after review) |
| Non-free nights use **net/grossOriginalPrice** | Non-free nights use the **base** `netPrice` (decision in SCN-15183, 14 Oct) | Same value in every test scenario. They would differ only if a promo ever sat on top of a discounted base (see R14) |
| `freeNightIndex` holds the positions within the stay, already expanded | Read the field as is. Indexes past the end of the stay are ignored. The promotion name is not used | ✅ |
| The free night applies per rate. With LOS-restricted rates (APFLRA3 min2/max3, APFLRA5 min5), ALPP returns the best of the eligible rate+promo combinations for the stay length | OBC prices each rate on its own. The best-rate choice works as today | Worth one check with a LOS-restricted rate (QA item 17) |

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
- **The wiki page is out of date:** it still lists the Nov 2026–Feb 2027 dates, with all 8 cases ticked as done. The current dates (Jan–Apr 2027) are only in Natasha's comment on SCN-15607 (12 Nov). The case numbers and logic are the same.

### 4.2.1 Reference case from the wiki (original data, 1 adult) – useful for the cancellation check

Case 1, 02–07 Nov 2026 (old dates), APFLRA3 DBC, 1 adult, `pointOfSale` ES:

- Nights: 87.18 / 87.18 / **0.18** / 87.18 / 87.18 net (gross 100.18, free night 0.18).
- Expected **348.90 net / 400.90 gross**. Taxes codes 12 and 19 at **0.45 each** (PS). `netOriginalPrice` 435.90 / `grossOriginalPrice` 500.90.
- Promotion `Promo_FN_Case1_Min5_D2`, applied from 04 to 05 Nov (the free night).
- Offer is **non-refundable**: penalty 100% = **348.90**, the promoted total and not the original 435.90. APFLMRB1 is refundable (50%, then 100%), so it is a better choice for the Cancel test.
- Running the same shape on the new dates (Case 1, 02–07 Jan 2027, 1 adult) should give the same structure. On 19/11, 1 adult gave the same price in OBC and ALPP availability, but ALPP's own Prebook rejected it (see R8).

### 4.3 QA checklist (SCN-15607)

| # | Area | Test | Expected result | Dev status |
|---|---|---|---|---|
| 1 | DB | `hotel_promotion` rows for 9174 | Free-night promos have `free_night_indexes` set and `require_all_days_stay = true`. Discount promos have NULL indexes | ✅ dev 09/11, ✅ QA 19/11 (SQL §3/§3b/§6) |
| 2 | DB | Column check (SCN-15601 verification query) on each shard | `ARRAY / _int2 / YES` | Stage ✅ (QA 19/11, single stage shard), prod ⏳ |
| 3 | Svalbard | `GET /debug/p_<hotel>_<day>_v2` | Step 2: rows carry `PromotionId` only, with no code or all-days flag | ✅ |
| 4 | obc-cache | `/v2/ari/daily`, `/v2/ari/los`, `/v2/hotel-details` for 9174 | `Promotions[]` listed once per hotel with indexes. Each promoted row or fee has `PromotionIndex` | ✅ daily + hotel-details. **LOS not reported** |
| 5 | obc-cache | `/stats` → `HotelPromotionsData` | Table loaded on start and populated | ⏳ |
| 6 | Searcher | Cases 1–8, 1 and 2 adults, every rate/room | Net = live ALPP CheckAvailability to the cent | ✅ |
| 7 | WBC | Search → CheckAvailability | No price change | ✅ |
| 8 | WBC | Prebook (2 adults, `RatePlanId` APFLRA3) | Same price as CheckAvailability | ✅ |
| 9 | WBC | **Book** for a sample of cases (at least 1, 3, 5, 7) | Booking created with the expected total | ⏳ **not done** |
| 10 | WBC | **Cancel**, plus cancellation policy and penalties on free-night stays | Policies and penalties correct. Penalty based on the **promoted** total (wiki Case 1: APFLRA3 non-refundable, 100% = 348.90, not 435.90). Use a refundable rate (APFLMRB1: 50%, then 100%) for a real Cancel | ⏳ **not done** |
| 11 | Regression | ALPP MinMaxStay promotion hotel | Same price as before / as production | ⏳ **not reported** |
| 12 | Regression | HBSI hotel with free nights | Unchanged | ❌ **No data on stage:** the `free_night` table has no future rows and stage has only one shard. Needs a production before/after comparison or HBSI test data (section 9) |
| 13 | Regression | Non-ALPP hotel without promotions (ARI v2 + hotel-details) | Identical response to production | ⏳ |
| 14 | Freshness | Create or change a promotion on stage | Consumer sends the `hotel_promotion` invalidation. Change visible in search after the obc-cache blob TTL (~108–132 min), without waiting for Svalbard's 30-minute map | ⏳ |
| 15 | Edge | Stay longer than the listed indexes / stay shorter than MinStay / stay partly outside the promo | Free nights only at the listed indexes. Promo not applied below MinStay or outside the period | Covered by unit tests. Case 2/6 on stage |
| 16 | Taxes | Free night with a percentage or amount tax | On a free night, only the promotion's taxes apply (0 charges nothing). Wiki Case 1: 400.90 gross, 0.45 per tax (codes 12, 19) | Unit test. Stage data (SQL §8): base taxes 0.09 per code per person per night; the Case 1 free-night promo has **no** tax rows, so the free night charges no tax, which fits the 520.00 gross at Prebook. A free night *with* a tax is not in the stage data (the old 0.18 case is gone) |
| 17 | Spec | Stays shorter than MinStay (1–4 nights on Case 1 dates), and a rate with LOS limits | No free night below MinStay. ALPP and OBC choose the same rate/promo for each stay length | ✅ below MinStay (folder 5, 19/11). LOS-limited rate ⏳ |
| 18 | Data | Stage holds no data from before the change, and nobody else pushes to 9174 (SCN-15607 "Important" note) | Only Natasha's push in the Case window, no leftovers | ⚠️ Leftovers from the 09/11 old-dates replay (SQL §13) and an unplanned push on 19/11 12:08 (SQL §3c). See R17, R18 |

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
| R16 | **Spec vs implementation:** the spec says non-free nights use `net/grossOriginalPrice` and MinStay = index + 1. The implementation uses the base `netPrice` and reads MinStay from the message | No impact on the current test data. Could differ in production if ALPP's real promotions differ from the test set | Decisions recorded in SCN-15183. Ask Natasha to update the wiki page (dates, the "+1" rule, price source) |
| R15 | **Large change set** (obc-cache alone is +4.7k lines), and part of it is only tested with fakes | Edge cases in production | Stage QA plus a gradual shard rollout |
| R17 | **Leftover promotion rows from older pushes.** On stage, `Promo_Disc_Case4` (from 10 Dec 2026) and `Promo_Disc_Case5` (from 22 Dec 2026) still have price rows from the 09/11 replay with the old dates, and `Promo_Disc_Case6` has tax rows on 05–06 Jan 2027 with no price rows | OBC may apply a discount to December stays that live ALPP no longer gives → price change at CheckAvailability. In production the same can happen when ALPP moves a promotion's dates without resending the old nights | Check OBC vs live for 10–17 Dec 2026. Ask for a push covering Nov 2026–Jan 2027 or a clean-up. Ask dev whether the Consumer should delete promo rows a push no longer contains (existing behaviour or new?) |
| R18 | **Unplanned push to 9174** on 19/11 12:08 UTC: promotion `qa-e2e-free-night` (id 727, index {2}) plus base and promo price rows | If it touched the case stays, OBC no longer equals live ALPP and the reference results are invalid | Find who pushed (dev e2e?), which nights changed (SQL §3c), and re-run folders 1–3 after it is reverted |
| R19 | **HBSI free nights can't be tested on stage** (no `free_night` rows, single shard) | A regression in the shared Searcher path would only show in production | Unit tests in #129 + a production before/after comparison on HBSI free-night hotels (section 9) |
| R20 | **Search shows a refundable policy, CheckAvailability a non-refundable one** (OBC `Refundable` 50 %/100 % vs live `Nrf` 100 %) | Customer sees a free-cancellation offer that turns non-refundable at CheckAvailability | Check whether it also happens on APFLRA1 (no promo) and on production; if so it's an existing OBC canpol issue. Raise a bug linked to SCN-15607 either way |

## 6. Next steps

1. QA: close the acceptance-criteria gaps listed in section 9 (Book/Cancel, MinMaxStay partial stay, HBSI, the data issues R17/R18/R20, freshness, obc-cache LOS/`/stats`). Record the results and the linked bugs in SCN-15607.
2. Get #37 reviewed. After #395 merges, release the NuGet and switch #37 to the final 1.0.237.
3. Run the SCN-15601 ALTER on all production shards and verify it.
4. Plan the production rollout in the order above, with a soak period before Svalbard step 2. Notify ALPP only after the Producer is live.
5. Before go-live, fill in the epic's Acceptance Criteria (still the template text) and decide on the optional items (Exporter, Viewer).

## 7. Postman collection (SCN-15607 stage QA)

Files: `ALPP_FreeNights_SCN-15607.postman_collection.json` (133 requests) and `ALPP_FreeNights_stage.postman_environment.json`. Each request has a description with the scenario, the rule tested and the expected result, plus test scripts that check it automatically. It was checked against a local mock (requests, variables and scripts work). **It has not run on stage yet.**

### 7.1 Safety rules (must always hold)

- **Never push to production.** The push requests refuse to run when `push_url` contains `obc.webbeds.com`.
- **Never push to hotel 9174.** Its OBC data comes from ALPP and must stay equal to ALPP live, or the OBC vs live tests break. The push requests refuse 9174 (the override `allow_push_9174=yes` exists but should not be used).
- **Never book against production** (Accor rule). The collection no longer contains Book/Cancel.
- The WBC Basic auth string is stored as a secret in the environment file. Anyone who gets that file gets the credential.

### 7.2 Structure

| Folder | What it does |
|---|---|
| 0. Environment checks | obc-cache `/stats` (`HotelPromotionsData` loaded) and `/v2/ari/daily` (body is a placeholder, TBC) |
| A. OBC push – synthetic pack | 12 ALPP push messages to a **test hotel** (default `PMDBE`, hub 444444, room DBC, rate APFLRA3, meal plan 3): availabilities, salesConditions (LOS 1–14), canpol (NRF), base price, then one price message per case with the promotions |
| B. Synthetic hotel checks | For each synthetic case: WBC Search, WBC CheckAvailability (OBC only) and Searcher direct, for 2 and 1 adults. No live comparison or Prebook: ALPP doesn't have this data |
| 1. WBC Search – 9174 | 8 ALPP cases, 2 adults, checked against the dev reference totals (section 4.2) |
| 2–3. WBC CheckAvailability – 9174 | OBC vs live ALPP, 2 adults and 1 adult. Run OBC first, then LIVE: the LIVE request compares every option to the cent |
| 4. WBC Prebook – 9174 | 2 adults, `RatePlanId` APFLRA3, using the option picked in folder 2. Prebook = CheckAvailability = reference |
| 5. Edge – 9174 | 1–4 night stays in the Case 1 period (below MinStay), OBC vs live |
| 6. Regression | OBC vs live on 0428 (hub 193650, MinMaxStay scenarios) and 2992 (hub 59503) |
| 7. ALPP QA direct | Optional, needs ALPP credentials: raw ALPP answer (promotion, free-night dates, cancellation policy) |

Hotels (hub ↔ ALPP): 193650 ↔ 0428 · 519595 ↔ 9174 · 65620 ↔ 1593 · 59503 ↔ 2992 · 444444 ↔ PMDBE.

WBC stage: `https://wbconnect-connectors.stage.webbeds.com/alpp/accor/{search/search | checkavail/checkavailability | prebook/prebook}?api-version=2.0`, Basic auth.

### 7.3 Synthetic push pack: data and expected results

Pricing per night: base **102.00** (2 ad) / **92.00** (1 ad), including 2.00 tax (code 19, RN, included). Free night = **2.00** (the tax only). Discounts apply to the room part only. Case 5 has expensive nights on 27–28 Feb (302.00 / 282.00). Period 2027-01-01 → 2027-04-10. Every night of a promotion carries the conditioned price, with the full `freeNightIndex` list, the way ALPP sends it.

| Case | Stay | Checks | Net 2 ad | Net 1 ad |
|---|---|---|---|---|
| 1 | 02–07 Jan 2027 | 1 free night (MinStay 5, index 2) | 410 | 370 |
| 2 | 15–21 Jan | Starts outside the promo → no promo | 612 | 552 |
| 3 | 25 Jan–01 Feb | 2 free nights (MinStay 6, indexes 2, 4) | 514 | 464 |
| 4 | 10–17 Feb | Free night beats 10% | 614 | 554 |
| 5 | 22 Feb–01 Mar | 10% beats free night | 1004 | 923 |
| 6 | 05–12 Mar | Free night not eligible → 10% | 644 | 581 |
| 7 | 20–25 Mar | Free nights (Min 2 / Max 5, indexes 2, 4) vs 15% | 310 | 280 |
| 8 | 01–06 Apr | Free nights (indexes 1, 3) vs 20% Min 3 / Max 6 | 310 | 280 |
| 8b | 20–26 Mar | MaxStay 5 exceeded → 15% | 522 | 471 |
| 1b | 02–05 Jan | Below MinStay → base | 306 | 276 |

These totals are calculated from the 1.1.3 rules applied to the pushed data; OBC hasn't produced them yet. Cases 1–4 and 6 match the totals in Natasha's scenario descriptions. Her totals for cases 7 and 8 don't add up, so they are not used. Promotion periods are set so the results don't change whether ALPP's `endDate` is exclusive or inclusive.

**Run order:** folder A in order (1 → 2 → 3 → 4 → 5.1 … 5.8), wait a minute or two for the Consumer, then folder B.

### 7.4 To confirm before running

- Stage push URL: `push_url` = `https://obc.stage.webbeds.com/obc/alpp/push` (Tamás, 19/11). A GET returns 405 Method Not Allowed, so the endpoint exists and expects POST. Production is `https://obc.webbeds.com/obc/alpp/push`.
- ~~PMDBE has room DBC, rate APFLRA3 and meal plan 3~~ **No:** on 19/11 PMDBE (OBC id 562) is inactive and has no room_rate rows at all (SQL §2). The push pack needs another target hotel with an active room/rate pair, then `push_hotel_code` / `push_room` / `push_rate` set to it.
- `api-version=2.0` on CheckAvailability and Prebook (given for Search; change `api_version` if rejected).
- `obc_cache_base` stage URL and the obc-cache `/v2/ari/*` request body.

### 7.5 Changes on 19/11 from Natasha's WBC captures

Natasha's folder `natasha-2adultos-8casos WBC` has the real requests and responses for the 8 cases (CheckAvailability OBC and LIVE, Prebook for APFLRA1/APFLRA3). The collection was replayed against them: **48 requests, 328 assertions, 0 failures** (folders 1–4).

- **Request shape** copied from her working requests: `OptionalCriteria { AllowRatePlans: true, IsObcRequest }` (was top-level `IsObcRequest`), `Currency: "EUR"`, `Language: "40"`, `UseCache`, `CutOffTime`.
- **OBC = LIVE key fixed.** OBC returns `rateType: Refundable` (50 % then 100 % penalty) and live ALPP `Nrf` (100 %) for the same offer. The old key included `rateType`, so every option would have failed as "missing in live". The difference is now an [INFO] test. **Worth raising:** search shows a refundable policy that CheckAvailability doesn't.
- **New checks:** APFLRA3 DBC nightly breakdown (folder 2), ALPP's applied promotion, free-night dates and gross read from `providerAudit`, and that OBC's free nights fall on ALPP's dates (folders 2, 3, 5).
- **Case data corrected:** case 3 has **3** free nights (26, 28, 30 Jan; indexes 1, 3, 5) and case 4's free night is the **2nd** night (11 Feb), as `freeNightIndex` says. Case 7/8 for 2 adults: 144.59 / 10.00 / 144.59 / 10.00 / 144.59.
- **Prebook** as in her working request: no `RateType`, `IsRefundable` from the LIVE option (`false`), plus promotion and gross checks.
- **Searcher direct:** `POST {{searcher_base}}/availability` with `ShowBreakdown: true`, the call WBC makes (`searcher_base` = `http://obc-searcher-go.service.aws-webbeds-westeurope-stage.consul`).
- **Push pack:** refuses any `push_url` outside `*.stage.webbeds.com`; promotion ids are now 24-hex like ALPP's.
- The old collection is kept as `ALPP_FreeNights_SCN-15607.postman_collection_v1_backup.json`.

### 7.6 Stage run, 19/11 ~11:50 UTC (folders 1–6, 60 requests)

The requests were sent from the browser to WBC stage, and the collection's checks were run on the responses: **258 checks passed, 0 failed.**

- All 8 cases, 2 adults: OBC = live ALPP on every option. Nightly breakdown, promotion, free-night dates and gross match the reference.
- 1 adult: OBC = live on every option. APFLRA3 DBC is 452.40 / 678.60 / 452.40 / 678.60 / 475.02 / 879.90 / 428.77 / 428.77.
- Prebook APFLRA3 (2 adults): same price as CheckAvailability in all 8 cases, with the expected promotion and gross.
- Below MinStay (1–4 nights in the Case 1 period): no free night in OBC or ALPP. Prices are 113.10 per night.
- **Known difference:** OBC returns rate type `Refundable` (50 %, then 100 % from 31 Dec). Live ALPP returns `Nrf`, and Prebook returns 100 % from 18/11.
- **Folder 6 tests nothing yet:** 0428 (hub 193650) and 2992 (hub 59503) return no rates for 10–13 Feb 2027, in OBC or live. For 0428, the searcher does return options (e.g. APFLRA1 225.72), but WBC maps none. The regression check needs another hotel or other dates.
- Folders A/B (push pack) not run yet.

### 7.7 More coverage and regression status (19/11, second pass)

**Folder 8 (new): 12 variations on 9174, plus a check of the old request shape.** All of them were run on stage at ~12:00 UTC. OBC = live on every option in every one.

| # | Variation | APFLRA3 DBC net | ALPP promotion / free nights |
|---|---|---|---|
| V1 | Case 1, check-in one day earlier (01–07 Jan) | 565.50 | FN Case1 · **03 Jan** (index counts from check-in) |
| V2 | Case 1, 6 nights (02–08 Jan) | 565.50 | FN Case1 · 04 Jan |
| V3 | Case 3, 8 nights | 452.40 | FN Case3 · 26, 28, 30 Jan, 01 Feb |
| V4 | Case 2 period, stay fully inside (16–20 Jan) | 339.30 | FN Case2 · 18 Jan (applies on 4 nights although the name says Min5) |
| V5 | Case 4 shifted (11–17 Feb) | 565.50 | FN Case4 · 12 Feb |
| V6 | Case 6, stay inside the free-night period (07–12 Mar) | 588.36 | FN Case6 · 08 Mar |
| V7 | Case 7, 6 nights (MaxStay 5 exceeded) | 738.60 | **15 % (no free night)** |
| V8 | Case 7, 2 nights | 154.59 | FN Case7 · 21 Mar |
| V9 | Case 8, 3 nights | 299.18 | FN Case8 · 02 Apr |
| V10–V11 | Cases 1 and 3, 2 adults + child (5) | 452.40 | same as 2 adults |
| V12 | Case 7, 3 adults | not offered | OBC and live agree: nothing sold for DBC |
| V13 | Old request shape (top-level `IsObcRequest: true`) | – | **Goes to live ALPP** and returns no `ratePlanId`: the old collection's "OBC" requests were live, so its OBC = LIVE check compared live with live |

**Regression: what is blocked and how much it matters**

| QA item | Status | Why | Importance |
|---|---|---|---|
| 11 MinMaxStay | **Partly covered** (candidates found 19/11: 2189 and A576, SQL §12a/§12d) | 0428 no longer has MinMaxStay promotions (the scenarios were Jan–Mar 2026) and WBC maps no 0428 rooms (DSB/DSBS/DBBH on hub 193650), so WBC returns nothing. Folder 6 now compares the Searcher with ALPP for 0428 from `providerAudit` (base prices match). The MinMaxStay logic is covered on 9174 by the conditioned discounts in cases 4–8 and V7 (MaxStay exceeded). | **High, still a gap:** a **partially applied MinStay** promotion (allowed by 1.1.2) isn't tested anywhere. The Searcher's whole-stay rule no longer depends on `require_all_days_stay`, so this is exactly where a regression would show. Ask Natasha/ALPP for a MinStay scenario that is applied partially (e.g. on 0428 or 9174) |
| 12 HBSI free nights | **Can't be done on stage** | The stage DB (one shard) has no future `free_night` rows (SQL §12b, 19/11), so no HBSI hotel with free nights exists to test. See section 9 for the alternatives | **High:** same Searcher code path, part of the Acceptance Criteria |
| 13 Non-ALPP hotel, ARI v2 / hotel-details | **Blocked** | `obc_cache_base` and the `/v2/ari/*` request body are unknown | **High:** obc-cache (+4.7k lines) serves every provider (R9) |
| 4–5 obc-cache LOS, `/stats` | **Blocked** | Same reason | Medium |
| ALPP without promotions | ✅ | 2992 has no data in OBC or ALPP, so it was replaced with **1593** (hub 65620): OBC = live in both directions | Medium |
| Push pack (A/B) | **Blocked on PMDBE** | PMDBE (id 562) is inactive and, in the DB on 19/11, has no room_rate rows at all (SQL §2) | Low for pricing: 9174 plus folder 8 cover the rules with real ALPP data. **Needed for freshness (item 14):** pick another stage test hotel with an active room/rate pair (not 9174) |

## 8. DB checks for the pushes (`SQL/SCN-15607_push_checks.sql`)

The SQL file checks the step between the push and search: what the Consumer wrote to the hotel shard. Its header holds the context needed to use it in a new session: the test hotels and their OBC ids, all 23 promotions of 9174, the expected WBC totals, and what the price and tax rows look like.

**How to use it:** connect to the stage hotel DB (read-only cluster from the wiki "Testing notes - OBC Exporter"; stage has a **single shard**). Run the queries one by one and change the values in the `p` CTE at the top of each one. A hotel code is not unique (9174 is ids 400 and 410, 1593 is 398 and 412, 0428 is 396, 397 and 1189), so every query also filters on `chain_id` (52 = ALPP/Accor test hotels). Sections 3b, 3c, 7, 13a and 13b return only problems: **expected 0 rows**.

### 8.1 Results on stage, 19/11 15:15 UTC

Exports are in the `SQL` folder (`20261119_1515 *.xml`).

| § | Check | Expected | Result |
|---|---|---|---|
| 0 | `free_night_indexes` column; `hotel_promotion_id` on price and breakdown rows | `ARRAY / _int2 / YES`; 2 rows | ✅ Both. Production shards ⏳ (R3) |
| 1 | Test hotels | One row per code and chain | ✅ 9174 = id **400** (chain 52; id 410 is an inactive chain-2 hotel). PMDBE id 562 is inactive, `ari_type` Unknown |
| 2 | Active room/rate pairs of PMDBE (push target) | At least one | ❌ **0 rows: PMDBE has no room_rate at all.** The push pack needs another target |
| 3 | 9174 promotions | 8 FN with indexes + whole-stay flag; discounts NULL | ✅ All 13 scenario promotions as expected. 23 rows in total: 9 legacy test promotions from Mar–Sep 2026 (NULL indexes) and **`qa-e2e-free-night`** (id 727, index {2}, written 19/11 12:08:07). `Promo_Disc_Case8` has `require_all_days_stay = true` (set 08/09, before the change) |
| 3b | Violations | 0 rows | ✅ 0 |
| 3c | Writes after Natasha's push (new) | 0 rows | ⚠️ Not run yet. §10 shows base and promo price rows written at 12:08:07 on 19/11, with the `qa-e2e` promotion (R18) |
| 4 | Promotion periods on the price rows | Case 2 and Case 6 stays start before their FN period | ✅ FN Case2 16–20 Jan; FN Case6 07–15 Mar, Disc Case6 05–15 Mar. APFLRA1 carries the Case 1 promo. Room TWC has partial rows (not a test room) |
| 5 | Night by night, Case 1, APFLRA3 DBC | FN row every night, free on 04 Jan | ✅ Base 113.10 every night (1 and 2 adults); FN Case1 row **0.00** every night; free only on 04 Jan → 4 × 113.10 = **452.40** |
| 6 | Eligibility of all 8 cases from the DB | Eligible promo for every case except 2; free nights as in 4.2 | ✅ FN Case2 covers 5/6 nights and FN Case6 5/7 (not eligible). Free nights 04 Jan / 26, 28, 30 Jan / 11 Feb / 21, 23 Mar / 02, 04 Apr. MinStay/MaxStay are on the price rows |
| 7 / 7b | Orphan promo rows; promo of another hotel | 0 rows | ✅ 0 / 0 |
| 8 | Taxes, Case 1 | Base taxes; free night only the promotion's taxes | ✅ Base: codes 12 and 19 at 0.09 per person per night. FN Case1 has **no** tax rows → the free night charges nothing (fits 520.00 gross). ⚠️ `Promo_Disc_Case6` tax rows on 05–06 Jan with no price rows (R17) |
| 9 | Inventory + restrictions, Case 1 | Open, base price every night | ✅ 999 free sale, open, LOS 1–31 allowed, base price every night |
| 10 | Last change per table | Moves after a push | `hotel_promotion` and `price_rule` 19/11 12:08:07 (the unplanned push), inventory 12:52, restriction 18/11 16:30 |
| 11 | Synthetic pack promotions | 13 rows once pushed | 0 rows (not pushed; PMDBE unusable) |
| 12a | Partial MinStay candidates | Hotels to test | **2189** ibis Paris 17 (promo 631 −20 % min 3, 01 Jan–28 Feb 2027; promo 700 min 2) and **A576** Fairmont San Francisco (promo 597 min 2, 03–31 Jan 2027). Stale 9174 discounts from Dec 2026 also show here (R17) |
| 12b | HBSI `free_night` hotels | Hotels to test | ❌ **0 rows.** No HBSI free-night data on stage, and there is no other stage shard |
| 12c | 1593 without promotions | 0 / 0 | ✅ 0 / 0 |
| 12d / 13a / 13b | Partial MinStay stay; leftover promo rows (new) | – / 0 / 0 | Not run yet. 13a/13b turn the R17 findings into checks |

### 8.2 What the results mean

- **The free-night data is right.** What the Consumer stored explains every WBC result: the indexes, the whole-stay flag, the promotion periods and the per-case eligibility all agree with the reference table in 4.2.
- **The file's first version had wrong expectations**, now fixed: hotel matched by code only (the 9174/1593/0428 duplicates), 13 instead of 23 promotions, a free-night price of 0.18 (it is what ALPP pushes: 0.00 in Case 1, 5.00/10.00 in Case 7), APFLRA1 "without promotion", and PMDBE "with inactive rooms".
- **To raise or clarify:** R17 (leftover rows from the old-dates replay), R18 (unplanned push to 9174), R19 (no HBSI data on stage).

## 9. SCN-15607 acceptance criteria: coverage and what is left (19/11)

| Acceptance criterion | Status | Evidence | What is left |
|---|---|---|---|
| **OBC:** for every usable scenario, the OBC search total matches ALPP's availability | ✅ | 8 cases × 1 and 2 adults, every rate/room (7.6); 12 variations (7.7); DB (8.1) | Re-run folders 1–3 once R18 is cleared, in case the 12:08 push changed the case nights. Optional: one LOS-limited rate (item 17) |
| **OBC:** free-night promotions are not applied to stays that don't meet their conditions | ✅ | Case 2 and Case 6 (period), 1–4 nights below MinStay (folder 5), V7 (MaxStay exceeded), DB §6 | – |
| **OBC:** ALPP MinMaxStay promotions give the same results as before | ⚠️ Partly | Conditioned discounts in cases 4–8 and V7; 0428 base prices | A **partially applied MinStay** promotion: run SQL §12d on 2189 (30 Dec–05 Jan, promo 631 from 01 Jan, min 3) or A576, get their hub ids, then compare WBC OBC vs live (or Searcher stage vs production) for that stay |
| **OBC:** HBSI free nights give the same results as before | ❌ No data on stage | #129 unit tests (MinMaxStay unchanged, existing free-night paths) | Stage can't prove it. Options, best first: **(a)** run SQL §12b on production (read-only) to pick 3–5 HBSI free-night hotels, record the production Searcher responses for stays that hit their free nights, and compare them after the Searcher release on the first shard (go/no-go gate in the rollout); **(b)** ask the HBSI team for a free-night push to a stage test hotel; **(c)** have dev insert `free_night` rows for an HBSI hotel on stage. Agree the option with the PO and record it on the ticket |
| **WBC:** no price change between Search and CheckAvailability/Prebook | ✅ 2 adults / ⚠️ 1 adult | Folders 1–4: Search = CheckAvailability = Prebook in all 8 cases (2 adults) | 1-adult Prebook blocked by ALPP QA (R8); re-test when ALPP fixes it. Raise the **Refundable vs Nrf** policy difference (R20) |
| **WBC:** bookings and cancellations with free nights complete, with correct totals and penalties | ❌ Not run | – | Book on **stage** (ALPP QA, never production) for cases 1, 3, 5 and 7 at least: 2 adults, APFLRA3, `RatePlanId` set. Check the booking total = Prebook (452.40 / 452.40 / 475.02 / 453.77 net). Cancel each one: APFLRA3 is non-refundable, so the penalty must be 100 % of the **promoted** total, not the original. APFLMRB1 (refundable) fails Prebook because it isn't in the ARI (R8); ask ALPP/Natasha to load it if a refundable cancel is needed. Add Book/Cancel requests to the collection (stage URL guard like the push pack) |
| Any mismatch is raised as a bug linked to this task | ⚠️ | – | Raise and link: R20 (canpol Refundable vs Nrf), R17 (leftover promo rows, ask dev if the Consumer should clean them), R18 (unplanned push to 9174). ALPP QA issues (R8) go to ALPP via Natasha, not as OBC bugs |

**Also in the ticket description, not in the AC:**

- **`/v2/ari` returns the promotion with its indexes:** dev reported daily and hotel-details; **LOS** and the obc-cache **`/stats`** (`HotelPromotionsData`) are not checked by QA. Needs `obc_cache_base` and the request body from dev (item 4, 5, 13).
- **Freshness:** create or change a promotion on stage and time how long search takes to show it (item 14). Do it on a hotel other than 9174, once the push pack has a usable target.
- **Old data on stage:** the description asks to clean or resend data sent before the change. R17 shows it is still there outside Natasha's window. Run SQL §13a/§13b and an OBC vs live check for 10–17 Dec 2026.

## Sources

- Jira: [SCN-15600](https://webbeds.atlassian.net/browse/SCN-15600) · [SCN-15601](https://webbeds.atlassian.net/browse/SCN-15601) · [SCN-15602](https://webbeds.atlassian.net/browse/SCN-15602) · [SCN-15603](https://webbeds.atlassian.net/browse/SCN-15603) · [SCN-15604](https://webbeds.atlassian.net/browse/SCN-15604) · [SCN-15605](https://webbeds.atlassian.net/browse/SCN-15605) · [SCN-15606](https://webbeds.atlassian.net/browse/SCN-15606) · [SCN-15607](https://webbeds.atlassian.net/browse/SCN-15607) · [SCN-15183](https://webbeds.atlassian.net/browse/SCN-15183)
- PRs: [webbeds-obc#395](https://github.com/WebBeds/webbeds-obc/pull/395) · [webbeds-obc-alpp#37](https://github.com/WebBeds/webbeds-obc-alpp/pull/37) · [svalbard#98](https://github.com/WebBeds/svalbard/pull/98) · [svalbard#99](https://github.com/WebBeds/svalbard/pull/99) · [obc-cache#226](https://github.com/WebBeds/obc-cache/pull/226) · [obc-searcher#129](https://github.com/WebBeds/obc-searcher/pull/129)
- Wiki (push formats and WBC flow): [WbOBC - ALPP](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/3360161816/WbOBC+-+ALPP) · [ALPP - Price](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/3359834189/ALPP+-+Price) · [ALPP - Availabilities](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/3360096274/ALPP+-+Availabilities) · [ALPP - Sales Conditions](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/3360227376/ALPP+-+Sales+Conditions) · [ALPP - Cancellation Policies](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/3360030845/ALPP+-+Cancellation+Policies) · [MinMaxStay in Promotion 1.1.2](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/4071194658/ALPP+-+Versions+-+MinMaxStay+in+Promotion+1.1.2) · [Internal Models](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/3257633057/Internal+Models)
- Spec: [ALPP - Versions - FreeNights in Promotion 1.1.3](https://webbeds.atlassian.net/wiki/spaces/WTO/pages/4983324688/ALPP+-+Versions+-+FreeNights+in+Promotion+1.1.3)
