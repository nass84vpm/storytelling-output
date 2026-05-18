# Storytelling flow — orchestration graph

The runtime state machine for the storytelling skill. This file describes what the orchestrator enforces deterministically: stage transitions, gate guards, tool-availability rules, state updates, interrupt points.

The companion file `storytelling.md` holds the prompt-level content — frameworks, voice rules, examples, output formatting. The two are complementary: this file says *when* things happen and *what state must be true*; the skill file says *what to say* and *how to say it*.

Where this file constrains the LLM (e.g., "data tools are blocked until audience + decision are set"), the corresponding skill-file rule is descriptive guidance; the graph is the actual enforcer. If both files disagree, the graph wins.

## Stage diagram

```
                   user input
                       │
                       ▼
              ┌────────────────┐
              │ Stage 1:       │
              │ Classify       │  (router, no LLM)
              └────────┬───────┘
                       │
              ┌────────┴─────────┐
       trigger│                  │no trigger
              ▼                  ▼
       ┌──────────────┐    exit → standard
       │ Stage 2:     │    answer flow
       │ Brief        │
       │ alignment    │
       └──────┬───────┘
              │ audience + decision set
              ▼
       ┌──────────────┐
       │ Stage 3:     │
       │ Data         │
       │ immersion    │
       └──────┬───────┘
              │ data retrieved
              ▼
       ┌──────────────┐
       │ Stage 4:     │
       │ Build outline│
       └──────┬───────┘
              │ outline complete
              ▼
       ┌──────────────┐
       │ Stage 5:     │
       │ Show + edit  │ ← INTERRUPT: wait for user
       └──────┬───────┘
              │ approved
              ▼
       ┌──────────────┐
       │ Stage 6:     │
       │ Generate     │ ← TERMINAL
       └──────────────┘
```

## State

The graph maintains:

- `output_format` — the named artefact (`presentation`, `deck`, `showreel`, `podcast`). Set by Stage 1 classification.
- `audience` — named role or group. Set by Stage 2 from user response.
- `decision` — named strategic outcome. Set by Stage 2 from user response.
- `audience_ask_count`, `decision_ask_count` — counters for the one-ask cap. Incremented each clarification.
- `arc` — macro-structure (Minto, SCQA, ABT, Hero's Journey, Data Storytelling, Theme-Led). Derived from `audience`, or set to `theme_led` if graceful degradation fires.
- `data_scope` — result of Stage 3 retrieval (projects, sample sizes, gaps).
- `outline` — the scene structure produced by Stage 4.
- `approved` — boolean. Set to true by Stage 5 user response, gating Stage 6.
- `degrade` — boolean flag set when graceful degradation fires.

## Stage 1: Classify (router)

Deterministic match. The LLM is not invoked.

### Trigger keywords (case-insensitive)

`presentation`, `deck`, `showreel`, `podcast`

### Trigger patterns

`build me a {keyword}`, `create a {keyword}`, `make me a {keyword}`, `summarise this as a {keyword}`, or any utterance naming `{keyword}` as the desired output.

### Routing

- Trigger matched → set `output_format`, transition to Stage 2.
- No trigger matched → exit the skill, hand to standard answer flow.

### Explicit non-triggers (route to standard answer flow)

Text-output keywords: `report`, `executive summary`, `one-pager`, `briefing`, `memo`, `summary`, `headlines`.

Exploratory patterns: open questions, fact lookups, "what's the story", "tell me about X".

The skill file (`storytelling.md`) Stage 1 section provides the prompt-level intuition and edge cases. This file is the authoritative router spec.

## Stage 2: Brief alignment

### Entry action

Emit the One-Gate question (text from `storytelling.md` 2b) as the first user-facing message. Wait for response.

The original request is *not* an answer to the One-Gate question. The graph does not allow Stage 2 to be skipped or short-circuited even if the LLM thinks it can infer the answers.

### Tool availability

Blocked until `audience != null AND decision != null`:

- `findProjects`
- `searchProjects`
- `listProjects`
- `getProjectAnalytics`
- `getProjectTranscripts`
- `semanticVideoSearch`
- `searchProjectsByTranscript`
- `getSegmentAnalysisResults`

If the LLM attempts any of these tools, the graph rejects the call. The LLM is then re-prompted to emit the correction message (text from `storytelling.md` Behaviour rules).

### State updates from user response

Parse the user's reply to the One-Gate question:

| Parsed response | State update | Next action |
| --- | --- | --- |
| Audience + decision both named | set `audience`, `decision` | → Stage 3 |
| Audience only | set `audience`; increment `decision_ask_count` | Re-ask for decision (partial-answer pause) |
| Decision only | set `decision`; increment `audience_ask_count` | Re-ask for audience (partial-answer pause) |
| Skip (*"just build it"*, *"I don't know"*) or vague (*"stakeholders"*, *"the team"*) | mark `degrade: true` | → Stage 2d (graceful degradation) |

### Validation — "locked in"

Validate parsed anchors against criteria (defined in `storytelling.md` 2e):

- `audience` is a named role/group, not a vague placeholder.
- `decision` is a named strategic outcome, not an open-ended exploration.

If validation fails, treat the response as vague → graceful degradation.

### One-ask cap

If `audience_ask_count > 1` or `decision_ask_count > 1` while the anchor remains unset, do not ask again. Mark `degrade: true`, route to Stage 2d.

### Graceful degradation (Stage 2d)

Triggered by: explicit skip, vague response that fails validation, one-ask cap hit.

When triggered:

- Set `arc = theme_led`.
- Skip audience-specific macro-structure logic.
- Continue to Stage 3 with `degrade: true` flag.
- Stage 5 emits the soft re-prompt for the audience (text from `storytelling.md` 2d).

## Stage 3: Data immersion

### Pre-flight guard

Before any data tool is callable in Stage 3:

```
audience set AND validates AND
decision set AND validates
```

If either check fails, the graph rejects tool calls and routes back to Stage 2 with a clarification question.

### Path branching after retrieval

| Retrieval result | Path | Next |
| --- | --- | --- |
| Relevant data found, complete | A: Ready | Emit data preamble, → Stage 4 |
| Project still in field | B: In-field | Emit in-field question, branch on response |
| Relevant data missing entirely | B: Gap | Emit gap question, branch on response |
| Sample <5 participants on remaining data | A: Ready, flagged | Emit preamble + directional Pro Tip, → Stage 4 |

### Path B branching

Each Path B variant ends with a user question. The graph routes:

- "Run with what's there" / "use as directional baseline" → set `data_scope` to partial, → Stage 4 (with reduced-scope flag).
- "Wait" / "run more research" → exit the skill, hand to project completion flow.

## Stage 4: Build outline

No graph branching. The LLM builds `outline` using `audience`, `decision`, `output_format`, `data_scope`. Construction rules live in `storytelling.md` Stage 4.

## Stage 5: Show + edit loop

### Interrupt — user approval required

The graph emits the outline + alignment question, then **pauses**. Stage 6 cannot fire without explicit approval in the next user turn.

### Edit-loop routing

User response routing:

| Response type | Action |
| --- | --- |
| Approval (*"yes"*, *"looks good"*, *"build it"*, silence) | Set `approved = true`, → Stage 6 |
| Prompt-based edit (*"merge X and Y"*, *"lead with Z"*) | Re-run Stage 4 with edit applied, return to Stage 5 |
| Edit that triggers pushback (cherry-pick, contradicts data, no measurable outcome) | Emit pushback message (skill file pattern), wait for user response, do not advance |
| Reject (*"scrap this"*, *"start over"*) | Reset `outline`, return to Stage 2 |

## Stage 6: Generate output

### Terminal action

Single tool call to the generation engine with `outline`, `output_format`, `audience`, `decision`, `arc`, `data_scope`.

The LLM emits the handoff message (text from `storytelling.md` Stage 6) and the graph stops. No further LLM turns in this skill.

## Forbidden patterns — graph enforcement

The skill file lists behaviours the LLM should avoid. Where the graph can enforce them deterministically, it does:

| Pattern | Graph enforcement |
| --- | --- |
| Data tool call before audience + decision are set | Block the tool call; re-prompt for the correction message |
| LLM question with content after it ("in the meantime, here's…") | Detect via response parser; reject and re-prompt |
| Multiple unrelated questions stacked in one message | Detect via response parser; reject |
| Stage 6 fired without Stage 5 approval | Block; return to Stage 5 |
| Conversation continues after Stage 6 | Terminate the skill; further messages handled by other flows |
| "Since we haven't locked in…" emitted as first response in Stage 2 | Detect string match; reject and re-prompt to ask the One-Gate question instead |
