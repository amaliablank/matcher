# VC ↔ Startup Matching Rubric

The single source of truth for what counts as evidence, what disqualifies a match,
how a match is scored, and what "high conviction" means. Every agent in
`vc-matcher` applies this rubric. No agent may relax, reinterpret or skip any part
of it. Doing so is **criteria drift**, and the Devils-Advocate must flag it.

---

## 1. Evidence standards

### 1.1 Source tiers

| Tier | What it is | Examples |
|------|------------|----------|
| **T1, Primary** | The entity itself, or an official register | Company / fund website, fund's own portfolio page, founder's own LinkedIn or blog, press release issued by the company or fund, Companies House / SEC Form D / Handelsregister / Registre du commerce / other official registries, regulatory filings |
| **T2, Reputable secondary** | Independent reporting with editorial standards, or curated databases | FT, Bloomberg, Reuters, WSJ, TechCrunch, Sifted, The Information, EU-Startups, Tech.eu, Axios Pro, public Crunchbase / Dealroom / PitchBook / Tracxn pages, partner interviews on reputable podcasts |
| **T3, Weak** | Anything that may be copied, auto-generated or unaccountable | SEO listicles, "top 50 VCs in X" lists, AI-generated directories, scraped aggregators, forum posts, anonymous blogs |

**Connector data** (MCP connectors such as Crunchbase, PitchBook or Dealroom,
when the user has them) is tiered by `connectors.md` §4. Venture, people and
traffic databases are T2. Official registries are T1. A connector record and
the same provider's public page are one source.

Aggregators that copy each other do **not** count as independent sources. If two
T2/T3 pages share identical wording or obviously cite the same origin, count them
as one source (avoid **citation laundering**).

### 1.2 Confidence labels (attach one to every fact)

| Label | Requirement |
|-------|-------------|
| **VERIFIED** | One unambiguous T1 source, **or** two independent sources of which at least one is T1/T2, and no credible contradicting source |
| **CORROBORATED** | Two independent T2 sources agree; no T1 available; no credible contradiction |
| **SINGLE-SOURCE** | Exactly one T2 source, or T3 sources only |
| **INFERRED** | Derived by reasoning from other facts or from indirect signals (e.g. headcount from LinkedIn employee count, demographics inferred from public signals). The inference method must be stated |
| **UNVERIFIED** | Claimed, but the source does not actually say it, could not be opened, or is contradicted. Not usable for anything |

Only **VERIFIED** and **CORROBORATED** facts may decide a hard gate (§2).
SINGLE-SOURCE and INFERRED facts may inform a score, with the dimension score
capped as described in §3.3. UNVERIFIED facts are discarded.

### 1.3 Recency

- **Run date** = the date the command is executed. Every agent states it.
- **Current window = 24 months before the run date.** Facts about *current
  activity* must come from inside this window: an active thesis, current fund,
  recent investments, latest round, current headcount, current team.
- Older evidence may be cited as **historical context** only and must be labelled
  `(historical, YYYY-MM)`.
- Undated sources get the date of the most recent event they mention, never the
  date they were accessed.

### 1.4 Required fact format

Every fact recorded in a dossier or ledger uses this shape:

```
[F-<agent-prefix>-<nn>] <claim>
  value:      <normalised value; currency + year for money, e.g. "EUR 2.0M (2025-03)">
  source:     <URL or connector:<server>/<tool> record <id>>  (<T1|T2|T3>, published/updated YYYY-MM-DD or "undated → YYYY-MM")
  source 2:   <URL>  (optional, for VERIFIED / CORROBORATED)
  confidence: <VERIFIED|CORROBORATED|SINGLE-SOURCE|INFERRED|UNVERIFIED>
  method:     <required for INFERRED: how it was inferred>
  entity:     <ENT-id from the Deduplicator identity card>
```

Money is always normalised to the original currency **plus** a EUR or USD
equivalent, with the year of the figure. Never mix up **fund size** and
**ticket size**, or **round size** and **this investor's cheque**.

---

## 2. Hard gates (instant disqualification)

Any gate that **FAILs** removes the pair from consideration permanently. It will
not be reported, except in the "excluded" appendix of the report.
Any gate that is **UNKNOWN** blocks HIGH conviction. The scout must resolve it
or drop the candidate.

