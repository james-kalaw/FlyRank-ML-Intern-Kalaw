# Capstone Report — Ranking Signal Analysis

- **Author:** James Kalaw
- **Lane:** Lane 1, Ranking Signal Analysis
- **Repo:** https://github.com/james-kalaw/FlyRank-ML-Intern-Kalaw
- **Date:** started 2026-11-20 (draft, updated after every assignment; last update 2026-11-23, after ML-08)

> Copied from `capstone_report_template.md`. Sections 1–8 follow the Pass / Needs-Work rubric.
> Sections 0 and 9 are paper sections. Lines marked **To do** are filled in later weeks.
> Details and exact numbers live in `work/PROJECT_LOG.md` (IDs like B7 or M3 point there).

## 0. Abstract

**To do (write last, Week 8).** Five sentences: question → data → method → headline result →
what the output is for.

## 1. Problem framing

**Decision:** whether a content or SEO team keeps acting on common beliefs (for example "refresh
old pages" or "write longer articles"), or moves its time to signals that actually go with pages
holding or losing search visibility; and which pages to review first.

**Who acts:** a content strategist or SEO lead deciding what to tell the team to review first.

**Unit of analysis:** one page. In the warehouse, each page gets features from February 2026 and
an outcome from March 2026, with 1 March 2026 as the decision day.

**Output:** a signal report. Each tested signal gets a verdict and a direction, plus evidence-backed
content recommendations. A ranked review queue (baseline rule, then model) supports the "review first"
decision.

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
- Warehouse release on Hugging Face (`FlyRank/internship-warehouse`, gated, build v20260703) from
  Week 3: `fact_content_daily_performance` (February and March 2026 partitions only), `dim_content`,
  `dim_clients`.

**Excluded on purpose, and why:**
- Outcome and its ingredients: `trend_direction`, `trend_pct`, the six `*_last_30d` / `*_prev_30d`
  columns (starter CSV), and March impressions (warehouse). Using them as clues copies the answer.
- Dates and flags that can describe the future relative to 1 March: `content_updated_date`,
  `last_optimized_date`, `is_deleted`, `is_published`.
- `word_count`: left out of the model (ML-08) because 76.5% of frame pages were updated after 1 March,
  so it may describe the page after the decision day.
- Product metadata: `provider_used`, `model_used`. No product flags or existing scores as features.
- GA4, sessions, and AI-referral columns: GA4 starts later for many clients and has gaps.
- Pseudonymous IDs (`client_hash_id`, `content_hash_id`): used only for joining, grouping, and
  client-holdout splits. Never as features, never decoded.

**Leakage checks done:**
- Starter CSV: confirmed `trend_pct` equals (last30 − prev30) / prev30 on 100% of pages with
  prev30 > 0, so every ingredient of the label is known and excluded.
- Warehouse: features use only 1–28 Feb; the outcome uses only March. A deliberate leak test added
  March impressions as a feature: AUC jumped from 0.594 to 0.994 on unseen clients, then the column
  was removed.
- Baseline (ML-07): reads only `content_type` and `feb_impressions`; an automatic check confirmed no
  March data, label, excluded column, or product flag.
- Model (ML-08): the peer CTR behind `ctr_vs_peers` is rebuilt from training clients only in every
  round, so test clients never shape it. No feature looked suspiciously strong (AUC 0.545, far from the
  0.994 leak test).

**Known risks (disclosed):**
- `content_type`, `search_volume`, and `word_count` come from `dim_content`, a present-day snapshot;
  `content_type` and `search_volume` are assumed stable since publishing.
- **Population choice:** 3,605 pages with no usable March rows were dropped because their label is
  unknown. This uses information from the outcome window.

**Public safety:** notebooks print aggregate numbers only (counts, rates, medians, AUC). Example pages
are shown without IDs, with clients labelled "client N" (rank by page count). No client names, domains,
URLs, queries, raw rows, or tokens appear in `work/`. The HF token is read from Colab Secrets only.

## 3. Baseline

**The rule (plain words):** review **feedly articles** first, starting with the ones that had the most
February impressions. Every other page gets score 0.

- Score: `is_feedly × feb_impressions` (no fitted weights). Reason code: `feedly_article`.
  Action label: `review_for_decline_risk`. Reads only `content_type` and `feb_impressions`.
- Notebook: `work/notebooks/w04_baseline_score.ipynb`. Metrics receipt: `work/outputs/baseline_metrics.json`.

**How it was chosen:** the first rule idea ("young pages on low-volume keywords") leaned on page age and
search volume (search volume is the signal behind FlyRank's quick-win flag). Both checks came back
**FALSE**: every bucket was within 2 points of the 25.1% base rate. Instead of building a rule that would
do no better than random, I checked four more clues with charts and picked the strongest using a rule
written before looking (the 500+ page bucket farthest from the base rate): content type, where feedly
articles declined at 53.3% (+28.2 points).

**Why it is a fair comparison:** same frame (65,756 pages, Feb 2026 features → Mar 2026 outcome), same
label, and the same metric (precision@K) as the model. In Week 5 it was recomputed in the model notebook's
own run and gave the same precision.

**Numbers (base rate 25.1%; random picking averaged over 20 seeded shuffles):**

| K | Rule | Random picking |
|---|---|---|
| 10 | 70.0% | 21.0% |
| 50 | 46.0% | 24.2% |
| 100 | 51.0% | 25.5% |
| 500 | 49.6% | 25.2% |

Over the whole list: AUC 0.486, average precision 0.258 (ML-08 run).

**Known weaknesses:** only 704 pages are feedly articles, so the rule cannot rank anything past them;
45% of those pages come from one client (6 of the top 10); sorting by February impressions did not help
(top 50: 46.0% vs 53.3% for all feedly pages). Six clues were tested, so part of the gap may be luck.
The feedly clue was picked by looking at all clients, so in the Week 5 comparison the baseline has a small
head start that the models do not.

## 4. Model / analysis

**Outcome (one sentence):** a page is declining when its March 2026 impressions per day are more
than 20% below its February 2026 impressions per day (pages with at least 100 February impressions).

**Part A: signal analysis (Weeks 2–4).** Each signal is compared against the outcome with grouped
comparisons, tier tables, and bar charts. Starter CSV: decline-rate gap (top 25% minus bottom 25%;
notable ≥ 5, weak 2–5, none < 2 points). Warehouse: bucket tables with verdicts set before looking
(buckets of 500+ pages; FALSE = every bucket within 2 points of the base rate). Signals checked on the
warehouse frame: page age, search volume, content type, February position, CTR vs position peers,
February impressions.

**Part B: learned model (Week 5, ML-08).** The question is "which first?", so each model outputs a chance
of declining and pages are ranked by it (scored with precision@K, like the baseline). "What drives it?" is
answered with permutation importance.

| Model | In plain words | Why |
|---|---|---|
| Logistic Regression | a weight per clue, added up, turned into a chance | simple and readable; the skill's starting point |
| Random Forest | many separate trees that vote | the skill's next step; stable |
| Gradient Boosting | small trees built one after another, each fixing the last | often strongest; catches bends and combinations |

Settings: LR with median imputation and standardization; RF with 200 trees and `min_samples_leaf=50`;
`HistGradientBoostingClassifier` with `learning_rate=0.05`, `max_iter=300`. Seed 42.

**Features (8, all known on 1 March):** `is_feedly` (from `content_type`), `ctr_vs_peers` (February CTR
÷ the pooled CTR of training pages in the same position group), `feb_position` (impression-weighted
February position, 0 treated as missing), `position_is_zero` (flag for the "position 0" quirk),
`log_feb_impressions`, `content_age_days`, `log_search_volume`, `volume_is_missing`.

**Left out on purpose:** `word_count` (snapshot problem), all March data, the label, IDs, snapshot dates and
flags, GA4. See Section 2.

**Selection rule (written before looking):** pick the simplest model (LR → RF → GB) whose precision@100
and precision@500 are both within 2 points of the best. Complexity is chosen only if it clearly wins.

## 5. Evaluation

**Split:** `GroupKFold(5)` by client. The 26 clients are split into 5 groups; each round trains on 4 and
tests on the 5th, so every page gets a score from models that never saw its client. Why: pages from one
client share habits, and one client holds 45% of the feedly pages, so a random split would let a model
memorize clients. Time order holds inside every page (features from February, label from March).

**Results (65,756 pages, base rate 25.1%; precision in %):**

| Ranking | P@10 | P@50 | P@100 | P@500 | AUC | Avg precision |
|---|---|---|---|---|---|---|
| Random picking (20 shuffles) | 22.5 | 24.2 | 25.4 | 24.2 | 0.500 | 0.251 |
| Baseline rule (feedly first) | 70.0 | 46.0 | 51.0 | 49.6 | 0.486 | 0.258 |
| **Logistic Regression (chosen)** | **80.0** | **50.0** | **54.0** | **51.0** | 0.545 | 0.281 |
| Random Forest | 70.0 | 46.0 | 47.0 | 42.2 | 0.567 | 0.304 |
| Gradient Boosting | 50.0 | 48.0 | 35.0 | 30.2 | 0.554 | 0.291 |

- The pre-set rule picked **Logistic Regression**: it was the best model at both @100 and @500. It beat the
  baseline by **3.0 points @100 and 1.4 @500**. That is small: for 100 pages, one standard error is about
  5 points, so chance could explain it.
- **Not steady:** inside each round (precision@50), LR beat the baseline in 4 of 5 rounds but lost badly in
  round 1 (20% vs 44%), a single client with no feedly pages. Gradient Boosting swung from 68% to 14%.
- Random Forest ranked the whole list best (AUC 0.567) but was weaker at the top, which is the part an
  editor reads.

**Errors (short analysis):**
- LR's top 500 are all feedly pages, from 7 clients, and one client holds 57.8% of them. The model mostly
  re-orders the baseline's pages.
- By content type the predictions are close to reality (feedly 53.5% predicted vs 53.3% real).
- Biggest misses: "position 0" pages (predicted 26.4% vs real 42.2%) and rank 51+ (8.5% vs 21.7%), where
  decline rises again after rank 50 and a straight-line model cannot bend.
- Concrete cases: the model was about 70% sure for three small, zero-click feedly pages that did not decline,
  and scored near 0% for pages with extremely high clicks compared with their peers (27–62×) that did decline.

**Limits of this evaluation:**
- It tests **new clients, not new months**: every round uses the same Feb → Mar window.
- The 5 rounds are very uneven (two rounds are one large client each, with no feedly pages).
- The baseline's clue was chosen using all clients (a small head start for the baseline).
- One ranking pools the scores of 5 round models; the per-round table is the check on that.

**To do (Week 6, ML-09):** report the random-split ("before") vs grouped ("after") numbers; repeat on a later
mid-panel month pair (time-aware check; never June 2026); client-by-client check of the feedly result;
leakage audit on the final 8 features.

## 6. Interpretation

**From the model (Week 5, unseen clients), observed associations, not causes:**
- **The three clues the model leaned on most:** February position, feedly article, and clicks compared with
  pages at the same rank (permutation importance 0.019, 0.016, 0.011). Their directions match the Week 4
  tables: higher-ranked pages, feedly articles, and pages with fewer clicks than their peers went with more
  decline.
- **Negative result: page age.** Shuffling page age made the model slightly better on new clients
  (importance −0.013). Together with the FALSE verdict in Week 4, this data shows no usable link between
  page age and a Feb → Mar decline.
- February impressions and search volume added almost nothing.
- The learned model added only a little over a one-line rule, and the simplest model was the best at the top
  of the list.

**Warehouse bucket tables, Feb → Mar 2026 (features before outcome):**
- **Page age and search volume showed no relationship with decline** (every bucket within 2 points of
  the 25.1% base rate), unlike the starter CSV below. A pattern that looks strong when signal and
  outcome are measured in the same window can disappear when the signal comes first.
- **Content type was the strongest clue.** Feedly articles declined at 53.3% (704 pages) vs 24.8% for
  keyword articles. All feedly pages lack search volume, which explains most of the "missing volume"
  bucket (42.9%).
- **Higher-ranked pages declined more often.** From rank 1 to 50, decline fell from 28.1% (top 3) to
  13.7% (ranks 21–50), the opposite of "low-ranked pages are the ones at risk."
- **Pages with fewer clicks than pages at the same rank declined more** (28.8% vs 18.8–20.6%),
  which supports the idea behind FlyRank's CTR-fix flag, moderately. Inside the feedly group, pages
  that declined had about half the CTR-vs-peers of those that didn't (0.30 vs 0.60).
- **February impressions were a weak clue** (every bucket within 2.4 points).

**Starter CSV (same 90-day window for signal and outcome), earlier findings:**
- **Page age runs opposite to the popular belief.** Decline rate falls from 66.9% (31–90 days) to 42.6%
  (365+ days). Top-vs-bottom gap: −16.4 points.
- **Position is hill-shaped.** Top 3 (24.1%) and deep results (34.4%) declined least; middle tiers
  declined most (up to 61.0%). The end-to-end gap called it "weak" (−3.7), which shows why one
  number is not enough.
- **Signals interact.** Page age mattered about 25 points for low-traffic pages but about 4 points
  for excellent-traffic pages.
- **Word count changed direction with the method.** Medians looked flat; the gap said longer pages
  declined more (+10.3). Not trusted (snapshot problem), and not used in the model.
- **Search volume:** pages on higher-volume keywords declined less (−12.2).

**To do:** explain why the starter-CSV patterns did not hold in Feb → Mar; check the feedly result client by
client and on another month pair; investigate the position-0 quirk and the rise after rank 50.

## 7. Recommendation

**To do (Week 7, ML-10).** Ranked actions, how a FlyRank editor would use them, confidence, limits.

## 8. Reproducibility

**From a fresh clone:**
1. `git clone https://github.com/james-kalaw/FlyRank-ML-Intern-Kalaw.git` (or open the notebooks
   straight from GitHub in Colab using the badges in `work/README.md`).
2. Request access to `FlyRank/internship-warehouse` on Hugging Face (instant) and create a READ token.
3. In Colab, add the token as the Secret `HF_TOKEN` with notebook access on (Week 3+). Never paste it in a cell.
4. Open each notebook in order and use Runtime → Run all. If the setup cell reports an old DuckDB:
   Runtime → Restart session, then Run all again.

**Notebooks, in order (all committed with outputs):** `work/notebooks/w01_research_question.ipynb`,
`w02_ml_task_framing.ipynb`, `w03_data_contract.ipynb`, `w04_baseline_score.ipynb`, `w05_model.ipynb`.

**Outputs:** `w04_baseline_score.ipynb` writes `work/outputs/baseline_action_score.csv` (git-ignored,
regenerated on every run) and `work/outputs/baseline_metrics.json` (committed). `w05_model.ipynb` writes
`work/outputs/model_metrics.json` (committed).

**Numbers a re-run should reproduce:** frame 65,756 pages from 26 clients, 3,605 dropped, base rate 25.1%;
baseline precision@10/50/100/500 = 70.0 / 46.0 / 51.0 / 49.6; Logistic Regression precision@100 / @500 =
54.0 / 51.0, AUC 0.545. Full expected values per notebook: `work/PROJECT_LOG.md`, Section 11.

**Data:** starter CSV in the repo; warehouse release build v20260703 on Hugging Face. Weeks 3–5 read only the
`month=2026-02` and `month=2026-03` partitions. June 2026 (the last month) is kept as a sealed test month
and has not been used.

**Seeds:** 42 everywhere (client-holdout splits, models, random-picking shuffles).

**Environment:** Google Colab; pandas, numpy, scikit-learn, matplotlib, DuckDB (installed in the notebook).
The Week 5 run used **DuckDB 2.0.1** and **scikit-learn 1.6.1**. `requirements.txt` only sets minimum versions
(`scikit-learn>=1.4`, `duckdb>=1.0`); tree-model results can shift a little between scikit-learn versions.

**Sealed evaluation:** none claimed so far. If one is claimed later, its frame-building cell and metrics
file will be committed.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — https://flyrank.ai

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run · base rate next to every score.
