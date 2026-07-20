# Project Kickoff: NBA Flopping Crackdown — Natural Experiment Analysis

## Context

I'm building a portfolio data science project to demonstrate applied statistics skills for job applications (target roles: data scientist / analytics engineer, ideally ones touching experimentation). The project analyzes whether the NBA's crackdown on "flopping" produced a measurable change in game officiating statistics. This is **not** a true A/B test (no random assignment) — it's a **quasi-experimental / interrupted time series design**, and I want the project itself to demonstrate that I understand that distinction, not obscure it.

Work with me interactively. Checkpoint with me at the end of each phase below before moving to the next — don't build all five phases unattended. If you hit a decision that changes the scope, methodology, or timeline, ask rather than assuming.

---

## Research question

Did the NBA's in-game flopping penalty (introduced for the 2023-24 season, allowing referees to call a live flopping technical foul during a game, rather than only fining players after video review post-game) produce a detectable change in game officiating behavior — specifically foul rates, technical foul rates, and free throw rates?

**Known complication, verify in Phase 0:** the league appears to have *escalated* flopping enforcement again for the 2025-26 season ("any referee may call a flopping violation immediately," per NBA officiating memos). This means there may be two candidate treatment dates, not one. Do not assume — confirm both against primary sources before locking in a design.

---

## Phase 0 — Verify facts before writing any code (do this first)

1. Pull the exact effective dates and rule text for:
   - The 2023-24 season change that introduced an in-game live flopping technical foul (previously flopping was only punished via post-game video review and fines).
   - The 2025-26 season expansion of flopping enforcement.
   - Source these from `official.nba.com` Points of Emphasis pages / league memos, not secondary sports-media summaries, where possible.
2. **Critical data feasibility check**: determine whether NBA play-by-play data actually tags flopping technical fouls distinctly from other technical fouls (e.g., in the `PLAYBYPLAYV2`/`PLAYBYPLAYV3` endpoint description text via `nba_api`). Pull a small sample of games from after the rule change and inspect real technical foul description strings.
   - If flopping technicals ARE distinctly tagged: great, that's your primary outcome metric.
   - If they are NOT distinguishable from other technicals in the box score/PBP data: tell me. We'll need to fall back to proxy metrics (total technical fouls/game, personal fouls/game, FTA/game, whistle rate per possession) and I want to know this up front, not discover it three phases in.
3. Report back: confirmed date(s), what's directly measurable vs. what needs a proxy, and your recommended primary treatment date. Wait for my sign-off before Phase 1.

---

## Phase 1 — Data pipeline

**Source:** `nba_api` (github.com/swar/nba_api) — free, no auth, actively maintained. Be aware it has real rate limits; build in request throttling/backoff and local caching from the start so re-runs don't re-hit the API.

Build:
- Ingestion scripts pulling team-game-level box score stats (personal fouls, technical fouls, FTA, FTM, pace) and, if Phase 0 confirms feasibility, technical-foul-level play-by-play detail, for a window spanning at least 2 full seasons before and 2 full seasons after the confirmed treatment date(s).
- Land raw pulls as Parquet files (don't hit the API repeatedly during development — cache aggressively).
- A lightweight local database (SQLite is fine for this scale — no need for Postgres) with:
  - `games` / `team_game_stats` table
  - A manually-curated `rule_changes` table: rule name, target behavior, effective date, source URL, confidence level in that date
- An ETL step that tags every game as pre/post relative to the treatment date(s), and computes days-since-change (needed later for the sequential monitoring simulation).

## Phase 2 — Statistical methodology

Build this as a standalone analysis module before wiring it to any API/frontend, so the stats are correct in isolation.

1. **Baseline test**: Welch's t-test comparing pre vs. post for each outcome metric. Report effect size (Cohen's d) alongside p-values — never one without the other.
2. **Count-data model**: outcome metrics here are counts (fouls/technicals per game), so also fit a Poisson or negative binomial GLM (check for overdispersion to decide which) with season and team fixed effects, treatment indicator as the variable of interest, reporting an incidence rate ratio.
3. **Difference-in-differences control**: pre/post alone can't separate "the rule did this" from "the league was already trending this way" (e.g., 3PT rate and pace have both been rising for a decade, which affects foul opportunities). Construct a control series — a foul/technical category that plausibly wasn't targeted by the flopping rule (e.g., loose-ball fouls, rebounding fouls) — and run the DiD: (post-pre change in targeted metric) − (post-pre change in control metric). If Phase 0 finds no good in-league control metric, consider a cross-league control (WNBA or G-League, which weren't necessarily subject to the same points of emphasis on the same timeline) as a stretch option — flag this to me as a design choice rather than deciding unilaterally.
4. **Power analysis**: given the historical variance in these metrics, compute the minimum detectable effect at 80% power for the number of post-change games actually available. Also compute how many games *would* be needed to detect a smaller effect (e.g., 5%), so the write-up can honestly state what the design can and can't detect.
5. **Multiple comparison correction**: you'll be testing several outcome metrics (technicals, PF, FTA, pace-adjusted whistle rate) — apply Benjamini-Hochberg FDR correction across that family, and be ready to explain in the README why FDR rather than Bonferroni was the right choice here.
6. **Sequential monitoring simulation**: simulate watching the target metric accumulate week-by-week after the treatment date. Run a naive test after every week and show how repeated peeking inflates the false-positive rate. Then apply a real correction (O'Brien-Fleming or Pocock alpha-spending boundary) and show how/when the "stop and declare significance" decision would change. This is meant to demonstrate genuine understanding of early stopping, not just define it.

Output of this phase: a results table (metric, raw p, corrected p, effect size, CI, power) that Phase 3 will serve.

## Phase 3 — Backend

- Small **FastAPI** service exposing the Phase 2 results (and ideally able to re-run the analysis on demand for a given metric/date range) — this is what makes the project "full-stack" rather than a notebook, and it's a reasonable thing to walk through in an interview.
- Persist computed results so the frontend isn't recomputing GLMs on every page load.

## Phase 4 — Frontend

Default to a **React frontend calling the FastAPI backend** for the strongest resume signal. If time is tight, propose Streamlit as a faster MVP fallback and let me decide — don't silently pick the smaller option.

Views needed:
- Pre/post distribution plot for the primary metric, with the DiD control series overlaid
- Summary table across all metrics: raw p-value, FDR-corrected p-value, effect size, CI
- The sequential monitoring chart: naive cumulative p-value vs. the alpha-spending boundary over time since the rule change

## Phase 5 — Documentation & polish

- README that states the research question, the design (and explicitly why it's a quasi-experiment, not an A/B test), data sources, and a plain-English summary of findings including honest limitations (confounds you couldn't fully rule out, what the power analysis says about what you could/couldn't detect).
- Deploy the dashboard somewhere with a live link (Streamlit Community Cloud, or Render/Vercel for the FastAPI+React version) — a working demo link matters more than the code for resume purposes.

---

## Working agreement

- Checkpoint with me after Phase 0 and after Phase 2 at minimum — those are the two phases where a wrong assumption (bad date, unmeasurable outcome, weak control series) would waste the most downstream work if uncaught.
- If `nba_api` data doesn't support something I assumed above, tell me and propose an alternative rather than quietly substituting one.
- Keep the stats code and the web app code cleanly separated (the stats module should be runnable/testable with no web framework involved) — I want to be able to point to the analysis code by itself in an interview.
- Use a git repo from the start with reasonably granular commits per phase.

Start with Phase 0.
