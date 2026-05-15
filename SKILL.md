---
name: snowday-lo-researcher
description: >
  Discovers high-quality learning opportunities (LOs) for high school students not already in the
  Snowday database (snow.day). Takes a subject domain as input (e.g., "engineering", "computer science",
  "biology", "math", "business", "arts") and runs a multi-agent pipeline: mapping existing Snowday
  programs, dispatching 4 parallel research agents to find niche programs, running parallel validation
  and pro/con debates, then making final keep/remove decisions via a judge agent. Produces a curated
  markdown report with at least 5 recommended LOs (name, provider, URL, type, deadline, cost,
  justification) plus a removed-programs log.

  Use when someone asks to: "find learning opportunities for Snowday", "research programs for high
  schoolers in [domain]", "add programs to Snowday database", "Snowday LO research", "find
  extracurriculars for high school students in [field]", or expand the Snowday database in any subject.
---

# Snowday Learning Opportunity Researcher

Orchestrates a multi-agent pipeline to surface high-quality, undiscovered learning opportunities (LOs)
for high school students. You — the Claude session reading this skill — are the orchestrator. You spawn
specialized agents for research, validation, debate, and judgment. Your role is coordination and synthesis,
not inline research. Each phase produces a file that the next phase reads.

> **Architecture note**: Sub-agents cannot themselves spawn further sub-agents. All agents in this
> pipeline are spawned directly by you (the orchestrator) using the Agent tool. Sub-agents write results
> to files and return only a brief summary.

## Input

The user provides a **subject domain** — e.g., "computer science", "biology", "engineering", "arts",
"social sciences", "mathematics", "medicine", "economics", "robotics", "environmental science".

## Quality Criteria (hard filters — all must be true)

1. **High school eligibility** — explicitly for grades 9–12 or ages 14–18
2. **Individual participation** — students apply directly; no school sponsorship or district partnership required
3. **Active, credible website** — real description, application process, and dates visible
4. **Not already on Snowday** — not found at snow.day
5. **Organized structure** — runs multiple weeks or has defined sessions; not a one-off event
6. **Quality signal** — prestige institution, competitive selection, strong alumni outcomes, or recognized credential

## File Naming Convention

Normalize the domain for filenames: lowercase, hyphens replace spaces.
`"computer science"` → `computer-science` | `"social sciences"` → `social-sciences`

All files go in the **current working directory** unless the user specifies otherwise.

---

## Progress Checklist

Copy this and check off each step as you complete it:
- [ ] Phase 1: Exclusion list built → `exclusion_list.txt`
- [ ] Phase 2: 4 research agents launched (parallel) → `research_agent_1.md` through `research_agent_4.md`
- [ ] Phase 3: Candidates consolidated → `lo_candidates_[domain].md`
- [ ] Phase 4: 2–3 validation agents launched (parallel) → statuses updated in candidates file
- [ ] Phase 5: Debate agent batches launched (parallel pairs) → `debate_summary.md` written
- [ ] Phase 6: Judge agent spawned → VERDICT list received
- [ ] Phase 7: Final `lo_candidates_[domain].md` written

---

## Phase 1: Map Existing Snowday Programs

**Tool to use**: WebSearch

Run 5–7 WebSearch queries to find what's already on snow.day for this domain. Use varied patterns:
- `site:snow.day "[domain]"`
- `site:snow.day "[domain] high school"`
- `site:snow.day "[related synonym]"` (e.g., for "computer science": "coding", "software", "programming")
- `site:snow.day "[domain] internship"`
- `site:snow.day "[domain] competition"`

Compile all returned program names. Write to `exclusion_list.txt`:

```
# Existing Snowday Programs — [Domain]
# Generated: [date]
[Program Name 1]
[Program Name 2]
...
```

If fewer than 3 results appear, also try WebFetch on `https://snow.day` to manually scan for programs.
If searches return nothing, write "No results found" in the file and proceed — an empty exclusion list
is valid; research agents will handle novelty checks.

Also append well-known flagship programs almost certainly already on Snowday:
- **CS/engineering**: USACO, MIT PRIMES, RSI, SEAP, MIT Beaver Works, Congressional App Challenge
- **Biology/science**: USABO, RSI, Intel ISEF, Regeneron STS
- **All domains**: Science Olympiad, DECA, FBLA, Model UN, NSDA, MATHCOUNTS

Adapt to the actual domain — only append examples that are genuinely relevant.

---

## Phase 2: Parallel Research Agents

