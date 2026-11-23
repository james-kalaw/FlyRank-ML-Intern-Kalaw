# Project Log — FlyRank ML Internship (James Kalaw)

A running record of decisions, findings, and rules for this track. Update it after every
assignment. It feeds `capstone_report.md` and the final paper, and lets any new AI chat
catch up quickly (paste this file in, or point the assistant to it).

> **Data safety:** this file holds **aggregate numbers only** (medians, rates, counts, AUC).
> No raw rows, no `content_hash_id` / `client_hash_id` values, no client details. Keep it that way.
> "Client N" labels are ranks by page count inside the Feb → Mar frame (client 1 = most pages), not IDs.

_Last updated: 2026-11-23 (after ML-08)_

---

## 0. Start here (new conversation or new reader)

**Status on 2026-11-23:** Weeks 1–5 are done: ML-02, ML-03, ML-04, ML-07, ML-08 (the five required
cards). **Next: Week 6, ML-09** (`work/notebooks/w06_validation_audit.ipynb`), then the capstone.

**Read in this order:**
1. `CLAUDE.md` → `skills/README.md` (the router: load **one** skill per task, plus
   `skills/flyrank/flyrank-data/SKILL.md` whenever the task touches the data).
2. This file (all findings, decisions, open issues, and the Week 6 plan in Section 10).
3. `work/capstone_report.md` (the rubric-shaped draft).
4. The notebook for the current week.

**Where everything is:**

| What | Where |
|---|---|
| My notebooks (run, with outputs) | `work/notebooks/w01_…` to `w05_model.ipynb` |
| Metrics receipts (committed) | `work/outputs/baseline_metrics.json` (ML-07), `work/outputs/model_metrics.json` (ML-08) |
| Baseline review queue | `work/outputs/baseline_action_score.csv` (git-ignored; rebuilt by `w04_baseline_score.ipynb`) |
| Capstone draft | `work/capstone_report.md` (template: `work/capstone_report_template.md`) |
| Assignment index (with Colab badges) | `work/README.md` |
| Research paper for ML-09 Section 1 | `docs/flyrank-seo-research-march-2026.pdf` (check it is the paper linked on the ML-09 card) |
| Lane guide, data dictionary, framework | `docs/ml-intern-dataset-and-lane-guide.md`, `docs/data-dictionary.md`, `docs/ml-core-foundation-framework.md` |
| Data rules | `DATA_USE.md` |

**How each assignment is done (working with the AI assistant):**
1. Read the card's "What done looks like" and the skill the router names. Explain the plan in simple
   words **before** building.
2. The assistant builds the notebook and tests it on **synthetic data with the real schema** (it never
   loads the real dataset). It pushes a branch and opens a pull request.
3. James merges the PR, opens the notebook in Colab (HF token in Colab Secrets as `HF_TOKEN`), and runs
   it top to bottom on the real warehouse.
4. James shares the run notebook with the assistant: code, errors, column names, and **aggregate outputs
   only** (`DATA_USE.md` rule 2). Check it first: no `hf_` token string, no raw rows with IDs.
5. The assistant explains the results section by section; the "My result" lines are written and checked
   against the outputs; the run notebook (with outputs) and any metrics JSON are committed.
6. Check the notebook item by item against the card, then submit the repo URL on the card.
7. Update this log and `capstone_report.md`.

Assistant setup notes: clone without the data folder
(`git clone --filter=blob:none --sparse <repo>` then
`git sparse-checkout set .github docs notebooks scripts skills submission work`). Branch per change;
open PRs with the REST API (`gh api repos/<owner>/<repo>/pulls`). Commit as `james-kalaw`.

**Before starting Week 6, check:**
- [ ] PR #13 (ML-08 run + this log + report) is merged into `main`.
- [ ] Portal: ML-08 submitted; ML-03, ML-04, ML-07 show as submitted.
- [ ] The research paper linked on the ML-09 card has been read (needed for ML-09 Section 1).
- [ ] The ML-09 card's "What done looks like" is pasted into the new conversation (the card text is not in the repo).

---

## 1. Project at a glance

