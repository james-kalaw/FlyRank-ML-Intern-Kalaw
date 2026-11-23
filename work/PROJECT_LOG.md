# Project Log — FlyRank ML Internship (James Kalaw)

A running record of decisions, findings, and rules for this track. Update it after every
assignment. It feeds `capstone_report.md` and the final paper, and lets any new AI chat
catch up quickly (paste this file in).

> **Data safety:** this file holds **aggregate numbers only** (medians, rates, counts, AUC).
> No raw rows, no `content_hash_id` / `client_hash_id` values, no client details. Keep it that way.

_Last updated: 2026-11-21 (after ML-07)_

---

## 1. Project at a glance

| | |
|---|---|
| **Lane** | Lane 1, Ranking Signal Analysis. **Final:** kept at the lane lock (end of Week 4) |
| **Question** | Which common SEO beliefs actually hold up in this data, and which don't? |
| **Who it helps** | A content strategist / SEO lead deciding what their team should spend time on |
| **Decision improved** | Keep acting on an assumption (e.g. "refresh old pages", "write longer") vs. drop it |
| **Unit of analysis** | One page. Starter CSV: 90-day totals. Warehouse: features from Feb 2026, outcome from Mar 2026 |
| **Outcome** | `is_declining`: impressions dropped more than 20% (warehouse: per day, Feb → Mar) |
| **Metric** | Decline-rate gap: decline rate in top 25% of a signal minus bottom 25%, in points. Notable ≥ 5, weak 2–5, none < 2 (set in advance) |
| **Output** | A signal report: each tested signal gets a verdict, plus content recommendations |
| **Baseline (ML-07)** | Feedly articles first, ordered by Feb impressions. Precision@100 **51.0%**, @500 **49.6%** vs base rate 25.1% (random ~25%) |
| **Claims allowed** | Observed, measured, directional, decision-support. Never causal, never "how Google works" |

**One-paragraph frame (from ML-02):**
For a content strategist or SEO lead deciding which common assumptions to act on (content age,
word count, search volume) versus ignore, we will build a signal report from the anonymized
content performance dataset, testing which measured signals show a real relationship with
traffic, clicks, and decline, using correlations and grouped comparisons as the method. A wrong
call costs wasted team hours chasing signals that do not matter, or continued neglect of signals
that do. A plain rule is not enough because several common assumptions have already failed
simple checks in this data. We will claim only observed, directional findings in this dataset,
never proof of how Google's algorithm works.

**Candidate original angle (not locked in):** a "signal reliability scorecard" that tests each
signal more than one way (end-to-end gap, tier table, interaction) and flags where they disagree.
Evidence it matters: position looked "weak" by the gap (−3.7) but is hill-shaped in the tier table (F13);
and page age / search volume looked notable in the same-window starter CSV but were FALSE in the
Feb → Mar warehouse test (B1, B2).

---

## 2. Assignment tracker

Submission = repo URL `https://github.com/james-kalaw/FlyRank-ML-Intern-Kalaw` on each portal card.

| Card | Notebook (`work/notebooks/`) | Status | Key output |
|---|---|---|---|
| warm-up | `notebooks/01_first_look_and_discovery.ipynb` | Done | Pipeline run + Discoveries A–C + own test (F1–F5) |
| warm-up | `notebooks/02_your_first_readable_model.ipynb` | Done | Hand rule vs. tree, leakage demo |
| ML-02 | `w01_research_question.ipynb` | Submitted 2026-11-18 | Lane 1 chosen; frame; 3 numbers |
| ML-03 | `w02_ml_task_framing.ipynb` | Complete 2026-11-19 (confirm portal submission) | Signal analysis; outcome; decline-gap metric (F7–F15) |
| ML-04 | `w03_data_contract.ipynb` | Complete 2026-11-20 (confirm portal submission) | Feb → Mar contract on warehouse; 3 queries; leak trap (W1–W8) |
| ML-05 | `w03_feature_leakage_check.ipynb` | Optional (card retired 2026-07-13; core moved to ML-04) | |
| ML-06 | `w04_signal_audit.ipynb` | Optional (card retired 2026-07-13) | Closest fit to Lane 1 |
| ML-07 | `w04_baseline_score.ipynb` | Complete 2026-11-21 (confirm portal submission) | Signal checks, EDA charts, feedly-first rule, top-10 review (B1–B12) |
| ML-08 | `w05_model.ipynb` | **Next** | Must beat the baseline on the same frame and metric |
| ML-09 | `w06_validation_audit.ipynb` | | |
| ML-10 | `w07_action_playbook.ipynb` | | |
| ML-11 | `capstone.ipynb` + deployed paper | | URL goes in `submission/paper_url.txt` |
| ML-12 | closing cells of `capstone.ipynb` | | 5-min demo outline, social post, employer summary |

