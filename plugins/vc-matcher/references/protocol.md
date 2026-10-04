# vc-matcher Operating Protocol

How the eight agents work together: topology, progressive disclosure, message
formats, identity checks, deliberation, drift detection and model selection.
Read alongside `matching-rubric.md`. Every agent follows both documents.

---

## 1. Topology

```
 main session (skill /match-vcs or /match-startups)
   └── coordinator                       depth 1   delegates, never researches
         ├── startup-profiler            depth 2 ─┐
         ├── vc-profiler                 depth 2  │ each may call
         ├── vc-scout / startup-scout    depth 2  ├─► deduplicator   depth 3 (leaf)
         ├── match-scorer                depth 2  │
         └── devils-advocate             depth 2 ─┘
```

- **The Coordinator never runs WebSearch or WebFetch itself.** It reads the
  request, decides which agents to run, in what order, on which model and with
  what budget. It relays deliberation between agents and assembles the final
  report from their files.
- **Every agent except the Coordinator** calls the Deduplicator before treating
  any named entity as identified (§4).
- **The Deduplicator is a leaf.** It never spawns agents.
- Agents never talk to each other directly. All traffic goes through the
  Coordinator, which keeps the record (§2) consistent.

### Fallback when nested spawning is unavailable

If an agent finds it has no `Agent` tool (spawn depth limit reached, or the
runtime withholds it), it must **not** skip identity checks. Instead it adds a
`DEDUP_REQUESTS` block to its return envelope (§3.3) and marks every affected
fact `UNVERIFIED (identity pending)`. The Coordinator then runs the
Deduplicator itself, as a relay only, and re-invokes the agent with the
identity cards.

---

## 2. The run directory

Every run has a working directory, the **run dir**, created by the main session:

```
./matcher-reports/.work/<run-id>/          run-id = <mode>-<slug>-<YYYYMMDD-HHMM>
  00-plan.md                coordinator: interpreted request, phases, models, budgets
  01-identity/ENT-*.md      deduplicator: one identity card per entity
  10-subject-profile.md     profiler: dossier of the subject (the startup or VC named by the user)
  11-subject-audit.md       devils-advocate: audit + deliberation log for the subject
  20-scout.md               scout: ideal-match spec, long list, gate screen, shortlist
  21-scout-audit.md         devils-advocate: audit of the scout's claims
  3n-candidate-<slug>.md    profiler: dossier of each shortlisted candidate
  3n-candidate-<slug>-audit.md
  40-scoring.md             match-scorer: scorecards + scorer↔scout deliberation log
  41-scoring-audit.md       devils-advocate: audit of scoring claims
  90-ledger.md              coordinator: status of every fact ID (confirmed / rejected / unverified)
```

The final report is written to `./matcher-reports/<mode>-<slug>-<YYYY-MM-DD>.md`.

Files are the shared memory. **Messages between agents carry short summaries
and file paths, never whole dossiers.**

---

## 3. Progressive disclosure

Information moves in four levels. Each agent receives the **smallest level that
lets it do its job** and reads deeper only when a specific question requires it.

| Level | What | Size | Where it lives |
|-------|------|------|----------------|
| **L0, Task card** | What to do, for which entity, with which constraints | ≤ 200 words | In the Agent prompt from the Coordinator |
| **L1, Summary** | Headline findings, key fields, confidence, open issues | ≤ 300 words | The agent's return envelope |
| **L2, Dossier** | Full structured findings, section by section, every fact in rubric §1.4 format | No limit | A run-dir file |
| **L3, Evidence** | Source URLs, quotes and dates behind each fact | No limit | Inside the L2 file, under each fact |

Rules:

1. **Downward: give tasks, not context dumps.** The Coordinator passes an L0
   task card plus *paths* to any L2 files the agent may need. It never pastes
   dossier contents into a prompt.
2. **Upward: return L1 only.** An agent writes its L2 file, then returns only
   its L1 envelope. The caller opens the file only when it needs more.
