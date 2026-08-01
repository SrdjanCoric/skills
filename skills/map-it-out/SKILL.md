---
name: map-it-out
description: Manually map and resolve a large, uncertain initiative as local decision records, dependencies, frontier, fog, and scope until it is ready for a planning brief. Use only through explicit start, resume, status, or release commands.
disable-model-invocation: true
---

# Map It Out

Map a large initiative whose route cannot fit safely in one planning context. Store the map and its
decisions as local Markdown, resolve one available decision per resumed session, and stop when the
route to the destination is clear. Plan by default. Do not implement the destination or create build
tasks.

Invoke manually:

```text
/skill:map-it-out start <initiative>
/skill:map-it-out resume <initiative-or-map-path> [decision-name-or-path]
/skill:map-it-out status <initiative-or-map-path>
/skill:map-it-out release <initiative-or-map-path> <decision-name-or-path>
```

Explicit invocation authorizes creation or updates only under the selected decision-map directory
and any specialized planning artifact path approved by this workflow. It does not authorize source
changes, production access, destructive actions, commits, pushes, or implementation planning.

## Ownership boundary

`map-it-out` owns all durable navigation state:

- destination and standing notes;
- decision records and their statuses, types, dependencies, and claims;
- the derived frontier;
- decisions already made;
- not-yet-specified areas;
- out-of-scope areas; and
- readiness and the final planning handoff.

`talk-it-through` is a stateless decision-resolution engine. Invoke it in embedded-decision mode for
one supplied human decision. It must return a structured result and must not write or update the map.
`research-for-planning` owns authoritative external research artifacts. A configured prototype
workflow owns an approved disposable prototype. `map-it-out` classifies and persists their results.

## Local artifact model

Choose the map directory in this order:

1. a path explicitly supplied by the user;
2. an existing matching directory under `plans/decision-maps/`;
3. `plans/decision-maps/<initiative-slug>/`.

Never create two maps for the same destination. Use this structure:

```text
plans/decision-maps/<initiative>/
  map.md
  decisions/
    001-<decision>.md
    002-<decision>.md
  research/
    <topic>.md
  prototypes/
    <artifact-or-pointer>
```

The map is a low-resolution index, not a duplicate store. A decision's question, evidence,
alternatives, and resolution live only in its decision file. `map.md` contains only the destination,
standing notes, one-line resolved-decision pointers, fog, scope exclusions, and readiness. Derive
the live frontier by scanning decision-file metadata rather than copying open decisions into the
map.

### Map format

```markdown
# Decision map: <initiative>

**Status:** mapping | active | ready

## Destination
<One or two lines defining what reaching the end of this map means.>

## Notes
<Domain constraints, source documents, standing preferences, and skills later sessions need.>

## Decisions so far
- [<resolved decision name>](decisions/<file>.md): <one-line result>

## Not yet specified
<In-scope areas that matter but cannot yet be stated as precise questions.>

## Out of scope
<Work beyond the destination, with a reason and a link when a decision record exists.>

## Handoff
<Final brief path and next command once ready, or "Not ready".>
```

### Decision format

```markdown
# Decision: <name>

**Status:** open | active | resolved | out-of-scope
**Type:** discussion | research | prototype | prerequisite
**Mode:** human | agent
**Depends on:** <linked decision names, or "none">
**Claim:** <current invocation marker when active, or "none">

## Question
<One precise question this record resolves.>

## Context
<Only context needed to understand the question.>

## Evidence
<Repository facts and links to research, design, or prototype artifacts.>

## Resolution
- **Decision:** <answer, or "Unresolved">
- **Rationale:** <material reason>
- **Alternatives rejected:** <durable alternatives, or "None">
- **Consequences:** <constraints created by the answer>
- **Non-goals:** <explicit exclusions>
- **Required proof:** <test or acceptance evidence later work must provide>

## Follow-on
- **Newly sharp decisions:** <links, or "None">
- **Still-foggy areas:** <map entries, or "None">
```

Use stable numbered file names, but refer to decisions by linked name in human-facing prose. Never
use a bare number as the decision's human-readable identity.

## Writing quality

Every map, decision record, research summary, prototype explanation, and final handoff is durable
human-facing documentation. Invoke `write-well` whenever creating or changing prose in those files.
In write mode, complete its draft and full adversarial audit before saving. For existing prose, use
review mode, apply every supported fix that preserves meaning, and complete the audit before saving.
A mechanical metadata-only change such as `open` to `active` does not require a prose audit.

Keep the `write-well` audit plan in working context and report its pass count. Do not write the audit
plan into the decision map unless the user asks for it.

## Decision states and frontier

A question becomes a decision file when it can be stated precisely, even if another decision blocks
it. Keep an area in `Not yet specified` when the question itself is still unclear. Do not create
speculative decision files merely to empty the fog section.