**Use the Agent tool to launch all 4 agents in a single message** (parallel, same tool call block).
Do not research inline — your role is coordination. Sub-agents write to files and return one-line summaries.

First read `exclusion_list.txt`. Then include its full contents verbatim in every agent prompt below.

**Agent 1 — University & Institute Programs**
> Task: Find summer and year-round programs at universities, colleges, and national labs in [domain]
> for high school students. Focus on regional and mid-tier institutions — Ivy programs are likely
> already on Snowday. Apply the quality criteria below. Skip any program on the exclusion list.
> Run at least 15 distinct WebSearch queries with varied keywords. For each promising result, use
> WebFetch to read the program's actual website and verify details before including it. Focus on
> niche, regional, or recently-launched programs — skip anything broadly famous.
> Write results to `research_agent_1.md`. Return: "Agent 1 complete. Found N candidates."

**Agent 2 — Competitions & Challenges**
> Task: Find national/international competitions in [domain] that high school students can enter
> individually (not just through school teams). Look for domain-specific, newer, or less-publicized
> ones. Apply quality criteria. Skip exclusion list programs.
> Run 15+ WebSearch queries. WebFetch promising URLs to verify. Focus on niche competitions.
> Write to `research_agent_2.md`. Return: "Agent 2 complete. Found N candidates."

**Agent 3 — Research Internships & Apprenticeships**
> Task: Find hands-on research programs in [domain] where high school students work with mentors.
> Include government labs (DOE, NASA, NIH affiliates), universities, nonprofits, and startups with
> structured programs. Apply quality criteria. Skip exclusion list.
> Run 15+ searches. WebFetch to verify. Focus on lesser-known programs.
> Write to `research_agent_3.md`. Return: "Agent 3 complete. Found N candidates."

**Agent 4 — Online Programs & Fellowships**
> Task: Find selective online programs, recognized certifications, and multi-week fellowships in
> [domain] for high school students. Not generic MOOCs — programs with selective admission or
> strong credentials. Apply quality criteria. Skip exclusion list.
> Run 15+ searches. WebFetch to verify. Focus on niche or underrepresented programs.
> Write to `research_agent_4.md`. Return: "Agent 4 complete. Found N candidates."

**Each agent writes to its file using this per-candidate format:**
```
## [Program Name]
- Provider:
- URL:
- Type: [summer program / competition / internship / online / fellowship]
- Eligibility: [grades or ages]
- Description: [2–3 sentences]
- Why high quality: [specific evidence — prestige, selectivity, alumni outcomes]
- Why not already on Snowday: [what makes it niche or underrepresented]
```

**After all 4 agents complete:** Check each file exists with at least 1 valid entry (must have Program
Name, URL, Provider). If an agent returned 0 valid entries, log `"Agent [N]: 0 candidates"` and
continue with remaining files. If fewer than 2 agents produced results, spawn one additional agent
before Phase 3.

---

## Phase 3: Consolidate

Read all `research_agent_[1–4].md` files. Merge into `lo_candidates_[domain].md`.

**Deduplication rule**: A duplicate is any entry sharing the same URL (strip query params) OR the
same program name from the same provider. If similar programs have different URLs and different
providers, keep both and note for validation.

Use this template per candidate:
```markdown
## [Program Name]
- **Provider**:
- **URL**:
- **Type**:
- **Eligibility**:
- **Description**:
- **Why High Quality**:
- **Source**: [Agent 1 / Agent 2 / Agent 3 / Agent 4]
- **Validation Status**: Pending
- **Validation Notes**: —
```

---

## Phase 4: Parallel Validation Agents

**Use the Agent tool to spawn 2–3 validation agents in parallel.** Do not validate inline — 15–25
WebFetch calls inline would flood your context. Divide the candidate list into equal batches.

Each validation agent receives its batch of program names + URLs and must:
1. Use WebFetch on each URL to check eligibility and quality
2. Use WebSearch `site:snow.day "[program name]"` to confirm it's not already listed
3. If WebFetch fails or returns no usable content, mark as UNCLEAR with note "Site unreachable"

Each agent returns compact results (no raw HTML):
```
[Program Name] — PASS / FAIL / UNCLEAR
Reason: [1 sentence]
HS Eligible: yes/no/unclear | Individual Apply: yes/no/unclear | Active Site: yes/no
Key notes: [deadline, cost, selectivity if found]
```

After all validation agents return, update `lo_candidates_[domain].md`:
- Set **Validation Status** to: ✅ PASS / ❌ FAIL / ⚠️ UNCLEAR
- Fill **Validation Notes** with the agent's key notes