| | |
|---|---|
| **Lane** | Lane 1, Ranking Signal Analysis. **Final:** kept at the lane lock (end of Week 4) |
| **Question** | Which common SEO beliefs actually hold up in this data, and which don't? |
| **Who it helps** | A content strategist / SEO lead deciding what their team should spend time on |
| **Decision improved** | Keep acting on an assumption (e.g. "refresh old pages", "write longer") vs. drop it; which pages to review first |
| **Unit of analysis** | One page. Starter CSV: 90-day totals. Warehouse: features from Feb 2026, outcome from Mar 2026 |
| **Outcome** | `is_declining`: impressions dropped more than 20% (warehouse: per day, Feb → Mar) |
| **Metrics** | ML-03 (starter CSV): decline-rate gap, top 25% minus bottom 25% (notable ≥ 5, weak 2–5, none < 2). ML-07: bucket verdicts (CONFIRMED / OPPOSITE / MIXED / FALSE). ML-07/08: precision@K next to the base rate and random picking. All set in advance |
| **Output** | A signal report (each tested signal gets a verdict and a direction) plus a ranked review queue as decision support |
| **Baseline (ML-07)** | Feedly articles first, ordered by Feb impressions. Precision@100 **51.0%**, @500 **49.6%** vs base rate 25.1% (random ~25%) |
| **Model (ML-08)** | Logistic Regression on 8 features, tested on unseen clients (GroupKFold by client). Precision@100 **54.0%**, @500 **51.0%**, AUC 0.545. Beats the baseline by a small, unsteady margin |
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
signal more than one way (end-to-end gap, tier table, interaction, model importance) and flags where
they disagree. Evidence it matters: position looked "weak" by the gap (−3.7) but is hill-shaped in the
tier table (F13); page age / search volume looked notable in the same-window starter CSV but were FALSE
in the Feb → Mar warehouse test (B1, B2); and in ML-08 the model's directions agreed with the Week 4
bucket tables, while page age had negative importance (M8).

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
| ML-05 | `w03_feature_leakage_check.ipynb` | Optional (card retired 2026-07-13; core moved to ML-04) | Not done |
| ML-06 | `w04_signal_audit.ipynb` | Optional (card retired 2026-07-13; core moved to ML-07) | Not done |
| ML-07 | `w04_baseline_score.ipynb` | Complete 2026-11-21 (confirm portal submission) | Signal checks, EDA charts, feedly-first rule, top-10 review (B1–B12) |
| ML-08 | `w05_model.ipynb` | Complete 2026-11-23 (run in Colab; merge PR #13, then submit) | LR vs RF vs GB vs baseline, grouped by client (M1–M12) |
| ML-09 | `w06_validation_audit.ipynb` | **Next** (plan in Section 10) | Paper claims audit; honest split before/after; leakage audit; claim rewrite |
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
≤ 1 Feb; page created before 1 Feb; ≥ 100 Feb impressions; `gsc_data_available IS TRUE`; pages with
no usable March rows dropped (label unknown).
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

The ML-08 run (2026-11-23) rebuilt the same frame and reproduced W5 and W6 exactly.

**Why 25.1% ≠ 54.2%:** different measurement (full month vs month, per day, ≥100 impressions,
mid-panel instead of end of panel). The two base rates must not be compared directly.

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
| B9 | Client concentration | Flagged pages come from **7** clients; one client (client 11) holds **45.0%** (317 pages); 6 of the top 10 are from that client |
| B10 | Top-10 review | **7 of 10** declined. Shared risks: a February spike returning to normal, traffic moving to a sibling page (not checked) |
| B11 | Weak picks (feedly misses vs hits, medians) | CTR vs peers **0.60** vs **0.30**; Feb impressions 291 vs 232; position 8.75 vs 7.98; age 221 vs 206 days |
| B12 | Ordering | Sorting by Feb impressions did not help: top 50 = 46.0% vs 53.3% for all feedly pages. Content type does the work |

Leakage check passed: the rule reads no March data, label, excluded column, or product flag.
`content_type` comes from the present-day `dim_content` snapshot; assumed stable since publishing.

---

## 6. Findings: model, Feb → Mar 2026 warehouse frame (ML-08, Week 5)

**M1. Setup** (notebook `w05_model.ipynb`, run in Colab on 2026-11-23, DuckDB 2.0.1, scikit-learn 1.6.1, seed 42)
- **Frame:** the same 65,756 pages, 26 clients, label, and base rate (25.1%) as ML-04 and ML-07.
- **Question:** can a learned model rank declining pages better than the baseline rule, and which February
  signals does it lean on?
- **8 features (all known on 1 March):**

| Feature | Made from | Note |
|---|---|---|
| `is_feedly` | `content_type == "feedly article"` | strongest Week 4 clue (B3) |
| `ctr_vs_peers` | Feb CTR ÷ peer CTR of the same position group | peer CTR = training clients' clicks ÷ impressions per group, rebuilt inside every round (no test-client leak) |
| `feb_position` | Feb avg position (impression-weighted) | 0 → missing |
| `position_is_zero` | Feb avg position = 0 | flags the quirk (1,251 pages) |
| `log_feb_impressions` | log10(Feb impressions) | a few pages are huge |
| `content_age_days` | days from creation to 1 Mar | |
| `log_search_volume` | log10(search volume + 1) | very skewed |
| `volume_is_missing` | search volume is missing | missing bucket declined at 42.9% (B2) |

  Position groups (for peer CTR and the error table): [0, 1), [1, 3.5), [3.5, 10.5), [10.5, 20.5),
  [20.5, 50.5), [50.5, ∞) on the Feb avg position (= Week 4's 0 / 1–3 / 4–10 / 11–20 / 21–50 / 51+).
- **Left out:** `word_count` (snapshot, W8), all March data, the label, IDs, snapshot dates/flags, GA4.
- **Models (simple → strong):** Logistic Regression (median impute + standardize, `max_iter=2000`);
  Random Forest (median impute, 200 trees, `min_samples_leaf=50`); Gradient Boosting
  (`HistGradientBoostingClassifier`, `learning_rate=0.05`, `max_iter=300`, handles missing values itself).
  No voting ensemble (dropped to keep it simple).
- **Split:** `GroupKFold(n_splits=5)` by `client_hash_id`. Every page is scored by models that never saw its
  client; the 5 rounds' out-of-round scores are pooled into one ranking per model.
- **Metrics:** precision@10/50/100/500, AUC, average precision; base rate; random picking (20 shuffles,
  seeded); the baseline recomputed in the same run.
- **Selection rule (written before looking):** the simplest model (order LR → RF → GB) whose precision@100
  **and** precision@500 are both within 2 points of the best model.
- Training time: about 1.5 minutes for all 5 rounds (29 + 15 + 15 + 14 + 17 s).

**M2. The 5 test rounds are very uneven**

| Round | Test clients | Test pages | Base rate | Feedly pages |
|---|---|---|---|---|
| 1 | 1 | 17,529 | 25.9% | 0 |
| 2 | 1 | 14,773 | 30.9% | 0 |
| 3 | 4 | 11,177 | 20.9% | 109 |
| 4 | 11 | 11,137 | 23.0% | 325 |
| 5 | 9 | 11,140 | 22.7% | 270 |

Rounds 1 and 2 are the two biggest clients alone and have no feedly pages, so the feedly clue cannot help there.

**M3. Comparison table** (all 65,756 pages, same labels; precision in %)

| Ranking | P@10 | P@50 | P@100 | P@500 | AUC | Avg precision |
|---|---|---|---|---|---|---|
| Base rate | 25.1 | 25.1 | 25.1 | 25.1 | 0.500 | 0.251 |
| Random picking (20 shuffles) | 22.5 | 24.2 | 25.4 | 24.2 | 0.500 | 0.251 |
| Baseline rule (feedly first) | 70.0 | 46.0 | 51.0 | 49.6 | 0.486 | 0.258 |
| **Logistic Regression** | **80.0** | **50.0** | **54.0** | **51.0** | 0.545 | 0.281 |
| Random Forest | 70.0 | 46.0 | 47.0 | 42.2 | **0.567** | **0.304** |
| Gradient Boosting | 50.0 | 48.0 | 35.0 | 30.2 | 0.554 | 0.291 |

The baseline's precision matches Week 4 exactly (B7). Its AUC is below 0.5 likely because the 99% of
pages that are not feedly are ordered by Feb impressions, and bigger pages declined slightly less (B6),
so the rest of its list is slightly backwards.

**M4. Precision@50 inside each test round (%)**

| Round | Pages | Base | Baseline rule | LR | RF | GB |
|---|---|---|---|---|---|---|
| 1 | 17,529 | 25.9 | 44.0 | 20.0 | 40.0 | 68.0 |
| 2 | 14,773 | 30.9 | 38.0 | 52.0 | 44.0 | 50.0 |
| 3 | 11,177 | 20.9 | 54.0 | 66.0 | 44.0 | 14.0 |
| 4 | 11,137 | 23.0 | 42.0 | 50.0 | 44.0 | 26.0 |
| 5 | 11,140 | 22.7 | 54.0 | 64.0 | 50.0 | 26.0 |

LR beat the baseline in rounds 2–5 and lost badly in round 1 (one client, no feedly pages).
GB swung from 68.0 (round 1) to 14.0 (round 3): unstable on new clients.

**M5. Selection rule result: Logistic Regression.** It was the best model at both @100 (54.0) and @500
(51.0); RF (47.0 / 42.2) and GB (35.0 / 30.2) were not within 2 points. LR vs baseline: **+3.0 points
@100, +1.4 @500**, AUC 0.545 vs 0.486, average precision 0.281 vs 0.258. The notebook prints "beats the
baseline at both @100 and @500: True", but the margin is small: for 100 pages, one standard error of a
precision near 50% is about 5 points, so chance could explain it. RF ranked the whole list best (AUC, average
precision) but was worse at the top, which is what an editor reads, so the pre-set rule did not pick it.

**M6. What the top of the list is made of:** LR's top 500 are **all feedly pages** (500 of the 704), from
**7 clients**, and one client holds **57.8%** (the baseline's flagged list: 45.0% from one client). So the
model mostly **re-orders the feedly pages** (using position and CTR vs peers) rather than finding new kinds
of declining pages, and leans even more on one client.

**M7. Permutation importance** (LR; drop in average precision when one feature is shuffled; mean of 5 test
rounds, 5 repeats each)

| Feature | Drop |
|---|---|
| `feb_position` | 0.0194 |
| `is_feedly` | 0.0161 |
| `ctr_vs_peers` | 0.0107 |
| `log_search_volume` | 0.0014 |
| `volume_is_missing` | 0.0011 |
| `position_is_zero` | 0.0000 |
| `log_feb_impressions` | −0.0005 |
| `content_age_days` | **−0.0133** |

**M8. LR weights** (standardized, mean of 5 rounds; + = more likely to decline): `is_feedly` +0.104 ·
`volume_is_missing` +0.047 · `content_age_days` +0.026 · `position_is_zero` +0.017 · `log_search_volume`
−0.031 · `log_feb_impressions` −0.108 · `ctr_vs_peers` −0.143 · `feb_position` −0.267.

Reading M7 + M8 (observed associations on unseen clients, not causes):
- **Top 3: February position, feedly, CTR vs peers.** Directions match Week 4: a smaller position number
  (higher rank) went with more decline (B4); feedly went with more decline (B3); fewer clicks than peers
  went with more decline (B5).
- **Page age had negative importance:** shuffling it made the model *better* on new clients. Whatever the
  model learned about age from training clients did not carry over. Consistent with B1 (FALSE).
- Feb impressions and search volume added almost nothing. The `position_is_zero` flag was not used.

**M9. Where the model is most wrong** (real % declining vs the model's average predicted %)

| Group | Pages | Real | Predicted | Gap (pts) |
|---|---|---|---|---|
| feedly article | 704 | 53.3 | 53.5 | +0.2 |
| keyword article | 65,047 | 24.8 | 23.9 | −0.9 |
| position [0, 1) ("0" quirk) | 1,251 | 42.2 | 26.4 | **−15.8** |
| position [1, 3.5) | 12,026 | 28.1 | 26.2 | −1.9 |
| position [3.5, 10.5) | 32,336 | 26.7 | 25.7 | −1.0 |
| position [10.5, 20.5) | 11,806 | 23.3 | 23.6 | +0.3 |
| position [20.5, 50.5) | 7,411 | 13.7 | 17.5 | +3.8 |
| position [50.5, ∞) | 926 | 21.7 | 8.5 | **−13.2** |

- Content type is almost perfectly calibrated.
- **"Position 0" pages are under-predicted** (26.4% vs 42.2%): the model did not use the flag.
- **Rank 51+ is under-predicted** (8.5% vs 21.7%): decline falls from rank 1 to 50 and then rises again,
  and a straight-line model cannot bend back up.

**M10. Six wrong cases** (described without IDs)
- **Most sure, did not decline:** 3 feedly pages from client 11, each with 111–184 Feb impressions, **zero
  clicks** (CTR vs peers 0.0), position 4.8–6.6, 202–251 days old; model 69.6–70.6% sure.
  Every clue pointed to "decline," but they held. A guess: pages with few impressions and no clicks may
  already be at a stable low level.
- **Declined, scored lowest:** 2 feedly pages from client 15 with **CTR vs peers of 62.3 and 61.2** (scored
  0.0%) and 1 keyword page from client 6 at position 65.2 with **CTR vs peers 26.8** (scored 0.1%). Extreme
  CTR-vs-peers values push a straight-line model to almost 0%. These high values may be a data quirk (not checked).

**M11. Leakage sanity check:** no feature looks suspiciously perfect; LR's AUC is 0.545, far from the
0.994 of the deliberate leak in W7. The models read no March data.

**M12. Run notes:** the first Colab run crashed in the error-analysis cell (`KeyError: ... not in index`)
because `frame.loc[...]` was given row positions, and the frame index has gaps after the 3,605
no-March pages were dropped. Fixed with `frame.iloc[...]`; the synthetic test now forces index gaps.
The committed notebook is the full run; the code matches the fixed version.

**Honest summary of ML-08:** on clients it never saw, Logistic Regression ranked declining pages at
precision@100 54.0% and @500 51.0% (base rate 25.1%, random about 25%), slightly above the baseline rule
(51.0% / 49.6%). The gain is small, not steady across rounds, and mostly comes from re-ordering feedly
articles. The simplest model won; added complexity (RF, GB) did not earn its place at the top of the list.

---

## 7. Data facts and traps

**Starter CSV** (`docs/data-dictionary.md`, `skills/flyrank/flyrank-data`)
- Label columns, never inputs: `trend_direction`, `trend_pct`. Outcome ingredients: six `*_last_30d` / `*_prev_30d`.
- IDs are pseudonyms: grouping/splitting only. Rate columns are ×100. `avg_position = 0` = no data.
- Missingness follows `content_type`; don't blind `fillna(0)`.

**Warehouse** (release build v20260703, Hugging Face, gated)
- Daily grain: report_date × client × content. Position column: `gsc_avg_position` (aggregate it weighted by impressions).
- `gsc_avg_position = 0` with impressions: 1,251 frame pages (42.2% declining) — a quirk; keep it as its own bucket/flag.
- DuckDB `CREATE SECRET` needs DuckDB 1.1+. The setup cell installs `duckdb>=1.1` and stops with a message
  if an older copy is already loaded: then Runtime → Restart session, and Run all again.
- `hf://` paths don't support `{a,b}` globs: pass a list of partition paths to `read_parquet([...])`.
- Flags can be TRUE / FALSE / NULL: always filter with `IS TRUE`.
- Unbalanced panel: check `dim_clients.gsc_data_start` before any window.
- `dim_content` is a present-day snapshot: `content_updated_date`, `last_optimized_date`, `is_deleted`,
  `is_published` can describe the future relative to a past decision day. `content_type` and
  `search_volume` also come from it (assumed stable).
- **Population choice to disclose:** pages with no usable March rows (3,605) are dropped because their label
  is unknown. This uses information from the outcome window (skill checklist: say so in limitations).
- Never develop on `_sample` or June 2026 (sealed test month). Iterate on mid-panel months.
- HF token only through Colab Secrets (`HF_TOKEN`). Never in a cell, never in chat. When checking a
  notebook, search for `hf_` followed by a long random string; `HF_TOKEN` matches are only the variable name.

**Modeling (ML-08)**
- After dropping rows the frame index has gaps: use `.iloc` with positions (e.g. from `np.argsort`), not `.loc` (M12).
- Peer statistics (like peer CTR) must be computed from training clients only, inside each round.
- `ctr_vs_peers` is unbounded (up to about 60× peers on small pages) and can dominate a linear model (M10).
- Pooled out-of-round scores mix 5 models whose scores may be on slightly different scales; the per-round
  table (M4) is the check on that.
- Tree models can shift a little between scikit-learn versions; the Week 5 run used 1.6.1.

---

## 8. Working rules

**Data and AI**
- Share only code, errors, column names, and aggregate outputs with AI tools. Never raw rows, the CSV,
  or warehouse extracts (`DATA_USE.md` rule 2). Notebooks print summaries only.
- Code is tested on synthetic data with the real schema before it touches the real data.
- No client names, domains, URLs, or queries anywhere; clients appear only as "client N".

**Repo**
- Build everything in `work/`. Don't edit `scripts/`.
- Commit notebooks **with outputs** (Run all first). CI checks this. Never commit datasets
  (`work/**/*.csv` is git-ignored); small metrics JSONs in `work/outputs/` are committed as receipts.
- Colab "Save a copy in GitHub": check the **Repository** dropdown says `FlyRank-ML-Intern-Kalaw`,
  the path is `work/notebooks/<exact name>.ipynb`, and "Include a link to Colab" is unticked.
- Merge a results PR only after the **run** version of the notebook is in it, so an unrun copy never
  overwrites the run.
- Before submitting, check the notebook item by item against the card's "What done looks like".
- GitHub's notebook preview does not draw `- [x]` checkboxes; the file still has them.

**Claims** (`skills/writing-honest-claims`)
- Use: observed, measured, directional, decision-support. Report the base rate next to any score.
- Never: causes, proves, "predicts Google", generalizes beyond these clients.
- Report negative results (e.g. page age) as results.

---

## 9. Decision log

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
| 2026-11-21 | ML-08 models: Logistic Regression, Random Forest, Gradient Boosting; no voting ensemble | Skill: "Logistic Regression, then Random Forest"; GB as the strong option; four models judged too complex |
| 2026-11-21 | ML-08 features: 8 (adds `is_feedly`, `ctr_vs_peers`, missing/zero flags, logs); `word_count` left out | Week 4 clues (B3, B5, B11); snapshot problem (W8) |
| 2026-11-21 | Split: GroupKFold(5) by client; peer CTR rebuilt from training clients in each round | Test on unseen clients; one client holds 45% of feedly pages; no leak through peer statistics |
| 2026-11-21 | Selection rule set before looking: simplest model within 2 points of the best at P@100 and P@500 | Don't reward complexity alone; avoid picking the luckiest model |
| 2026-11-23 | Chosen model: Logistic Regression | Result of the pre-set rule (also best at both K) |
| 2026-11-23 | Do not change features after seeing test errors (e.g. capping `ctr_vs_peers`); test such fixes only on a new month pair | Fixing what the test showed would tune the model to the test |

---

## 10. Open issues / next steps

### Week 6 — ML-09 Validation and Research Claim Audit (`w06_validation_audit.ipynb`) — NEXT

Skeleton sections (from the notebook): 1) two paper findings + methodology questions; 2) my model under an
honest split (before/after); 3) leakage audit on the final feature set; 4) claim rewrite; self-check.
Skills: `hunting-leakage-and-validating` (+ `flyrank/flyrank-data`) for Sections 2–3;
`writing-honest-claims` for Sections 1 and 4. **Proposed plan, to confirm against the ML-09 card first:**

