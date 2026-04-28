# Validated Source List — Stage 3b
**Run ID:** 20260426-jeep-wrangler-2024-us-a7f2c  
**Validation Stage:** 3b (Source Validation)  
**Date:** 2026-04-26  
**Input Candidates:** 16  
**Validated (Kept):** 15  
**Dropped:** 1  

---

## Validation Rules Applied

| Heuristic | Action |
|:---|:---|
| Car match (brand, model, year/market) | ✅ All kept candidates match 2024 Jeep Wrangler |
| Year match | ✅ No wrong-year drops |
| Market match | ✅ All candidates US-scoped or US-compatible |
| Paywall/login-wall | ✅ No soft-paywall sources |
| Aggregator/resale | ⚠️ Forum flagged but retained (technical content) |
| Dead link | ✅ No 404s or dead links detected |
| Forum/UGC as primary | ⚠️ 1 downgraded; 1 dropped |

---

## Validated Candidates (15 Kept)

| Rank | Source Site | URL | Validation Status | Flags | Why Selected |
|:---:|:---|:---|:---:|:---|:---|
| 1 | jeep.com (Official) | https://www.jeep.com/am/en/wrangler/technology.html | ✅ PASS | — | Official Jeep source, explicit 2024 Wrangler tech/ADAS/infotainment coverage |
| 2 | jeep.com (Official) | https://www.jeep.com/active-driving-assist.html | ✅ PASS | — | Official Jeep, detailed ADAS feature spec (Active Driving Assist 2) |
| 3 | jeep.com (Official EV) | https://www.jeep.com/ev/technology.html | ✅ PASS | — | Official Jeep EV/hybrid platform, covers 4xe powertrain & charging |
| 4 | Stellantis (Official) | https://www.facebook.com/StellantisNA/posts/... | ✅ PASS | — | Official Stellantis press, 2024 Wrangler safety & tech announcement |
| 5 | uconnect (Official) | https://www.driveuconnect.com/system-features.html | ✅ PASS | — | Official Uconnect service portal, infotainment & connectivity features |
| 6 | Edmunds (Specialist) | https://www.edmunds.com/jeep/wrangler/2024/features-specs/ | ✅ PASS | — | Authoritative specialist automotive review, exact 2024 Wrangler specs |
| 7 | Car and Driver (Press) | https://www.caranddriver.com/jeep/wrangler/specs/2024/jeep_wrangler_jeep-wrangler_2024 | ✅ PASS | — | Specialist press, detailed 2024 Wrangler powertrain & performance |
| 8 | Car and Driver (Press) | https://www.caranddriver.com/jeep/wrangler-4xe-2024 | ✅ PASS | — | Specialist press, 2024 Wrangler 4xe hybrid dedicated coverage |
| 9 | KBB (Specialist) | https://www.kbb.com/jeep/wrangler/2024/specs/ | ✅ PASS | — | Kelly Blue Book specialist, trim-level specs & options |
| 10 | US News (Specialist) | https://cars.usnews.com/cars-trucks/jeep/wrangler-4xe/2024/specs/wrangler-4xe-rubicon-x-4x4-435351 | ✅ PASS | — | Authoritative automotive coverage, 2024 Wrangler 4xe detailed specs |
| 11 | YouTube (Video) | https://www.youtube.com/watch?v=8VfRKhTNBoA | ✅ PASS | — | Press/video overview, 2024 Wrangler feature walkthrough |
| 12 | YouTube (Video) | https://www.youtube.com/watch?v=tGXDTs4fyLc | ✅ PASS | — | Press/video, 2024 infotainment (12.3" screen, wireless CarPlay/Android Auto) |
| 13 | YouTube (Video) | https://www.youtube.com/watch?v=He3mpzLQMi0 | ✅ PASS | — | Press/video, comprehensive Uconnect 5 infotainment system review |
| 14 | Scribd (Documentation) | https://www.scribd.com/doc/274251201/Jeep-Wrangler-Unlimited-Rubicon | ✅ PASS | — | Brochure documentation, 2024 Unlimited trim-specific specs |
| 15 | Universal Auto (Dealer) | https://www.universalautogroup.com/2024-jeep-wrangler-specs/ | ✅ PASS | — | Dealer press, 2024 off-road & performance spec coverage |

---

## Dropped Candidates (1)

| Rank | Source Site | URL | Drop Reason | Notes |
|:---:|:---|:---|:---|:---|
| — | JL Wrangler Forum | https://www.jlwranglerforums.com/forum/threads/... | `forum-primary-only` | Community UGC discussing wireless CarPlay but no authoritative source link; downgraded during discovery; re-evaluated as drop during validation |

**Rationale:** Forum as primary source carries anecdotal claims only; no official validation. If technical deep-dives are needed post-approval gate, can circle back with specialist sites (#6–#10) as authoritative backup.

---

## Category Coverage (Post-Validation)

| Category | Covered | Sources | Confidence |
|:---|:---:|:---|:---|
| **ADAS** | ✅ | jeep.com (#1, #2), Stellantis (#4) | High (Official) |
| **Infotainment** | ✅ | jeep.com (#1), Uconnect (#5), YouTube (#12, #13) | High (Official + Video) |
| **EV/Hybrid 4xe** | ✅ | jeep.com (#3), Car and Driver (#8), US News (#10) | High (Official + Specialist) |
| **Powertrain (Gas)** | ✅ | Edmunds (#6), Car and Driver (#7), KBB (#9) | High (Specialist) |
| **Safety** | ✅ | jeep.com (#1), Stellantis (#4) | High (Official) |
| **General Specs** | ✅ | Edmunds (#6), KBB (#9), Universal Auto (#15), Scribd (#14) | High (Specialist + Brochure) |

---

## Assessment

✅ **Validation Complete — Ready for Approval Gate**

- **Input:** 16 candidates
- **Output:** 15 validated (93.75% retention)
- **Coverage:** All major categories represented
- **Source Type Distribution:** 5 Official + 5 Specialist + 3 Video + 1 Brochure + 1 Dealer
- **Confidence:** High — no paywall, no dead links, no wrong-year mismatches

**Next Step:** → **Stage 3c (Approval Gate)** — Present validated list to user for explicit approval before download/ingest.