**❌ FAIL programs stop here.** Do not send them to Phase 5. They go directly to the removed list.

**⚠️ UNCLEAR programs proceed to Phase 5** but must be flagged — debate agents must treat unresolved
eligibility as a strike against inclusion unless they can find evidence to resolve it.

---

## Phase 5: Parallel Debate Agent Batches

**Use the Agent tool** to spawn debate pairs. Batch 4–5 programs per pair. Launch all pairs in one
message (parallel). Only include programs with ✅ PASS or ⚠️ UNCLEAR validation status.

For each batch, spawn **two agents simultaneously**:

**Advocate Agent** (one per batch):
Receives: the batch program names + their sections from `lo_candidates_[domain].md` (description +
validation notes + URL). Task: Argue FOR including each program. Must cite specific evidence from
the program's website or validation notes. For ⚠️ UNCLEAR programs: argue for inclusion only if
you can find evidence via WebFetch to resolve the uncertainty. Write to `debate_advocate_batch_[N].md`:
```
## [Program Name]
[2–4 sentences of specific, evidence-based argument for inclusion]
```
Return: "Advocate batch [N] complete."

**Critic Agent** (one per batch):
Same input. Task: Argue AGAINST each program. Look for: school-required access, vague eligibility,
unknown organization, no verifiable track record, geography/cost barriers with no aid, broken or
outdated application process. Use WebFetch if needed. Write to `debate_critic_batch_[N].md`:
```
## [Program Name]
[2–4 sentences of specific, evidence-based argument against inclusion]
```
Return: "Critic batch [N] complete."

**After all debate agents complete**, merge all advocate and critic files into `debate_summary.md`:
```markdown
## [Program Name]
**Advocate**: [paste advocate argument]
**Critic**: [paste critic argument]
```
Organize by program name alphabetically.

---

## Phase 6: Judge Agent

**Use the Agent tool** to spawn one judge agent. The judge agent's prompt:

> You are a final judge for Snowday's LO database. Read these two files:
> - `lo_candidates_[domain].md` (candidate details + validation results)
> - `debate_summary.md` (advocate and critic arguments per program)
>
> For each program, make a final KEEP or REMOVE decision using these criteria:
> - **KEEP**: Advocate's case is specific and evidence-based; critic's concerns are minor or addressable
> - **REMOVE**: Critic identifies a hard disqualifier — school-only access, not actually for HS students,
>   dead/missing website, no application process, organization unverifiable
> - **⚠️ UNCLEAR programs**: Default to REMOVE unless the advocate provided concrete evidence resolving
>   the eligibility uncertainty
>
> Output each decision as:
> `VERDICT: [Program Name] | KEEP | [1–2 sentence reason citing specific evidence]`
> `VERDICT: [Program Name] | REMOVE | [1–2 sentence reason — which criterion failed and why]`
>
> At the end, output: `KEPT: [N]  REMOVED: [N]`
> If KEPT < 5, also output: `INSUFFICIENT: Need [X] more programs.`

Collect the judge's VERDICT list from its return value. Do not have the judge write to a file —
capture its output directly.

---

## Phase 7: Final Output

Write `lo_candidates_[domain].md` from scratch using the judge's VERDICT list:

**Part 1 — Summary Table**:
```markdown
# Learning Opportunities — [Domain]
*[N] programs recommended for Snowday | Generated: [YYYY-MM-DD]*

| LO Name | Provider | URL | Type | Deadline | Cost |
|---------|----------|-----|------|----------|------|
| [name] | [org] | [url] | [type] | [date or "Check site"] | [Free / Fee / Aid available] |
```

**Part 2 — Why We Chose Each Program** (immediately after the table):
```markdown
---
## Why We Chose These Programs

### [Program Name]
[2–4 sentences. Specific quality evidence. The strongest advocate argument and why it outweighed
concerns. At least one sentence on the application process — how a student would actually apply.]
```

**Part 3 — Programs Removed** (at the bottom):
```markdown
---
## Programs Removed During Review
| LO Name | Reason |
|---------|--------|
| [name] | [specific reason from judge verdict or validation] |
```

If fewer than 5 programs were kept, add at the very top:
```
> ⚠️ Only [N] programs met the bar for this domain. Consider re-running with a broader
> domain or adjacent field (e.g., "STEM" instead of "physics").
```

Confirm the file is saved. Intermediate files (`research_agent_*.md`, `debate_*.md`,
`debate_summary.md`, `exclusion_list.txt`) are kept by default for auditability — only clean
them up if the user asks.