---

## 3. Findings: starter CSV (30,000 pages, 32 clients)

Observed in this sample only. Rates are on a ×100 scale (`ctr = 0.35` means 0.35%).

### Warm-up 1 and ML-02

| # | Belief tested | Result | Verdict | Source |
|---|---|---|---|---|
| F1 | "High search volume means more traffic" | corr(`search_volume`, `impressions_90d`) = **0.001** | Doesn't hold | NB01; ML-02 |
| F2 | "Better position means more clicks" (grouped) | Mean CTR, pages ≥100 impressions: page_1 **0.35%**, top_3 **0.33%**, striking **0.26%**, page_3_5 **0.14%**, deep **0.06%** | Holds, when grouped | NB01 |
| F3 | "Longer content performs better" (medians) | Median `word_count`: down 2,909 · stable 2,912 · up 2,848 · flat 2,698 · new 2,239 | Looked flat (but see F9) | NB01 |
| F4 | "Better position means more clicks" (raw correlation) | corr(`avg_position`, `ctr`) = **−0.073** | Looks absent → non-linear | NB01; ML-02 |
| F5 | "Old content declines" (medians) | Median age: down **216** · flat 231 · new 279 · up 292 · stable **300** days | Backwards | NB01; ML-02 |
| F6 | Share of pages by trend | down **54.2%** · stable 19.9% · up 14.6% · new 7.5% · flat 3.8% | Base rate 54.2% | ML-02 |

### ML-03 (Week 2)

**Label check:** `down` = `trend_pct` from −100% to −20% (16,262 pages); `stable` −20% to +20% (5,962);
`up` +20% to +44,900% (4,388). `trend_pct` matched (last30 − prev30) / prev30 on **100%** of pages
with prev30 > 0. 3,388 pages had prev30 = 0 (blank trend: `new` / `flat`).

**Decline-rate gap (top 25% minus bottom 25%), base rate 54.2%:**

| # | Signal | Bottom 25% | Top 25% | Gap | Verdict |
|---|---|---|---|---|---|
| F7 | `content_age_days` | 60.0% | 43.7% | **−16.4** | notable: older pages declined less |
| F8 | `search_volume` | 63.2% | 51.1% | **−12.2** | notable: higher-volume keywords declined less |
| F9 | `word_count` | 49.7% | 60.0% | **+10.3** | notable: longer pages declined more. **Contradicts F3**; re-check by `content_type` |
| F10 | `avg_position` (> 0 only) | 55.1% | 51.3% | **−3.7** | weak by the gap, but see F13 |

| # | Check | Result |
|---|---|---|
| F11 | Unit | 30,000 rows = 30,000 unique pages. Pages per client: min 3, median 567, max 7,008. **Top 3 clients = 43.3%** of pages |
| F12 | Folklore rule "pages 365+ days old decline" | 6,360 pages flagged, **42.6%** declining vs **54.2%** base rate → points the wrong way |
| F13 | Decline rate by position tier | top_3 **24.1%** (2,321) · page_1 57.0% (11,814) · striking **61.0%** (7,304) · page_3_5 56.2% (7,242) · deep **34.4%** (1,319) → **hill-shaped** |
| F14 | Decline rate by age tier | 31–90 days **66.9%** (492) · 91–180 62.6% (11,780) · 181–365 51.5% (11,368) · 365+ **42.6%** (6,360) → near-straight decline |
| F15 | Age × impression tier (cells ≥100 pages) | Low traffic: newest **65.6%** vs oldest **40.1%** (~25 pts). Excellent traffic: 46.2% vs 42.1% (~4 pts) → **interaction** |

Columns: 32 signals, 13 excluded (2 IDs, 3 outcome, 6 outcome ingredients `*_last_30d` / `*_prev_30d`,
2 metadata). Most missing: `word_count` / `char_count` 25.7%, keyword columns ~8%.

### Model results seen in the warm-ups (context, not my lane's method)