- [ ] **Section 1:** pick two findings from the FlyRank research paper (`docs/flyrank-seo-research-march-2026.pdf`,
  if it matches the card's link). For each: where the label comes from; does the validation design carry the
  claim? Constructive tone ("how to make it stronger").
- [ ] **Section 2 (before/after):** "before" = the same Week 5 Logistic Regression on a **random** (not grouped)
  5-fold split of the same frame; "after" = the grouped-by-client result (P@100 54.0, P@500 51.0, AUC 0.545,
  M3). Report both and explain the gap. Also a **time-aware** check: train on the Feb → Mar frame, test on a
  later mid-panel month pair built the same way (e.g. Mar → Apr; never June 2026), against the baseline and
  base rate of that new frame.
- [ ] Inside Section 2 or as an extra: **client-by-client check of the feedly result** (one client holds 45%
  of feedly pages and 57.8% of LR's top 500).
- [ ] Optional, decided in advance: test a cap on `ctr_vs_peers` (and a position-0 handling) **only on the new
  month pair**, and report it whether it helps or not.
- [ ] **Section 3 (leakage audit on the 8 features):** timeline check (all features ≤ 28 Feb, label in March);
  re-run the deliberate leak (add March impressions → AUC should jump toward 1.0, then remove); peer CTR from
  training clients only; snapshot columns (`content_type`, `search_volume`); population choice (3,605 dropped
  pages) disclosed; no product flags as features.
- [ ] **Section 4 (claim rewrite):** take my boldest sentence (e.g. "feedly articles decline twice as often")
  and rewrite it with the claim ladder: observed, in this data, with n, base rate, and the client-concentration caveat.

### Capstone backlog (Weeks 7–8)

- [ ] Explain why page age and search volume were notable in the starter CSV but FALSE in Feb → Mar
  (same-window vs before → after measurement is the leading guess, not yet tested).
- [ ] Investigate `avg_position = 0` (1,251 pages, 42.2% declining; the model under-predicts it by 15.8 points)
  and `word_count = 0` in the warehouse frame.
- [ ] Rank 51+ bend: decline rises again after rank 50 (B4, M9).
- [ ] Rule out look-alikes of decline (lane guide §7): consolidation (sibling pages), seasonality, noise,
  a February spike returning to normal.
- [ ] Ask the mentor what "feedly article" means (not documented in the repo; the name suggests news-feed-based content — unconfirmed).
- [ ] ML-10 action playbook (`w07_action_playbook.ipynb`): ranked actions, confidence, limits.
- [ ] ML-11 paper (`capstone.ipynb` + deployed page; URL in `submission/paper_url.txt`); ML-12 closing cells.
- [ ] Keep `work/capstone_report.md` updated after every assignment; write the Abstract last.

### Closed

- [x] ML-07: build the transparent baseline (2026-11-21).
- [x] ML-08: model on the same frame vs the baseline (2026-11-23): LR wins slightly (M5).
- [x] Test whether combining content type with CTR vs peers improves on the baseline (B11): yes, slightly (M5, M6).
- [x] Decide on `word_count`: left out of the model (snapshot problem, W8). F9's re-check by `content_type`
  is dropped for the same reason.

---

## 11. How to reproduce (with the numbers to expect)

1. Open the notebook from `work/README.md` (Colab badge) or Colab → GitHub → this repo.
2. Colab Secrets: add the Hugging Face READ token as `HF_TOKEN` and turn on notebook access (Week 3+).
   Gate access to `FlyRank/internship-warehouse` must be approved on Hugging Face.
3. Runtime → Run all. If the setup cell says DuckDB is too old: Runtime → Restart session, then Run all.
4. Compare with the committed outputs:

| Notebook | Data | Check these numbers |
|---|---|---|
| `w01_research_question` | starter CSV | F1, F4, F5, F6 |
| `w02_ml_task_framing` | starter CSV | F7–F15; base rate 54.2% |
| `w03_data_contract` | warehouse Feb + Mar | W1–W8; frame 65,756 pages / 26 clients / 3,605 dropped / 25.1%; leak trap AUC 0.594 → 0.994 |
| `w04_baseline_score` | warehouse Feb + Mar | B1–B12; P@10/50/100/500 = 70.0 / 46.0 / 51.0 / 49.6; writes `baseline_metrics.json` (+ the git-ignored CSV queue) |
| `w05_model` | warehouse Feb + Mar | M2–M9; LR chosen; LR P@100 54.0 / P@500 51.0; writes `model_metrics.json` |

Seeds: 42 everywhere (splits, models, random-picking shuffles). Week 5 environment: Colab, DuckDB 2.0.1,
scikit-learn 1.6.1. Week 3–5 notebooks read only the `month=2026-02` and `month=2026-03` partitions.