3. **Read by section.** Dossiers use stable `##` headings (listed in each
   agent's file) so a reader can open only the section it needs: a scout reads
   `## Stage & round` of a profile, not the whole thing.
4. **Escalate detail on demand.** A downstream agent that needs a fact not in
   L1 reads the relevant L2 section. If the fact is missing from L2, it says so
   in its envelope (`GAPS:`) and does not fill the gap from memory.

### 3.1 Task card (L0) format

```
TASK CARD — <agent> — run <run-id> — run date <YYYY-MM-DD>
Objective:   <one sentence>
Entity:      <name> (ENT-id if already resolved)
Inputs:      <paths to read, with the sections that matter>
Constraints: <user constraints, e.g. "raising Series A, EUR 8M, H1 2027">
Output file: <path to write>
Budget:      <max searches / fetches; model chosen>
Rubric:      ${CLAUDE_PLUGIN_ROOT}/references/matching-rubric.md
Protocol:    ${CLAUDE_PLUGIN_ROOT}/references/protocol.md
```

### 3.2 Context hygiene

- Each agent restates the **objective** and **entity ENT-id** at the top of its
  file and checks its output against both before returning (anti-drift).
- Never rely on memory or training data for facts about specific companies,
  people, funds or deals. Training-data recall may be used only to pick search
  terms, never as evidence.

### 3.3 Return envelope (L1)

Every agent ends with exactly this block:

```
=== ENVELOPE ===
AGENT:        <name>
STATUS:       DONE | NEEDS_DISAMBIGUATION | BLOCKED | PARTIAL
ENTITY:       <canonical name> (ENT-id)
FILE:         <path written>
SUMMARY:      <≤ 300 words>
KEY FACTS:    <fact IDs most relevant downstream, with confidence>
GAPS:         <what could not be established, and why>
DEDUP_REQUESTS: <only in fallback mode: name + context per entity>
OPEN OBJECTIONS: <for deliberation rounds: IDs still disputed>
=== END ===
```

---

## 4. Identity protocol (Deduplicator)

### 4.1 When it is mandatory

- **Before** profiling, scouting or scoring the subject entity.
- **Before** adding any candidate VC or startup to a long list *and* again before
  it reaches a shortlist (the second check re-confirms with the fuller context).
- For any **portfolio company, co-investor, founder or partner** whose identity
  decides a gate (G1, G3) or a score of 3–4.
- Whenever two sources describe "the same" entity with conflicting basics
  (country, founding year, domain, founders).

### 4.2 Request format (what the caller sends)

```
DEDUP REQUEST
Name as found:   <string>
Context:         <where it was found + what the caller believes it is>
Known anchors:   <domain, country, founders, founding year, if any>
Decision at stake: <e.g. "G3: is this the competitor in VC X's portfolio?">
```

### 4.3 Identity card (what the Deduplicator returns and writes to `01-identity/`)

```
ENT-<type>-<slug>   (type: VC | SU | PER | FUND)
Canonical name:   ...
Legal name(s):    ... (registry + number where available)
Domain(s):        ...
HQ / registered:  ...
Founded:          ...
Key people:       ...
Affiliated entities: <parent funds, fund vehicles, rebrands, former names>
Confusables:      <list of distinct entities with similar names, each with a one-line distinguishing anchor>
Verdict:          RESOLVED | AMBIGUOUS (list candidates) | NOT FOUND
Confidence:       VERIFIED | CORROBORATED | SINGLE-SOURCE
Evidence:         <URLs>
```

An **AMBIGUOUS** verdict on the user's subject entity stops the run. The
Coordinator returns `NEEDS_DISAMBIGUATION` to the main session, which asks the
user. An AMBIGUOUS candidate is dropped unless resolved.

---

## 5. Deliberation protocol

Two loops, both with a **3-round maximum** (rubric §5).

### 5.1 Fact audit: Devils-Advocate ↔ originating agent

**Round structure**

1. The Coordinator sends the DA a task card pointing at the file to audit.
2. The DA checks **every** fact (sampling is not allowed for gate-deciding
   facts or facts behind scores of 3–4; it may sample other facts but must
   say so). For each fact it either confirms it or writes an objection:

```
[OBJ-<nn>] on [F-xx-nn]   type: <drift type from §6>   severity: BLOCKING | MAJOR | MINOR
  Claim:    <what the agent asserted>
  Problem:  <what is wrong, with the DA's own evidence/URL>
  To resolve: <exactly what evidence would satisfy the DA>
```

3. The Coordinator relays the objections to the originating agent (resuming it
   with `SendMessage` if available, otherwise a fresh invocation with the
   dossier and audit paths). The agent answers **each** objection:

```
[RESP-<nn>] to [OBJ-<nn>]   DEFEND | AMEND | WITHDRAW
  New evidence: <URL + quote>   (required for DEFEND)
  Change:       <what was changed in the dossier>   (required for AMEND)
```

4. The DA rules on each response: `RESOLVED` or `STANDS` (with reason).
5. Repeat up to round 3. After round 3, any `STANDS` objection is resolved
   **against** the claim: the fact becomes UNVERIFIED and is removed from gates
   and scores.

**Consensus** means no BLOCKING or MAJOR objection remains open. MINOR
objections can stay open if they are disclosed in the report.

### 5.2 Match deliberation: Match-Scorer ↔ scout

1. The scout's shortlist includes its own provisional gate results and
   dimension scores for each candidate.
2. The Match-Scorer scores each pair independently **before reading the
   scout's scores**. It then compares, and for every gate disagreement and every
   dimension that differs by ≥ 1 point writes a `[CRIT-<nn>]` with evidence.
3. The scout answers each critique with DEFEND / AMEND / WITHDRAW, as in §5.1.
4. Up to 3 rounds. Consensus = gates agree **and** totals within 10 points.
   If there is no consensus after round 3, the **lower** total and the
   **stricter** gate result stand, and the match drops one conviction level.
5. The DA then audits the Match-Scorer's final scorecards (§5.1). Any fact
   that becomes UNVERIFIED triggers a rescore of the affected dimensions.

### 5.3 Rules of engagement

- **Concede to evidence, not persistence.** If an agent changes its position
  without new evidence or a shown error, the DA treats that as
  *sycophantic collapse* (§6) and the original dispute remains open.
- **No new claims in rebuttals without sources.**
- **The DA is also audited.** Every objection needs its own source or a
  specific logical flaw. An unsupported objection is dismissed.

---

## 6. Drift and failure catalogue

The Devils-Advocate checks for each of these, and every agent self-checks
before returning.

| Code | Failure | Typical symptom |
|------|---------|-----------------|
| **ENT** | Entity drift / conflation | Facts from "Amadeus Capital Partners" (UK) attributed to an unrelated "Amadeus Capital" elsewhere; a startup confused with a namesake in another country |
| **TMP** | Temporal drift | Stale facts presented as current; a 2021 thesis treated as today's; a former partner listed as current; outdated headcount |
| **SCP** | Scope drift | Answering a different question: profiling the parent company, a different fund vehicle, or the wrong round |
| **CRT** | Criteria drift | Relaxing a gate or anchor "because the fit is otherwise great"; inventing new criteria |
| **SRC** | Source hallucination / misattribution | URL does not contain the claim; quote not on page; dead link presented as evidence |
| **LAU** | Citation laundering | Several aggregators copying one origin, counted as independent corroboration |
| **NUM** | Number drift | Fund size vs ticket size; round size vs this investor's cheque; currency or year missing; ranges collapsed into a point |
| **ROL** | Role drift | "Participated" reported as "led"; angel investment by a partner counted as a fund investment; advisor counted as founder |
| **INF** | Inference passed off as fact | INFERRED value labelled VERIFIED; demographic inference stated without method |
| **CNF** | Confirmation / anchoring bias | Only supporting evidence searched; first search result anchors the whole profile; no disconfirming search performed |
| **SUR** | Survivorship / selection | Only famous portfolio companies counted; failures and recent small deals ignored, skewing stage or ticket estimates |
| **SYC** | Sycophantic collapse | Concession in deliberation without new evidence |
| **GAP** | Gap filling | Missing data silently filled with plausible values or training-data recall |
| **OBJ** | Objective drift | The final output no longer answers the user's request (wrong direction, wrong count, missing constraint) |

---

## 7. Model selection (Coordinator responsibility)

The Coordinator chooses a model for every agent invocation and records the
choice and reason in `00-plan.md`. Goal: **no loss of accuracy**. Token
efficiency is pursued only where it carries no accuracy risk.

### 7.1 Tiers

| Tier | Alias | Use for |
|------|-------|---------|
| **Deep** | `opus` | Devils-Advocate (always); Match-Scorer (always); scouts; profilers for entities with thin, conflicting or non-English coverage; any Deduplicator call with a known name collision |
| **Standard** | `sonnet` | Profilers for well-documented entities (rich T1 site, multiple T2 articles); Deduplicator for routine checks; re-runs that only patch named gaps |
| **Light** | `haiku` | Only mechanical tasks with no judgement, e.g. re-confirming an already RESOLVED identity card after a minor context change. Never for auditing, scoring, scouting or first-pass profiling |

If the runtime offers other model aliases, the Coordinator may use them,
mapped to the tier that fits their capability, and records why.

### 7.2 Escalation rules

- If the DA raises ≥ 2 BLOCKING/MAJOR objections of type SRC, GAP or ENT
  against a Standard-tier run, the agent's next round is re-run on **Deep**.
- Any agent returning `PARTIAL` because of complexity (not because data is
  unavailable) is re-run once on a higher tier.
- Do not downgrade within a run once an entity has needed escalation.

### 7.3 Mechanics

Pass the chosen alias in the `model` parameter of each `Agent` call (and of
`SendMessage` when resuming). Plugin agents default to `model: inherit`. If
the runtime ignores or rejects the per-call `model` parameter, the session
model applies. Record that in `00-plan.md` and continue; never skip an agent
because its preferred model is unavailable.

### 7.4 Budgets

The Coordinator also sets per-agent search budgets in the task card (typical:
profiler 20–40 searches/fetches; scout 30–60; DA 1–3 verification lookups per
gate-deciding fact; Deduplicator 3–8). Budgets are ceilings, not targets. An
agent that hits its budget with gate-deciding facts still unverified returns
`PARTIAL` and lists them under `GAPS`. It must not guess.