| Setting | Metric | Hand rule | Model |
|---|---|---|---|
| NB01 pipeline, client-holdout | Precision@50 | 0.240 | 0.740 (random forest) |
| NB02 in-sample, "stale × visible" vs depth-2 tree | Precision@20 / @50 | 0.900 / 0.680 | 0.550 / 0.600 |
| NB02 depth-4 tree, in-sample | Precision@20 / @50 | — | 0.600 / 0.680 (ties the rule at @50) |
| NB02 client-holdout (seed 42) | Precision@50 | 0.620 | 0.560 |
| NB02 leaky tree (used `trend_pct`) | Precision@50 | — | 1.000 (fake: leakage) |

---

## 4. Findings: warehouse, Feb → Mar 2026 (ML-04, Week 3)

**Design:** features from 1–28 Feb 2026; decision day 1 Mar; outcome from 1–31 Mar.
`is_declining` = Mar impressions/day < 80% of Feb impressions/day. Filters: client `gsc_data_start`
≤ 1 Feb; page created before 1 Feb; ≥ 100 Feb impressions; `gsc_data_available IS TRUE`.
Tables: `fact_content_daily_performance` (Feb + Mar partitions), `dim_content`, `dim_clients`.

| # | Check | Result |
|---|---|---|
| W1 | Clients | **104** total; **41** with a full February; **37** with no Search Console start date |
| W2 | Grain | **0** duplicate (report_date, client, content) rows → one row = one page on one day |
| W3 | Counts | Feb: **7,355,108** rows, 321,546 pages, 54 clients, 28 days. Mar: **9,841,378** rows, 331,437 pages, 55 clients, 31 days |
| W4 | Availability | `IS TRUE`: Feb **35.6%** (FALSE 4,641,069; NULL 92,256). Mar **36.7%** (FALSE 6,230,317; NULL 0) |
| W5 | Frame | **65,756** pages from **26** clients; 3,605 dropped (no usable March data); base rate **25.1%** declining |
| W6 | Features (median; min–max) | age 209 days (29–464) · word_count 2,722 (0–10,987), **37.9% missing** · search_volume **10** (0–246,000), 2.0% missing · Feb avg position 6.9 (0–104.7) · Feb impressions 823 (100–203,401) |
| W7 | Leak trap (client-holdout, 7 test clients, 39,148 pages, base 25.6%) | AUC **0.594** honest → **0.994** with `mar_impressions` → removed; honest kept |
| W8 | Snapshot limit | **76.5%** of frame pages have `content_updated_date` after 1 Mar; **43.9%** have `last_optimized_date` after 1 Mar → `word_count` unreliable as a Feb clue |

**Why 25.1% ≠ 54.2%:** different measurement (full month vs month, per day, ≥100 impressions,
mid-panel instead of end of panel). The two base rates must not be compared directly.

**Data quirks to check in Week 4:** `word_count = 0`; `feb_avg_position = 0` with ≥100 impressions
(no "rank 0" exists); `search_volume` heavily skewed (use medians / rank-based methods);
test split held 60% of pages from 25% of clients (uneven client sizes again).

---

## 5. Findings: baseline, Feb → Mar 2026 warehouse frame (ML-07, Week 4)

Same frame and label as ML-04 (65,756 pages, 26 clients, base rate **25.1%**). Verdict rules set before
looking (buckets with 500+ pages): CONFIRMED / OPPOSITE = end buckets 5+ points apart (right / wrong way);
MIXED = ups and downs or only 2–5 points; FALSE = every bucket within 2 points of the base rate.

**Signal checks for the first rule idea ("young pages on low-volume keywords"):**

| # | Signal | Buckets (% declining, n) | Verdict |
|---|---|---|---|
| B1 | Page age (days on 1 Mar) | <90: 24.9% (10,859) · 90–179: 27.0% (14,436) · 180–269: 23.8% (23,063) · 270–364: 25.7% (10,439) · 365+: 25.2% (6,959) | **FALSE** (max 1.9 pts) |
| B2 | Search volume (flag-linked: quick-win) | 0: 26.0% (20,742) · 1–10: 23.6% (18,967) · 11–100: 23.7% (16,753) · 101–1000: 26.3% (6,619) · 1001+: 26.8% (1,329) · **missing: 42.9% (1,346)** | **FALSE** (max 1.7 pts); missing bucket +17.8 |

Both rule signals failed, so the rule was changed before it was built. Contrast with the starter CSV
(same-window measure): age gap −16.4, volume gap −12.2 (F7, F8).

