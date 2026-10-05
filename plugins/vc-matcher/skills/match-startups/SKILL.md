---
name: match-startups
description: Find up to three high-conviction startups for a named VC investor, using the vc-matcher multi-agent pipeline (profiling, scouting, adversarial fact audit, independent scoring) with web search plus any venture-data connectors the user has. Returns only matches that clear the shared rubric's HIGH-conviction bar.
argument-hint: <VC name> [optional context, e.g. "seed fund, focus on Nordics, B2B SaaS"]
disable-model-invocation: true
allowed-tools: Agent, SendMessage, Read, Write, Glob, AskUserQuestion, ToolSearch, ListMcpResourcesTool, Bash(date *)
---

> **Reference files.** Paths written as `${CLAUDE_PLUGIN_ROOT}/references/…`
> point into the vc-matcher plugin. If that prefix appears unexpanded, or the
> file isn't there (for example because these files were loaded from a
> project's `.claude/` folder rather than installed as a plugin), find the
> files with Glob `**/vc-matcher/references/*.md` and use those paths, also
> in any task cards or prompts you pass on.

# /match-startups: find startups for a VC

User input: `$ARGUMENTS`

You are the **main session**. You set up the run, hand it to the
`coordinator` agent, deal with anything that needs the user, and present the
result. **Do no research and add no facts yourself.**

## 1. Parse the input

- `SUBJECT`: the VC name. Take the first quoted string if there is one,
  otherwise the text before the first comma or context words. Keep the user's
  exact spelling.
- `CONSTRAINTS`: everything else (a specific fund vehicle, a partner, sectors
  or regions to focus on, a stage, exclusions, the fund's website URL). Pass it
  on verbatim.
- If `$ARGUMENTS` is empty, ask the user for the VC name with
  AskUserQuestion. Suggest that adding the fund's domain helps with identity.

## 2. Set up the run

- `RUN DATE`: today's date (run `date +%Y-%m-%d` if you are not certain).
- `slug`: lowercase SUBJECT, non-alphanumerics → `-`.
- `RUN ID`: `match-startups-<slug>-<YYYYMMDD-HHMM>` (`date +%Y%m%d-%H%M`).
- `RUN DIR`: `./matcher-reports/.work/<RUN ID>/`
- `REPORT PATH`: `./matcher-reports/match-startups-<slug>-<YYYY-MM-DD>.md`
  (if it exists, append `-2`, `-3`, …).

Create the run dir by writing `<RUN DIR>/README.md` containing the RUN ID,
RUN DATE, SUBJECT and CONSTRAINTS.

### 2b. Connector inventory

Follow `${CLAUDE_PLUGIN_ROOT}/references/connectors.md` §1 to find which MCP
data connectors this session has (Crunchbase, PitchBook, Dealroom, Harmonic,
registries, CRMs, etc.):

1. Look at the `mcp__*` tools available to you. If tools are deferred, run
   `ToolSearch` with the family keywords listed in connectors.md §1.
2. Classify each server into a family (M1–M4, P). Ignore servers that fit none.
3. Probe each data server once with a cheap **read** call. Keep only those that work.
4. If any **private** (family P) connector is usable, ask the user once with
   AskUserQuestion (multi-select) which of them this run may read. Default to none.
5. Write the inventory to `<RUN DIR>/02-connectors.md`, or write `No data
   connectors detected: web-only run.`

Tell the user in one line which connectors this run will use.
Never call write-type connector tools (create, update, send, delete, …).

## 3. Hand off to the coordinator

Invoke the `Agent` tool with subagent type `vc-matcher:coordinator` (or
`coordinator` if not namespaced) and this prompt, filled in:

```
MODE:        match-startups   (subject is a VC; find startups for it)
SUBJECT:     <SUBJECT>
CONSTRAINTS: <CONSTRAINTS or "none">
RUN DATE:    <RUN DATE>
RUN DIR:     <RUN DIR>
REPORT PATH: <REPORT PATH>
CONNECTORS:  <RUN DIR>/02-connectors.md
POLICY:      Up to 3 matches; HIGH conviction only; return fewer rather than pad.
             Deliberation ≤ 3 rounds. Web research always; MCP data connectors as well, per 02-connectors.md.
             Rubric: ${CLAUDE_PLUGIN_ROOT}/references/matching-rubric.md
             Protocol: ${CLAUDE_PLUGIN_ROOT}/references/protocol.md
```

Do not set a `model` for the coordinator. It chooses the models for the
agents it runs.

## 4. Handle the coordinator's status

- **`NEEDS_DISAMBIGUATION`**: show the candidate identities (name, domain, HQ,
  one distinguishing fact) with AskUserQuestion. Pass the user's choice back to
  the coordinator: resume it with `SendMessage` if you have its ID, otherwise
  invoke it again with the same RUN DIR plus `RESUME: subject resolved to <ENT-id/description>`.
- **`BLOCKED`**: relay the coordinator's question to the user, then resume as above.
- **`DONE`** or **`PARTIAL`**: go to step 5.

## 5. Present the result

Read the report at REPORT PATH. In chat, give a **short** summary taken only
from the report:

1. Verdict line (e.g. "3 HIGH-conviction startups out of 3 sought").
2. For each HIGH match: name, score/100, and a one-line "why", with its key source link.
3. One line each: how many near-misses and exclusions, plus the most important gap.
4. The report path, plus a note that the full evidence, deliberation logs and
   sources are in the report and the run dir.

If the coordinator returned `PARTIAL`, say clearly what is missing.
