# Project Log — FlyRank ML Internship (James Kalaw)

A running record of decisions, findings, and rules for this track. Update it after every
assignment. It feeds `capstone_report.md` and the final paper, and lets any new AI chat
catch up quickly (paste this file in).

> **Data safety:** this file holds **aggregate numbers only** (medians, rates, counts, AUC).
> No raw rows, no `content_hash_id` / `client_hash_id` values, no client details. Keep it that way.

_Last updated: 2026-11-20 (after ML-04)_

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
Evidence it matters: position looked "weak" by the gap (−3.7) but is hill-shaped in the tier table (F13).

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
| ML-07 | `w04_baseline_score.ipynb` | **Next** | |
| ML-08 | `w05_model.ipynb` | | |
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

## 5. Data facts and traps

**Starter CSV** (`docs/data-dictionary.md`, `skills/flyrank/flyrank-data`)
- Label columns, never inputs: `trend_direction`, `trend_pct`. Outcome ingredients: six `*_last_30d` / `*_prev_30d`.
- IDs are pseudonyms: grouping/splitting only. Rate columns are ×100. `avg_position = 0` = no data.
- Missingness follows `content_type`; don't blind `fillna(0)`.

**Warehouse** (release build v20260703, Hugging Face, gated)
- Daily grain: report_date × client × content. Position column: `gsc_avg_position` (aggregate weighted by impressions).
- Flags can be TRUE / FALSE / NULL: always filter with `IS TRUE`.
- Unbalanced panel: check `dim_clients.gsc_data_start` before any window.
- `dim_content` is a present-day snapshot: `content_updated_date`, `last_optimized_date`, `is_deleted`,
  `is_published` can describe the future relative to a past decision day.
- Never develop on `_sample` or June 2026 (sealed test month). Iterate on mid-panel months.
- HF token only through Colab Secrets (`HF_TOKEN`). Never in a cell, never in chat.

---

## 6. Working rules

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

## 7. Decision log

| Date | Decision | Why |
|---|---|---|
| 2026-11-18 | Lane 1 (Ranking Signal Analysis) | Builds on tools already working; Week 1–2 tests already produced non-obvious findings |
| 2026-11-18 | Share aggregates only with AI tools | `DATA_USE.md` rule 2 |
| 2026-11-18 | Outcome = `is_declining`; metric = top-vs-bottom-25% decline-rate gap, thresholds 5 / 2 points | Plain English; grouped comparison; set before looking |
| 2026-11-19 | Warehouse design: Feb 2026 features → Mar 2026 outcome, per-day label, ≥100 Feb impressions | Features before outcome (fixes Week 2's same-window limit); per day because Feb has 28 days and Mar 31 |
| 2026-11-19 | 5 features: content age, word count, search volume, Feb avg position, Feb impressions | Strongest Week 2 signals + visibility before decision day |
| 2026-11-20 | Keep Lane 1 at the lane lock | Continuous story since Week 1; foundation already built |
| 2026-11-20 | Follow FlyRank rules strictly; check every notebook against its card | Goal: capstone graded Pass |

---

## 8. Open issues / next steps

- [ ] **ML-07 (next):** build the transparent baseline.
- [ ] Decide what to do with `word_count` (76.5% updated after 1 Mar): test on not-updated pages only, or drop it.
- [ ] Re-check F9 (longer pages declined more) by `content_type`.
- [ ] Investigate `avg_position = 0` and `word_count = 0` in the warehouse frame.
- [ ] Rule out look-alikes of decline (lane guide §7): consolidation (sibling pages), seasonality, noise.
- [ ] Client-by-client check of every notable signal (Week 6), since a few clients hold most pages.
- [ ] Repeat the Feb → Mar result on at least one other mid-panel month pair.
- [ ] Keep `work/capstone_report.md` updated after every assignment; write the Abstract last.