| # | Gate | FAIL when… |
|---|------|------------|
| **G1** | Existing relationship | The VC is already an investor in the startup (any round, including via an affiliated fund), or the startup is already in the VC's portfolio. Also FAIL if the VC is publicly known to have passed on or exited this startup |
| **G2** | Thesis / vertical mismatch | The startup's sector, business model or technology is outside the VC's stated thesis **and** outside the verticals of its last 24 months of investments. Generalist funds pass only if they have made ≥1 investment in an adjacent vertical within 24 months |
| **G3** | Competing portfolio company | The VC holds a current portfolio company that sells a substitutable product to the same customer segment as the startup. Adjacent or complementary companies are **not** competitors (they score under D6) |
| **G4** | Stage mismatch | The round the startup is raising, or will plausibly raise next, is outside the stages at which the VC has *actually* written initial cheques in the last 24 months, whatever its marketing says. One-off exceptions don't widen the range |
| **G5** | Inactive fund | The VC has no VERIFIED/CORROBORATED new investment (initial or lead follow-on) in the **last 18 months**, **or** there is credible evidence that it is not deploying from a current fund (fund fully deployed, wind-down, team departed, fundraising stalled) |
| **G6** | Geography mismatch | The startup's HQ or primary operating market is outside every region the VC states it covers **and** outside every region it has invested in during the last 24 months |

Gate results are recorded as:

```
G<n>: PASS | FAIL | UNKNOWN — <one-line reason> — evidence: [F-..], [F-..]
```

---

## 3. Scored dimensions (100 points)

Only pairs that pass all six gates are scored.

### 3.1 Dimensions and weights

| # | Dimension | Weight | What is measured |
|---|-----------|-------:|------------------|
| **D1** | Thesis & vertical fit | 25 | How centrally the startup sits in the VC's thesis and recent deal flow (core vs adjacent) |
| **D2** | Stage & round fit | 15 | How well the startup's round (stage, timing, traction level) matches the VC's typical entry point |
| **D3** | Ticket size fit | 15 | Whether the VC's typical initial cheque fits the startup's raise (as lead: ≈ 50–100% of round; as participant: a meaningful slice) |
| **D4** | Geography fit | 10 | Core region vs secondary region vs "will invest anywhere" |
| **D5** | Lead / role fit | 10 | Whether the VC's lead / co-lead / follow tendency matches what the startup needs in this round |
| **D6** | Portfolio adjacency | 10 | Non-competing portfolio companies that offer customers, distribution, partnerships or pattern knowledge |
| **D7** | Momentum & timing | 10 | Fund vintage and deployment pace on the VC side; traction and growth signals vs the VC's usual bar on the startup side |
| **D8** | Founder-profile fit | 5 | The VC's demonstrated track record with similar founders (e.g. repeat vs first-time, technical vs commercial, specific mandates such as female-founder or diaspora programmes) |

### 3.2 Scoring anchors (each dimension 0–4, then × weight / 4)

| Score | Meaning |
|------:|---------|
| **4** | Bullseye. Multiple VERIFIED facts show a direct, specific fit (e.g. three or more near-identical recent deals) |
| **3** | Strong. Clear fit with VERIFIED/CORROBORATED evidence; minor caveats |
| **2** | Plausible. Fit is reasonable but evidence is thin, generic or partly INFERRED |
| **1** | Weak. Edge case; fit only under a charitable reading |
| **0** | No fit, or no evidence |

### 3.3 Evidence caps

- A dimension whose supporting evidence is only **SINGLE-SOURCE** is capped at **2**.
- A dimension whose supporting evidence is only **INFERRED** is capped at **2**,
  except D8, which is capped at **1** when it rests on inferred demographics.
- A dimension with no evidence scores **0**, never a "neutral" midpoint.

### 3.4 Total

```
Total = Σ (dimension_score / 4 × weight)          → 0–100
```

---

## 4. Conviction levels

| Level | All of the following must hold |
|-------|-------------------------------|
| **HIGH** (reported) | Total ≥ **75**; all six gates PASS on VERIFIED/CORROBORATED evidence; D1 ≥ 3; no open Devils-Advocate objection; the Match-Scorer's and the scout's totals differ by ≤ 10 after deliberation; the Deduplicator has confirmed identity for both entities |
| **MEDIUM** (near-miss) | Total 60–74, **or** HIGH except for one unresolved soft issue. Listed under "Did not make it", with the reason |
| **LOW** (dropped) | Total < 60, or any gate UNKNOWN after deliberation |
| **EXCLUDED** | Any gate FAIL |

**Only HIGH matches are presented as recommendations.** If fewer than three
candidates reach HIGH, the report shows fewer and says plainly how many fell
short and why. Never pad the list.

---

## 5. Deliberation limits

- Devils-Advocate ↔ originating agent: **max 3 rounds** per disputed fact.
- Match-Scorer ↔ scout: **max 3 rounds** per match.
- Anything still disputed after round 3 is resolved **against** the claim. The
  fact becomes UNVERIFIED, or the match is downgraded one conviction level.
- An agent may only concede to new **evidence** or a demonstrated reasoning
  error, never to persistence. Giving in without evidence is **sycophantic
  collapse** and is itself a drift failure (see `protocol.md` §6).