**EDA on four more clues (bar charts in the notebook):**

| # | Clue | Result | Reading |
|---|---|---|---|
| B3 | Content type | keyword article 24.8% (65,047) · **feedly article 53.3% (704)** · comparison article 80.0% (5, too few) | Strongest: +28.2 pts. All 704 feedly pages have missing search volume (missing bucket = 704 feedly + 642 keyword) |
| B4 | Feb position | 0 (no data) 42.2% (1,251) · 1–3 28.1% (12,026) · 4–10 26.7% (32,336) · 11–20 23.3% (11,806) · 21–50 **13.7%** (7,411) · 51+ 21.7% (926) | From rank 1 to 50, higher-ranked pages declined more. Not hill-shaped as in F13. "0" is a data quirk |
| B5 | CTR vs position peers (flag-linked: CTR-fix) | <0.5× 28.8% (34,346) · 0.5–0.9× 23.3% (10,096) · 0.9–1.1× 22.0% (3,738) · 1.1–2× **18.8%** (10,249) · 2×+ 20.6% (7,327) | Fewer clicks than peers → more decline. MIXED (end gap −8.2, small uptick at the end). Peer CTR = group clicks ÷ group impressions |
| B6 | Feb impressions | 100–249 26.8% · 250–499 27.5% · 500–999 25.9% · 1k–2.5k 23.0% · 2.5k–10k 23.1% · 10k+ 24.4% | Weak (max 2.4 pts) |

Selection rule (set before looking): use the clue whose 500+ page bucket is farthest from the base rate →
**content type (feedly article, +28.2)**. Six clues tested in total, so some luck is possible.

**The baseline rule:** feedly articles first, ordered by February impressions; everything else score 0.
Score `is_feedly × feb_impressions`; reason code `feedly_article`; action `review_for_decline_risk`.
Reads only `content_type` and `feb_impressions`. Queue written to `work/outputs/baseline_action_score.csv`
(git-ignored); metrics in `work/outputs/baseline_metrics.json`.

| # | Check | Result |
|---|---|---|
| B7 | Precision@K (rule / random picking, 20 shuffles) | @10 **70.0%** / 21.0% · @50 **46.0%** / 24.2% · @100 **51.0%** / 25.5% · @500 **49.6%** / 25.2% |
| B8 | All flagged pages | 704 flagged; 375 declined → **53.3%** |
| B9 | Client concentration | Flagged pages come from **7** clients; one client holds **45.0%** (317 pages); 6 of the top 10 are from that client |
| B10 | Top-10 review | **7 of 10** declined. Shared risks: a February spike returning to normal, traffic moving to a sibling page (not checked) |
| B11 | Weak picks (feedly misses vs hits, medians) | CTR vs peers **0.60** vs **0.30**; Feb impressions 291 vs 232; position 8.75 vs 7.98; age 221 vs 206 days |
| B12 | Ordering | Sorting by Feb impressions did not help: top 50 = 46.0% vs 53.3% for all feedly pages. Content type does the work |

Leakage check passed: the rule reads no March data, label, excluded column, or product flag.
`content_type` comes from the present-day `dim_content` snapshot; assumed stable since publishing.

**Hint for ML-08:** within feedly pages, misses had about twice the CTR-vs-peers of hits, so a model that
combines content type with CTR vs peers may beat the baseline.

---

## 6. Data facts and traps

**Starter CSV** (`docs/data-dictionary.md`, `skills/flyrank/flyrank-data`)
- Label columns, never inputs: `trend_direction`, `trend_pct`. Outcome ingredients: six `*_last_30d` / `*_prev_30d`.
- IDs are pseudonyms: grouping/splitting only. Rate columns are ×100. `avg_position = 0` = no data.
- Missingness follows `content_type`; don't blind `fillna(0)`.

**Warehouse** (release build v20260703, Hugging Face, gated)
- Daily grain: report_date × client × content. Position column: `gsc_avg_position` (aggregate it weighted by impressions).
- `gsc_avg_position = 0` with impressions: 1,251 frame pages (42.2% declining) — a quirk; keep it as its own bucket.
- DuckDB `CREATE SECRET` needs DuckDB 1.1+. If Colab already loaded an older copy: Runtime → Restart session.
- Flags can be TRUE / FALSE / NULL: always filter with `IS TRUE`.
- Unbalanced panel: check `dim_clients.gsc_data_start` before any window.
- `dim_content` is a present-day snapshot: `content_updated_date`, `last_optimized_date`, `is_deleted`,
  `is_published` can describe the future relative to a past decision day.
