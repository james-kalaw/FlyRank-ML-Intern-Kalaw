# Capstone Report — Ranking Signal Analysis

- **Author:** James Kalaw
- **Lane:** Lane 1, Ranking Signal Analysis
- **Repo:** https://github.com/james-kalaw/FlyRank-ML-Intern-Kalaw
- **Date:** started 2026-11-20 (draft, updated after every assignment)

> Copied from `capstone_report_template.md`. Sections 1–8 follow the Pass / Needs-Work rubric.
> Sections 0 and 9 are paper sections. Lines marked **To do** are filled in later weeks.
> Details and exact numbers live in `work/PROJECT_LOG.md`.

## 0. Abstract

**To do (write last, Week 8).** Five sentences: question → data → method → headline result →
what the output is for.

## 1. Problem framing

**Decision:** whether a content or SEO team keeps acting on common beliefs (for example "refresh
old pages" or "write longer articles"), or moves its time to signals that actually go with pages
holding or losing search visibility.

**Who acts:** a content strategist or SEO lead deciding what to tell the team to review first.

**Unit of analysis:** one page. In the warehouse, each page gets features from February 2026 and
an outcome from March 2026, with 1 March 2026 as the decision day.

**Output:** a signal report. Each tested signal gets a verdict (notable / weak / none) and a
direction, plus evidence-backed content recommendations.

**Cost of a wrong call:** if the report says a signal matters when it does not, a team wastes hours
on edits with no payoff. If it misses a signal that matters, the team keeps ignoring it. Because
this is a report for people, not an automatic tool, the cost is time and trust.

**Why data helps:** popular rules sound right but are rarely checked. In this data, "old pages
decline" points the wrong way: pages 365+ days old declined at 42.6% against a 54.2% base rate
(starter CSV). Position has a hill-shaped pattern, and the effect of page age depends on traffic
level. A one-line rule cannot capture patterns that run backwards, bend, or interact.

## 2. Data safety

**Data used:**
- Starter CSV (`data/raw/content_refresh_anonymized.csv`, 30,000 pages, 32 clients) for Weeks 1–2.
- Warehouse release on Hugging Face (`FlyRank/internship-warehouse`, gated) from Week 3:
  `fact_content_daily_performance` (February and March 2026 partitions only), `dim_content`,
  `dim_clients`.

**Excluded on purpose, and why:**
- Outcome and its ingredients: `trend_direction`, `trend_pct`, the six `*_last_30d` / `*_prev_30d`
  columns (starter CSV), and March impressions (warehouse). Using them as clues copies the answer.
- Dates and flags that can describe the future relative to 1 March: `content_updated_date`,
  `last_optimized_date`, `is_deleted`, `is_published`.
- Product metadata: `provider_used`, `model_used`.
- GA4, sessions, and AI-referral columns for now: GA4 starts later for many clients and has gaps.
- Pseudonymous IDs (`client_hash_id`, `content_hash_id`): used only for joining, grouping, and
  client-holdout splits. Never as features, never decoded.

**Leakage checks done:**
- Starter CSV: confirmed `trend_pct` equals (last30 − prev30) / prev30 on 100% of pages with
  prev30 > 0, so every ingredient of the label is known and excluded.
- Warehouse: features use only 1–28 Feb; the outcome uses only March. A deliberate leak test added
  March impressions as a feature: AUC jumped from 0.594 to 0.994 on unseen clients, then the column
  was removed.
- **Known risk:** `word_count` and `search_volume` come from `dim_content`, a present-day snapshot.
  76.5% of pages in the frame were updated after 1 March, so `word_count` may not reflect the
  February page.

**Public safety:** notebooks print aggregate numbers only (counts, rates, medians, AUC). No client
names, domains, URLs, queries, raw rows, or tokens appear in `work/`. The HF token is read from
Colab Secrets only.

## 3. Baseline

**To do (Week 4, ML-07).** The transparent rule or score built first, why it is a fair comparison,
and its numbers on the same data and metric.

## 4. Model / analysis

**Method:** signal analysis. Each signal is compared against one outcome using a grouped
comparison, then checked with tier tables to catch shapes the comparison misses.

**Outcome (one sentence):** a page is declining when its March 2026 impressions per day are more
than 20% below its February 2026 impressions per day (pages with at least 100 February impressions).

**Metric:** decline-rate gap = decline rate in the top 25% of a signal minus the bottom 25%, in
percentage points. Thresholds set before looking: notable ≥ 5, weak 2–5, none < 2.

**Features (5):** `content_age_days` (days from creation to 1 March), `word_count`, `search_volume`,
`feb_avg_position` (February position, weighted by impressions), `feb_impressions`.

**Left out on purpose:** see Section 2.

**To do (Weeks 4–5):** run the metric on all five features in the warehouse frame; decide on
`word_count`; explain the final method choice.

## 5. Evaluation

**So far:** splits are grouped by client (whole clients go to train or test, never both), with
`random_state=42`. Base rates are reported next to every score: 54.2% declining (starter CSV),
25.1% (warehouse frame).

**To do (Week 6, ML-09):** client-by-client check of every notable signal; repeat on a second
month pair; error analysis.

## 6. Interpretation

**Findings so far (observed associations, not causes):**
- **Page age runs opposite to the popular belief.** Starter CSV: decline rate falls from 66.9%
  (31–90 days) to 42.6% (365+ days). Top-vs-bottom gap: −16.4 points.
- **Position is hill-shaped.** Top 3 (24.1%) and deep results (34.4%) declined least; middle tiers
  declined most (up to 61.0%). The end-to-end gap called it "weak" (−3.7), which shows why one
  number is not enough.
- **Signals interact.** Page age mattered about 25 points for low-traffic pages but about 4 points
  for excellent-traffic pages.
- **Word count changed direction with the method.** Medians looked flat; the gap said longer pages
  declined more (+10.3). Not yet trusted (snapshot problem, missing values by content type).
- **Search volume:** pages on higher-volume keywords declined less (−12.2).

**To do:** confirm these on the warehouse frame; feature importance; negative results.

## 7. Recommendation

**To do (Week 7, ML-10).** Ranked actions, how a FlyRank editor would use them, confidence, limits.

## 8. Reproducibility

- **Notebooks, in order:** `work/notebooks/w01_research_question.ipynb`,
  `w02_ml_task_framing.ipynb`, `w03_data_contract.ipynb` (all committed with outputs).
- **Run:** open each in Colab from the repo, add the Hugging Face READ token as the Colab Secret
  `HF_TOKEN` (Week 3+), then Runtime → Run all.
- **Data:** starter CSV in the repo; warehouse release build v20260703 on Hugging Face.
- **Seeds:** `random_state=42` for client-holdout splits and models.
- **Environment:** Google Colab, pandas, scikit-learn, DuckDB (installed in the notebook).
- **To do:** pin versions; commit any sealed-test cell and its metrics file if a sealed evaluation
  is claimed.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — https://flyrank.ai

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run · base rate next to every score.
