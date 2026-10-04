# vc-matcher

A Claude Code plugin from Backed VC that matches **startups with VCs** (and
VCs with startups) with high accuracy, depth and conviction. Eight agents
research using **web search only**, audit each other's facts, and argue it out
until they agree. Only matches that clear a shared rubric are reported.

## Install

```text
/plugin marketplace add amaliablank/matcher
/plugin install vc-matcher@backed-plugins
```

## Use

```text
/vc-matcher:match-vcs <startup> [context]
/vc-matcher:match-startups <vc> [context]
```

Examples:

```text
/vc-matcher:match-vcs "Acme Robotics" acmerobotics.com, raising EUR 4M seed in Q1 2027
/vc-matcher:match-startups "Amadeus Capital Partners" focus on the deep-tech seed fund
```

Adding a domain or other context helps the Deduplicator pin down the right
entity. If the name is ambiguous, the run pauses and asks you which one you
meant.

**Output:** a short summary in chat, plus a full report at
`./matcher-reports/<mode>-<slug>-<date>.md`. All working files (dossiers,
audits, deliberation logs, identity cards) stay in
`./matcher-reports/.work/<run-id>/`.

## How it works

```
/match-vcs or /match-startups (main session: sets up the run, asks you when needed)
  └── coordinator: plans the run, picks agents, models and budgets; relays deliberation; writes the report
        ├── startup-profiler ─┐
        ├── vc-profiler       │
        ├── vc-scout          ├── each calls ──► deduplicator (identity checks)
        ├── startup-scout     │
        ├── match-scorer      │
        └── devils-advocate  ─┘
```

**/match-vcs pipeline:** profile the startup → Devils-Advocate audit →
VC-Scout writes an ideal-VC spec, builds a long list, screens it against the
hard gates and shortlists ≤ 3 → audit → VC-Profiler on each shortlisted VC (in
parallel) → audit → Match-Scorer scores blind, then deliberates with the scout
→ final audit → report. **/match-startups** runs the mirror image.

| Agent | Role |
|-------|------|
| `coordinator` | Delegator. Interprets the request, chooses agents, order, **models** and search budgets, relays deliberation, assembles the report. Never researches |
| `startup-profiler` | Startup dossier: identity, headcount, founders (incl. demographics), stage, rounds, existing investors, traction, geography, growth plans, competitors |
| `vc-profiler` | VC dossier: funds and deployment, stated vs **revealed** stage / geography / vertical focus, ticket size, lead behaviour, 24-month deal log, team |
| `vc-scout` | Ideal-VC spec → long list → gate screen → ≤ 3 shortlisted VCs |
| `startup-scout` | Ideal-startup spec → long list → gate screen → ≤ 3 shortlisted startups |
| `match-scorer` | Scores each pair blind, critiques the scout, ≤ 3 deliberation rounds, assigns conviction |
| `devils-advocate` | Audits every fact against its sources and a catalogue of agent failure modes (entity drift, stale facts, citation laundering, number and role drift, sycophantic concessions…); ≤ 3 rounds with each author |
| `deduplicator` | Identity sleuth. Separates *Amadeus Capital Partners* from Mozart, Amadeus IT Group and every other namesake |

### Shared references

- [`references/matching-rubric.md`](plugins/vc-matcher/references/matching-rubric.md): source tiers, confidence labels, 24-month recency window, six hard gates, eight weighted dimensions, conviction levels.
- [`references/protocol.md`](plugins/vc-matcher/references/protocol.md): topology, progressive disclosure (L0 task card → L1 summary → L2 dossier → L3 evidence), identity cards, deliberation formats, drift catalogue, model-selection policy.

### Hard gates (any failure disqualifies a match)

G1 existing investor / already in portfolio · G2 thesis or vertical mismatch ·
G3 competing portfolio company · G4 stage mismatch · G5 inactive fund (no new
deal in 18 months, or not deploying) · G6 geography mismatch.

### Conviction policy

Only **HIGH** matches are recommended: score ≥ 75/100, all gates passed on
verified evidence, no open Devils-Advocate objections, and scorer and scout in
agreement. If fewer than three qualify, you get fewer, with near-misses and
exclusions listed and the reasons given.

## Notes

- **Web search only:** agents use `WebSearch` and `WebFetch` and no other data
  sources. Coverage is limited to what is public. Gaps are reported, never
  guessed.
- **Founder demographics** are recorded on request, including **inferred**
  attributes. Inferred values are always labelled with their method, never
  inferred from photos, weighted lightly (rubric D8), and never used to rule
  a match out.
- **Models:** agents default to `inherit`. The coordinator picks a model per
  invocation (protocol §7) and records its choices in the run's `00-plan.md`.
- **Nested agents:** the profilers, scouts, scorer and Devils-Advocate call the
  Deduplicator directly, which needs Claude Code's subagent spawn depth ≥ 3
  (the default). If a lower limit is set, the agents fall back to sending
  identity-check requests through the coordinator.
- Runs are thorough and therefore long and token-heavy.

## Layout

```
.claude-plugin/marketplace.json
plugins/vc-matcher/
  .claude-plugin/plugin.json
  agents/        8 agents
  skills/        match-vcs, match-startups
  references/    matching-rubric.md, protocol.md
```

## License

MIT
