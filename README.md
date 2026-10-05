# vc-matcher

A Claude Code plugin from Backed VC that matches **startups with VCs** (and
VCs with startups) with high accuracy, depth and conviction. Eight agents
research on the **web plus any venture-data connectors you have** (Crunchbase,
PitchBook, Dealroom, Harmonic, registries, your CRM…), audit each other's
facts, and argue it out until they agree. Only matches that clear a shared rubric are reported.

## Install

```text
/plugin marketplace add amaliablank/matcher
/plugin install vc-matcher@backed-plugins
```

### No `/plugin` command? (claude.ai/code cloud sessions)

This repo also carries the agents and commands in `.claude/` (links into
`plugins/vc-matcher/`). Any Claude Code session opened **on this repo**
loads them automatically, with no install. There the commands are simply
`/match-vcs` and `/match-startups`. Reports are written inside the session's
container, so ask Claude to commit them if you want to keep them.

To use them in another repo without the plugin system, copy the real files
(not the links): `plugins/vc-matcher/agents/` → `.claude/agents/`,
`plugins/vc-matcher/skills/` → `.claude/skills/`, and
`plugins/vc-matcher/references/` → `.claude/vc-matcher/references/`.

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
/match-vcs or /match-startups (main session: sets up the run, finds your connectors, asks you when needed)
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
| `devils-advocate` | Audits every fact against its sources and a catalogue of agent failure modes (entity drift, stale facts, citation laundering, database over-trust, number and role drift, sycophantic concessions…); ≤ 3 rounds with each author |
| `deduplicator` | Identity sleuth. Separates *Amadeus Capital Partners* from Mozart, Amadeus IT Group and every other namesake |

### Shared references

- [`references/matching-rubric.md`](plugins/vc-matcher/references/matching-rubric.md): source tiers, confidence labels, 24-month recency window, six hard gates, eight weighted dimensions, conviction levels.
- [`references/connectors.md`](plugins/vc-matcher/references/connectors.md): how connectors are discovered, classified (venture databases, people data, registries, traffic signals, private workspace), assigned to agents and treated as evidence.
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

- **Data sources:** agents always research the web (`WebSearch`, `WebFetch`).
  At the start of each run, the command checks which MCP connectors you have.
  It probes each venture-data connector (Crunchbase, PitchBook, Dealroom, CB
  Insights, Tracxn, Harmonic, Specter, people-data providers, company
  registries, traffic data) with one read call, and hands the working ones to
  the agents that need them. Connector data is cited as `connector:<server>/…`
  and treated as a curated database (T2; registries T1). Gate-deciding facts
  still need an independent web source, and the Devils-Advocate checks this.
  With no connectors, the run is web-only and the report says so.
- **Private connectors** (CRM such as Affinity, Attio, HubSpot or Salesforce;
  Notion, Drive, Gmail, Slack) are used **only if you approve them** when
  asked at the start of the run. They help with gate G1 (existing
  relationship) and warm-intro paths. Facts from them are tagged `[private]`
  in the report.
- **Read-only:** agents may only call read tools. Write-type connector tools
  (send, create, update, delete, post, share, …) are blocked in the agents'
  configuration and forbidden in their instructions.
- **Permissions:** to avoid repeated permission prompts (and denials for
  background agents), pre-approve your data connectors' read tools in
  `/permissions`, e.g. `mcp__crunchbase`.
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
  references/    matching-rubric.md, protocol.md, connectors.md
```

## License

MIT