- Never develop on `_sample` or June 2026 (sealed test month). Iterate on mid-panel months.
- HF token only through Colab Secrets (`HF_TOKEN`). Never in a cell, never in chat.

---

## 7. Working rules

**Data and AI**
- Share only code, errors, column names, and aggregate outputs with AI tools. Never raw rows, the CSV,
  or warehouse extracts (`DATA_USE.md` rule 2). Notebooks print summaries only.
- Code is tested on synthetic data with the real schema before it touches the real data.

**Repo**
- Build everything in `work/`. Don't edit `scripts/`.
- Commit notebooks **with outputs** (Run all first). CI checks this.
- Colab "Save a copy in GitHub": check the **Repository** dropdown says `FlyRank-ML-Intern-Kalaw`,
  the path is `work/notebooks/<exact name>.ipynb`, and "Include a link to Colab" is unticked.
- Before submitting, check the notebook item by item against the card's "What done looks like".

**Claims**
- Use: observed, measured, directional, decision-support. Report the base rate next to any score.
- Never: causes, proves, "predicts Google", generalizes beyond these clients.

---

## 8. Decision log

| Date | Decision | Why |
|---|---|---|
| 2026-11-18 | Lane 1 (Ranking Signal Analysis) | Builds on tools already working; Week 1–2 tests already produced non-obvious findings |
| 2026-11-18 | Share aggregates only with AI tools | `DATA_USE.md` rule 2 |
| 2026-11-18 | Outcome = `is_declining`; metric = top-vs-bottom-25% decline-rate gap, thresholds 5 / 2 points | Plain English; grouped comparison; set before looking |
| 2026-11-19 | Warehouse design: Feb 2026 features → Mar 2026 outcome, per-day label, ≥100 Feb impressions | Features before outcome (fixes Week 2's same-window limit); per day because Feb has 28 days and Mar 31 |
| 2026-11-19 | 5 features: content age, word count, search volume, Feb avg position, Feb impressions | Strongest Week 2 signals + visibility before decision day |
| 2026-11-20 | Keep Lane 1 at the lane lock | Continuous story since Week 1; foundation already built |
| 2026-11-20 | Follow FlyRank rules strictly; check every notebook against its card | Goal: capstone graded Pass |
| 2026-11-20 | First rule idea: young pages on low-volume keywords (search volume = flag-linked) | Strongest Week 2 gaps; staleness can't be measured honestly (W8) |
| 2026-11-20 | Both signals FALSE → change the rule before building it | A rule on FALSE signals is no better than random; the card counts a clear negative as a win |
| 2026-11-20 | Pick the next clue by a rule set before looking: farthest 500+ bucket from the base rate | Avoid choosing what we hoped to see |
| 2026-11-20 | If every clue is weak, still use the strongest and say the baseline is weak | Skill: a baseline must be "honestly beatable"; testing more clues raises luck |
| 2026-11-21 | Baseline = feedly articles first, by Feb impressions | Content type was the strongest clue (+28.2) |

---

## 9. Open issues / next steps

- [x] ~~ML-07: build the transparent baseline~~ (done 2026-11-21).
- [ ] **ML-08 (next):** train a model on the same frame; it must beat precision@100 51.0% / @500 49.6%.
- [ ] Client-by-client check of the feedly result (one client holds 45% of feedly pages).
- [ ] Test whether combining content type with CTR vs peers improves on the baseline (B11).
- [ ] Explain why page age and search volume were notable in the starter CSV but FALSE in Feb → Mar.
- [ ] Decide what to do with `word_count` (76.5% updated after 1 Mar): test on not-updated pages only, or drop it.
- [ ] Re-check F9 (longer pages declined more) by `content_type`.
- [ ] Investigate `avg_position = 0` (1,251 pages, 42.2% declining) and `word_count = 0` in the warehouse frame.
- [ ] Rule out look-alikes of decline (lane guide §7): consolidation (sibling pages), seasonality, noise.
- [ ] Client-by-client check of every notable signal (Week 6), since a few clients hold most pages.
- [ ] Repeat the Feb → Mar result on at least one other mid-panel month pair.
- [ ] Keep `work/capstone_report.md` updated after every assignment; write the Abstract last.