A decision is available when it is `open`, every linked dependency is `resolved`, and it has no
claim. The frontier is the set of available decisions in file-number order. An `active` decision is
claimed. Never work on it from another invocation. If a claim is stale, require an explicit
`release` invocation before changing it back to `open`.

An out-of-scope decision is closed and never returns to the frontier. Record its one-line reason in
`map.md` and keep it out of `Decisions so far`, which records decisions on the route to the
destination.

## Start mode

Use when the user supplies a loose initiative.

1. Inspect relevant repository facts and source documents. Use an isolated read-only subagent for broad
   reconnaissance when available.
2. Invoke `talk-it-through` in embedded-decision mode to establish one precise destination and its
   scope boundary. Do not ask it to map the initiative.
3. Explore breadth-first across actors, behavior, state, interfaces, dependencies, safety, proof,
   and external facts. Identify precise open questions without resolving them. Keep unclear future
   areas as fog.
4. If there is no material fog and the whole discussion fits one bounded planning context, do not
   create a map. Recommend standalone `/skill:talk-it-through <topic>` and stop.
5. Otherwise create `map.md` and the decision files that can be stated precisely now. Create every
   file before adding dependency links so references are valid.
6. Classify each decision by type and mode. Discussion and prototype decisions are human-led.
   Research is agent-led. A prerequisite may be agent-led only when it is safe, local, and exists
   solely to unblock a decision.
7. Run independent external research decisions in parallel through `research-for-planning` when
   doing so cannot depend on another open decision. Store each artifact under `research/`, record
   its result in the owning decision, and update the map. Do not resolve human decisions during
   charting.
8. Audit all new prose through `write-well`, return the map path and derived frontier, then stop.

## Resume mode

Resolve at most one non-research decision per invocation.

1. Resolve the map path. Read `map.md` and only the metadata headers of decision files first. Derive
   the frontier without loading every decision body.
2. If the user names a decision, verify that it is open, unblocked, and unclaimed. Otherwise choose
   the first frontier decision. If none is available, report blocked and active decisions by linked
   name and explain what prevents progress.
3. Set the selected decision to `active` and add the claim before doing work. Preserve an active
   claim if the invocation is interrupted.
4. Read that decision and only directly related resolved records or evidence.
5. Route by type:
   - `discussion`: invoke `talk-it-through` in embedded-decision mode with the destination,
     question, context, resolved dependencies, constraints, and expected result.
   - `research`: invoke `research-for-planning` and store its cited artifact under `research/`.
   - `prototype`: state the single uncertainty a disposable artifact would answer and obtain user
     approval before invoking an available prototype workflow. Otherwise leave it open.
   - `prerequisite`: perform only safe planning-enablement work. Require explicit approval for
     external accounts, credentials, destructive actions, real data, or trust-boundary changes.
6. Persist the result in the selected decision. On resolution, set it to `resolved`, clear the
   claim, and append one linked one-line result to `Decisions so far`.
7. Treat `talk-it-through` follow-on outputs as proposals. Create decision files only for questions
   now precise enough to state. Put unclear areas in `Not yet specified`. Add dependencies after new
   files exist. Move anything beyond the destination to `Out of scope`.
8. Invoke `write-well` for every prose addition or revision, audit the affected complete documents,
   return the updated frontier and fog, then stop.

Independent research decisions may be delegated in parallel. Their results can close more than one
research record in a session, but no second human decision may be resolved.

## Status and release modes

`status` reads the map and decision metadata without changing files. Return destination, readiness,
resolved-decision count, frontier by linked name, blocked decisions and dependencies, active claims,
fog, and out-of-scope count.

`release` resets one `active` decision to `open` and clears its claim only after confirming that no
resolution was recorded. It changes metadata only. Never release a claim merely because another
invocation wants the decision.

## Completion and handoff

The route is clear only when:

- no decision is `open` or `active`;
- every dependency is resolved or explicitly out of scope without leaving a broken route;
- `Not yet specified` contains no material in-scope fog; and
- the destination has enough behavioral, state, interface, safety, and proof detail for its stated
  handoff.

When the route is clear, set the map to `ready`. If the destination is planning input, invoke
`write-planning-brief` with the map, every resolved decision, and all research, design, and prototype
artifacts. Then invoke `write-well` in review mode on the resulting brief, apply supported fixes,
and complete its full audit. Record the brief path and recommended canonical command in `Handoff`:

```text
/skill:write-a-prd <brief-path>
```

when a product-facing PRD adds value, otherwise:

```text
/skill:to-plan <brief-path>
```

Do not invoke either command automatically. If the destination is a final decision or another
artifact rather than planning input, record that completed artifact in `Handoff` and stop.
