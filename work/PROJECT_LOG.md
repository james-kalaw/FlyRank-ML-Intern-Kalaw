# Project Log — FlyRank ML Internship (James Kalaw)

A running record of decisions, findings, and rules for this track. Update it after every
assignment. It feeds `capstone_report.md` and the final paper, and lets any new AI chat
catch up quickly (paste this file in).

> **Data safety:** this file holds **aggregate numbers only** (medians, correlations, counts,
> rates). No raw rows, no `content_id` / `client_id` values, no client details. Keep it that way.

_Last updated: 2026-11-18 (after ML-02)_

---

## 1. Project at a glance

| | |
|---|---|
| **Lane** | Lane 1, Ranking Signal Analysis (provisional; can change until end of Week 4) |
| **Question** | Which common SEO beliefs actually hold up in this data, and which don't? |
| **Who it helps** | A content strategist / SEO lead deciding what their team should spend time on |
| **Decision improved** | Keep acting on an assumption (e.g. "refresh old pages", "write longer") vs. drop it |
| **Unit of analysis** | One page (content item), metrics over a trailing 90-day window |
| **Output** | A signal report: each tested signal gets a verdict (holds / doesn't hold / only shows when grouped) |
| **Method (so far)** | Correlations + grouped comparisons (`.corr()`, `.groupby()`); no trained model required |
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
signal two ways (plain correlation vs. grouped/tier comparison) and flags where they disagree,
because a raw correlation can hide a real but non-linear effect (see F4).

---

## 2. Assignment tracker

Submission = repo URL `https://github.com/james-kalaw/FlyRank-ML-Intern-Kalaw` on each portal card.

| Card | Notebook (`work/notebooks/`) | Status | Key output |
|---|---|---|---|
| warm-up | `notebooks/01_first_look_and_discovery.ipynb` | Done | Pipeline run + Discoveries A–C + own test (F1–F5) |
| warm-up | `notebooks/02_your_first_readable_model.ipynb` | Done | Hand rule vs. tree, leakage demo (F6–F8) |
| ML-02 | `w01_research_question.ipynb` | Submitted 2026-11-18 | Lane 1 chosen; frame; 3 numbers |
| ML-03 | `w02_ml_task_framing.ipynb` | Next | |
| ML-04 | `w03_data_contract.ipynb` | | |
| ML-05 | `w03_feature_leakage_check.ipynb` | Optional (card retired 2026-07-13; core moved to ML-04) | |
| ML-06 | `w04_signal_audit.ipynb` | Optional (card retired 2026-07-13; core moved to ML-07) | Still very relevant to Lane 1 |
| ML-07 | `w04_baseline_score.ipynb` | | |
| ML-08 | `w05_model.ipynb` | | |
| ML-09 | `w06_validation_audit.ipynb` | | |
| ML-10 | `w07_action_playbook.ipynb` | | |
| ML-11 | `capstone.ipynb` + deployed paper | | URL goes in `submission/paper_url.txt` |
| ML-12 | closing cells of `capstone.ipynb` | | 5-min demo outline, social post, employer summary |

---

## 3. Findings so far (starter CSV: 30,000 pages, 32 clients)

All numbers are observed in this sample only. Rates are on a ×100 scale (`ctr = 0.35` means 0.35%).

| # | Belief tested | Result | Verdict | Source |
|---|---|---|---|---|
| F1 | "High search volume means more traffic" | corr(`search_volume`, `impressions_90d`) = **0.001** | Doesn't hold | NB01 Discovery A; ML-02 |
| F2 | "Better position means more clicks" (grouped) | Mean CTR, pages with ≥100 impressions: page_1 **0.35%**, top_3 **0.33%**, striking **0.26%**, page_3_5 **0.14%**, deep **0.06%** (~6× drop page_1 → deep) | Holds, when grouped | NB01 Discovery B |
| F3 | "Longer content performs better" | Median `word_count`: down 2,909 · stable 2,912 · up 2,848 · flat 2,698 · new 2,239 | Doesn't hold (down ≈ up) | NB01 Discovery C |
| F4 | "Better position means more clicks" (raw correlation) | corr(`avg_position`, `ctr`) = **-0.073** | Looks absent; contradicts F2 → relationship is non-linear | NB01 own test; ML-02 |
| F5 | "Old / stale content declines" | Median `content_age_days`: down **216** · flat 231 · new 279 · up 292 · stable **300** | Backwards: declining pages are youngest | NB01 own test; ML-02 |
| F6 | Share of pages by trend | down **54.2%** · stable 19.9% · up 14.6% · new 7.5% · flat 3.8% | Base rate for "declining" = 54.2% | ML-02 |

**Caveats to carry forward**
- F2: the `top_3` tier has low volume per page (data dictionary: median ~53 impressions/90d), so
  one click moves its CTR a lot. Always state the volume floor with tier averages.
- F4: `avg_position = 0` means "no data" (1,205 rows). F4 did not filter those out; redo it with
  them excluded, and also try a rank (Spearman) correlation.
- F5: possible explanations (not tested): survivorship (old pages still around are the durable
  ones); recently published pages spiking then settling. Worth a cut by `content_type`.

### Model results seen in the warm-ups (context, not my lane's method)

| Setting | Metric | Hand rule | Model |
|---|---|---|---|
| NB01 pipeline, client-holdout | Precision@50 | 0.240 (baseline score) | 0.740 (random forest) |
| NB02 in-sample, "stale × visible" rule vs. depth-2 tree | Precision@20 / @50 | 0.900 / 0.680 | 0.550 / 0.600 |
| NB02 depth-4 tree, in-sample | Precision@20 / @50 | — | 0.600 / 0.680 (ties the rule at @50) |
| NB02 client-holdout (GroupShuffleSplit, seed 42) | Precision@50 | 0.620 | 0.560 |
| NB02 leaky tree (used `trend_pct`) | Precision@50 | — | 1.000 (fake: leakage) |

Lessons: a sharp human rule can win at the very top of a list; a bigger model can win overall;
a perfect score is a red flag; client-holdout lowered both scores (rule margin 0.08 in-sample →
0.06 holdout) but the rule still led.

---

## 4. Data facts and traps (from `docs/data-dictionary.md`, `skills/flyrank/flyrank-data`)

- One row = one page; 44 columns; 32 clients; trailing 90-day metrics plus last-30 vs prev-30 windows.
- **Label columns, never inputs:** `trend_direction`, `trend_pct` (`trend_direction` is bucketed
  from `trend_pct`: up > +20%, down < -20%).
- **IDs** (`content_id`, `client_id`): grouping/splitting only, never features, never mapped back.
- Rate columns are ×100 percentages: `ctr`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`, `trend_pct`.
- `avg_position = 0` means no data. `scroll_rate` and `ai_traffic_pct` can exceed 100.
- Missingness follows `content_type` (one type has ~100% missing keyword data). Don't blind
  `fillna(0)`; use has_-flags or filter.
- Week 3+: warehouse on Hugging Face (~79M daily rows, gated). Iterate on a mid-panel month
  (e.g. `month=2026-03`), never on `_sample` (it is the final month). Check
  `dim_clients.gsc_data_start` before any time window. HF token only via Colab Secrets.

---

## 5. Working rules

**Data and AI**
- Share only code, errors, column names, and aggregate outputs with AI tools.
  Never raw rows (`df.head()`), the CSV, or warehouse extracts. (`DATA_USE.md` rule 2.)
- Never commit datasets; CI blocks CSV/parquet/zip outside the starter files.

**Repo**
- Build everything in `work/`. Don't edit `scripts/` (copy into `work/scripts/` instead).
- Every `work/notebooks/` notebook needs a setup cell that clones the repo if
  `data/raw/content_refresh_anonymized.csv` is missing (see ML-02 Section 1).
- Commit notebooks **with outputs** (Run all first). CI checks this at the end.
- Colab "Save a copy in GitHub": always check the **Repository** dropdown says
  `FlyRank-ML-Intern-Kalaw` (it once defaulted to another repo), and untick "Include a link to Colab".
- `work/capstone_report_template.md` exists: copy to `work/capstone_report.md` and fill as we go.
  Its sections mirror the grading rubric.

**Claims**
- Use: observed, measured, directional, decision-support. Report the base rate (54.2% declining)
  next to any precision number.
- Never: causes, proves, "predicts Google", generalizes beyond these clients.

---

## 6. Decision log

| Date | Decision | Why |
|---|---|---|
| 2026-11-18 | Lane 1 (Ranking Signal Analysis) | Builds on tools already working (`.corr`, `.groupby`); no model to break; Week 1–2 tests already produced 5 non-obvious findings |
| 2026-11-18 | Stay in one AI conversation; keep this log as the backup | Context needed through capstone; log protects against long-chat drift |
| 2026-11-18 | Share aggregates only with AI tools | `DATA_USE.md` rule 2 |

---

## 7. Open questions / next steps

- [ ] Run the data profile cell (summaries only) and record column ranges + missingness here.
- [ ] ML-03: frame Lane 1 as an ML task (which signals, which outcome, which comparison method).
- [ ] Redo F4 excluding `avg_position = 0`; add Spearman.
- [ ] Cut F5 by `content_type` to test the survivorship idea.
- [ ] Decide whether the "two-method scorecard" is the capstone's core method.
- [ ] Start `work/capstone_report.md` from the template.
