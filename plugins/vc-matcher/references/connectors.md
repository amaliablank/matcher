# Connector Policy

vc-matcher always researches on the open web (`WebSearch`, `WebFetch`).
If the user has **MCP connectors** for venture data (Crunchbase, PitchBook,
Dealroom and the like), the agents use them **as well**, never instead of the
web. A connector adds depth and structured data. It does not lower the
evidence bar in `matching-rubric.md`.

---

## 1. Discovery (main session, once per run)

The main session builds a **connector inventory** before handing off to the
Coordinator and writes it to `<run dir>/02-connectors.md`.

1. **List candidate tools.** Look at every tool available in the session
   whose name starts with `mcp__`. If tools are deferred behind `ToolSearch`,
   also run keyword searches for each family in §2, for example:
   `crunchbase`, `pitchbook`, `dealroom`, `harmonic`, `tracxn`, `cb insights`,
   `specter`, `affinity`, `attio`, `linkedin`, `companies house`, `opencorporates`,
   `sec edgar`, `funding round`, `investor`, `company search`.
2. **Classify** each server into a family in §2 by its server name *and* its
   tool descriptions, since names vary, e.g. `mcp__crunchbase__…`,
   `mcp__claude_ai_Crunchbase__…`, `mcp__cb__…`. Ignore servers that fit no
   family (e.g. calendar or audio tools).
3. **Probe** each data server with one cheap read call, such as an
   organisation search for a well-known company. Record `OK`, `AUTH ERROR`,
   `RATE LIMITED` or `NO ACCESS` (a plan or tier restriction). Only servers
   marked `OK` go into the inventory as usable.
4. **Private connectors need consent.** If any family-P server (§2) is usable,
   ask the user once (AskUserQuestion, multi-select) which of them this run
   may read. Market-data families are used without asking.
5. Write the inventory:

```
# Connector inventory: run <run-id>
| Server | Family | Status | Read tools to use | Notes (plan limits, coverage) |
|--------|--------|--------|-------------------|-------------------------------|
| crunchbase | M1 | OK | mcp__crunchbase__search_organizations, …get_organization | Basic tier: no funding-round detail |
Private connectors approved by user: <list or "none">
```

If nothing is found, write `No data connectors detected: web-only run.`
The run continues as web-only and the report says so.

---

## 2. Connector families

| Family | Examples | Typical use |
|--------|----------|-------------|
| **M1, Venture databases** | Crunchbase, PitchBook, Dealroom, CB Insights, Tracxn, Harmonic, Specter | Funding rounds, investors per round, VC portfolios and deal logs, fund details, headcount history, competitors / similar companies, long-list sourcing |
| **M2, People & company graph** | LinkedIn or people-data providers, Clay, Apollo, People Data Labs | Founder and partner careers, current roles, headcount trends, hiring velocity |
| **M3, Registries & filings** | Companies House, OpenCorporates, SEC EDGAR, national registries | Legal identity, officers, filings, Form D raises, fund vehicles |
| **M4, Web & traffic signals** | Similarweb, app-store data, job-board data | Growth signals, hiring velocity |
| **P, Private workspace** *(consent needed)* | Affinity, Attio, HubSpot, Salesforce (CRM); Notion, Google Drive, Gmail, Slack | The user's own relationship history, previous passes or meetings, internal notes. Especially relevant to gate G1 and to warm-intro paths |

---

## 3. Who uses what

| Agent | Families to use when available |
|-------|-------------------------------|
| deduplicator | M3 first (registry anchors), then M1 (database IDs and domains) |
| startup-profiler | M1 (rounds, investors, headcount), M2 (founders, team), M3 (legal, filings), M4 (traction) |
| vc-profiler | M1 (portfolio, deal log, funds, lead role), M2 (partners current?), M3 (fund vehicles, Form D) |
| vc-scout / startup-scout | M1 for long-list sourcing (investors in comparable companies; companies matching filters), then web to confirm |
| match-scorer | Only to settle a specific gate or dimension dispute |
| devils-advocate | Any family, to cross-check, **and** the web, to check connector facts independently |
| coordinator | None. It only passes the inventory path in task cards |

Agents use **read-only** tools only. They never create, update, delete, send,
post or enrich records in any connector.

---

## 4. Evidence rules for connector data

These extend `matching-rubric.md` §1.

- **Tier:** M1, M2 and M4 records are **T2** (curated databases). M3 registry
  records are **T1**. Family-P records are **T1 only for facts about the user's
  own firm's relationship** (e.g. "we met them in 2025-06", "we passed in
  2024"). Otherwise they are T3.
- **Source line format:** `connector:<server>/<tool> record <id or permalink>`,
  with the record's own last-updated date if it has one (otherwise
  `retrieved YYYY-MM-DD`), e.g.
  `source: connector:crunchbase/get_organization record acme-robotics (T2, updated 2026-08-12)`.
- **Independence:** a connector record and the same provider's public web page
  are **one** source. Two databases that both cite the same press release are
  one source (LAU). A connector record plus an independent T1/T2 web source
  can make a fact VERIFIED.
- **Gate-deciding facts** (G1–G6) and facts behind a score of 3–4 need at least
  one source **outside** the connector. Databases lag, merge namesakes and
  mislabel lead investors. The Devils-Advocate checks this.
- **Conflicts:** when a connector and a T1 web source disagree, T1 wins. When a
  connector and a T2 web source disagree, the fact is SINGLE-SOURCE at best
  until a third source settles it. Record both values.
- **Recency:** use the record's own updated date, not the retrieval date, for
  the 24-month window.
- **Private data:** facts from family P are tagged `[private]` in dossiers and
  in the report, so the user can tell what must not be shared outside the firm.
  Never quote private emails or notes verbatim beyond what a fact needs.

---

## 5. Failure handling

- Connector errors (auth, rate limit, plan restriction) never stop a run. The
  agent falls back to the web and records the failure under `## Gaps`.
- If a connector returns an entity whose domain or HQ does not match the
  identity card, treat it as a **namesake** (ENT drift), not a data point.
- Do not loop on paid or rate-limited APIs. Keep within the call budget the
  Coordinator gives each agent for connectors.
