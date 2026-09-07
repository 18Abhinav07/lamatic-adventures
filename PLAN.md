# Lamatic Oracle Flow — Build Plan

## BUILD COMPLETE (2026-09-07)

Both flows are built in Studio and **deployed to the edge**, and
`oracle-eval-flow` has been **run end to end** over the GraphQL `executeWorkflow`
API: `threshold=4` -> `status "success"`, `{ pass: true, passed: 7, failed: 0,
total: 7, pass_rate: 1, regressions: [], per_case_results: [7 rows] }`. Full
proof + curl command in **`flows/DEMO-PROOF.md`**; final node YAML in
`flows/oracle-eval-flow.yaml` and `flows/target-flow.yaml`.

What actually shipped differs from the plan below in two ways — the plan text is
kept for the reasoning trail, these are the corrections:

- **Grader is deterministic, not an LLM judge.** The golden answers are exact
  facts (Paris, 144, Mars, 1945, ...), so `codeNode_500` "Grade Case" does
  `normalize(lowercase, strip punctuation, collapse whitespace) + pipe-split the
  accept string + substring match`. No `InstructorLLMNode`, no second model
  dependency, no average-score maths — `pass = (passed count >= threshold)`. The
  LLM-judge variant is kept as a note on an alternative design (bottom of
  `oracle-eval-flow.yaml`).
- **Models: no bundled catalogue.** This project had **zero** model providers
  connected. A **Groq** credential named `groq-1` (credentialId
  `12454f6f-f877-4bb3-9921-7c4e1acfec0e`) was added by the user. Groq via
  Lamatic exposes only **`gpt-oss-120b` / `gpt-oss-20b`**. `target-qa-flow`'s
  LLM node uses `groq/openai/gpt-oss-20b`. (Contradicts "bundled 300+ models"
  in the pricing blurb and the "Locked decisions" section below.)

### Config-tab / reference-resolution facts confirmed by building it

- Pasting a **complete** Config YAML into the per-flow Config tab's Monaco
  editor syncs **all** nodes onto the canvas, including brand-new ones. Save
  flips the top-right badge to "Pending Deployments"; **Deploy** button ->
  "Deploy Project" dialog (tick the flows) -> Deploy pushes to the edge. Studio
  then re-serialises the YAML canonically (reorders keys within
  `values` / `allConfigs`, reorders nodes).
- **codeNode reference resolution** (proven with a temporary `_debug` dump node):
  - **Bare** `{{nodeId.output.field}}` (not quoted/backticked) **is** resolved —
    the JSON value is spliced in literally. Safe idiom for maybe-undefined refs:
    `const x = [{{ref}}][0] || {};`
  - **Backtick-wrapped** `` `{{nodeId.output.field}}` `` is **not** dereferenced —
    it becomes the literal path string `"workflow.nodeId.output.field"`. This was
    the v1/v2 bug (Grade Case read backtick refs, got path strings).
  - References **more than one level past `.output`** do not resolve
    (`{{forLoopNode_193.output.currentValue.input}}` fails). Fix: a `codeNode`
    ("Case Input") that re-exposes `currentValue.{input,accept}` one level down.
- `forLoopEndNode.output.loopOutput` is an **array**; each element is an object
  **keyed by nodeId**: `{ codeNode_050:{...}, flowNode_379:{...},
  codeNode_500:{output:{question,accept,got,pass}, ...} }`. Aggregate unwraps
  `r.codeNode_500.output`.
- `flowNode.values.requestInput` is a string, but a `{{...}}` inside it **does**
  render at execution time (target flow received the real questions).
- Trigger payload: `[{{triggerNode_1.output}}][0].threshold` == the value sent in
  the GraphQL `payload`. `threshold` must be typed `Int` in the query; a `Float`
  or string is rejected.
- GraphQL API is execute-only: `executeWorkflow` / `checkStatus`, introspection
  disabled, and `executeWorkflow` returns a `WorkflowResponse` so the selection
  set `{ result status }` is mandatory.

## Verification status

Updated 2026-09-07 from a **live read of `lamatic.ai/docs`** (the prior
session was egress-blocked and worked from search snippets). Node types and
value-field names below are quoted from the docs' own YAML examples. What a
live docs read still can't verify — flagged inline in `flows/` — is settled
only by building in Studio: exact loop-child YAML nesting, the field name of
the question text inside a retrieved memory record, and Code node input/output
mapping. Build node-by-node in Studio and run Flow Test after each add.

## Confirmed facts (live docs read)

### There is no API to create or update flows
- The GraphQL API is **execute-only**: one operation, `executeWorkflow(workflowId, payload)`,
  plus a status poll for async. Auth: `Authorization: Bearer <API_KEY>` and
  `x-project-id: <PROJECT_ID>` against a per-project endpoint.
  (`docs/interface/graphql`, `docs/api-overview`)
- Flow authoring is **Studio-only** — visual editor or the per-flow **Config
  tab** (YAML, "flexibility similar to GitHub Actions"; edits sync to the
  canvas live). There is no "import a YAML file as a new flow" endpoint.
  Reuse paths are: duplicate a flow, publish/consume a Template, or export a
  flow **package** (YAML + JSON + README). GitHub/VCS branch-per-environment
  sync is a separate feature; tier not confirmed, and not needed here.
- Practical consequence: the `flows/*.yaml` files are a **structural target**
  to reconcile against each flow's Config tab in Studio, not something that
  can be pushed in.

### Free tier (Starter) covers this build
- 5 flows (need 3), 3,000 requests/mo, 1,000 memory/vector records, 5
  integrations, 3-day log history. Includes Visual Builder Studio, GraphQL API,
  webhooks, widgets. Config-tab YAML editing is part of the Studio editor — not
  tier-gated. (`lamatic.ai/pricing`)
- NOTE: the pricing page's "all 300+ models" is **not** what a fresh project
  gets — no provider is connected until you add a credential. This project uses
  a user-added **Groq** credential (`groq-1`); only `gpt-oss-120b` /
  `gpt-oss-20b` are exposed through it. See "BUILD COMPLETE" above.

### Real node schema (corrects the earlier drafts)
- **Trigger**: `nodeType: graphqlNode`, `nodeId: triggerNode_1`. Input schema
  is `values.advance_schema` as a **JSON string** (`'{"question": "string"}'`),
  not an `inputSchema` map. The response-type key is misspelled **`responeType`**
  (`realtime` | `async`) — confirmed from a live runtime dump in
  `docs/interface/graphql`, not a doc typo.
- **Response**: `nodeType: graphqlResponseNode`, `nodeId:
  responseNode_triggerNode_1`, `values.outputMapping` as a **JSON string**.
- **LLM node**: `nodeType: LLMNode`. The `flow-config` doc shows
  `systemPrompt` + `promptTemplate` + `messages: "[]"`; a *deployed* flow's
  runtime dump instead shows `prompts: [{id, role, content}, …]` + `tools: []`.
  Treat `systemPrompt`/`promptTemplate` as the Config-tab shorthand that
  compiles to `prompts[]` — CONFIRM against a real Config tab. Output text:
  `output.generatedResponse` (+ `output._meta` token counts).
- **Variable references**: `{{nodeId.output.field}}` (was `{{node-id.field}}`).
- **nodeId pattern**: `<type>_<number>` (`LLMNode_746`, `conditionNode_347`).
  Studio assigns the number; don't rely on a specific one.
- **RAG**: `nodeType: RAGNode`. `values`: `vectorDB`, `queryField`, `limit`,
  `certainty`, `systemPrompt`, `embeddingModelName`, `generativeModelName`.
  Output: `modelResponse` (string) + `references[]`. RAGNode retrieves **and**
  generates — no separate LLM node needed for a minimal target flow.
- **Generate JSON (the judge)**: `nodeType: InstructorLLMNode`. `values.schema`
  is a **JSON string**; `promptTemplate`, `systemPrompt`, `generativeModelName`,
  `attachments: "[]"`, `messages: "[]"`. Output fields under `output.<name>`.
- **Loop**: `nodeType: forLoopNode` + paired `forLoopEndNode`. `values`:
  `iterateOver` (`"list"` | `"range"`), `iteratorValue`. Current item:
  `{{forLoopNode_X.output.currentValue}}`. Collected results:
  `{{forLoopEndNode_X.output.loopOutput[i].<nodeId>.output.<field>}}`.
- **Execute Flow**: `nodeType: flowNode`. `values`: `flowId` (a **UUID** of an
  active flow, not a name) and `requestInput` (JSON string). Output:
  `flowOutput` (object).
- **Memory Add**: `nodeType: memoryNode`. `values`: `uniqueId`, `sessionId`,
  `memoryValue` (array of `{role, content}`), `memoryCollection`,
  `embeddingModelName`, `generativeModelName`. Output: `memoryActions`,
  `extractedFacts`.
- **Memory Retrieve**: `nodeType: memoryRetrieveNode`. `values`:
  `memoryCollection`, `searchQuery`, `limit`, `filters` (JSON string),
  `embeddingModelName`. Output: `memories[]` (processed) + `rawMemories[]`.
  Note: Memory Store is **semantic/conversational** — it embeds entries and
  derives `extractedFacts`; it is not a list-all KV store and does not
  guarantee verbatim readback. See the golden-set note below.
- **Variable**: `nodeType: variablesNode`. It is a **static key/value mapper
  only** — `values.mapping` is a JSON string of `{name: {type, value}}`.
  **No append / accumulate / average.** Referenced downstream as
  `variables.<name>`.
- **Code**: `nodeType: codeNode`, `values.code` = **JavaScript** (only
  language today). Input mapping and return-shape conventions are set in the
  node's config panel — not shown in docs, verify in Studio.
- **Condition**: `nodeType: conditionNode`. Complex operand/branch structure
  that **routes to `plus-node-addNode` targets** — it does not return a
  boolean into the response. Keep the real pass/fail decision in the Code
  node; use Condition only as a visual gate if wanted.

### Still no built-in eval/judge/golden-set primitive
Confirmed against the live docs + changelog: no Eval Node, Judge Node, or
golden-set regression feature. Flow Testing / Debugging / Prompt Testing /
Code Testing validate execution and output-match only — no semantic quality
scoring. The gap this experiment targets is real. (Absence of a feature in
the docs still isn't absolute proof it doesn't exist.)

## Why this approach fits Lamatic's existing primitives

1. **No new primitive required.** Loop/Batch, Execute Flow, an LLM node as
   judge, Generate JSON Node for structured scoring, Memory Store/VectorDB
   for the golden set, Condition Node for the gate — every piece already
   exists. The whole thing is "extend what is already there," not "build new
   infrastructure."
2. **It follows the existing deploy path.** A flow exports as a YAML/JSON
   package and every flow is callable over the GraphQL `executeWorkflow`
   API (branch-per-environment GitHub sync also exists). The natural
   extension of the oracle flow is a CI gate — score drops, deploy/merge
   blocks.

## Locked decisions (2026-09-07, with the user)

- **Models:** ~~Lamatic's bundled 300+ — no external provider credential.~~
  CORRECTED — no provider was connected; the user added a Groq credential
  (`groq-1`). Standardised on `groq/openai/gpt-oss-20b`, kept fixed for
  reproducibility.
- **Target flow: plain LLM Q&A**, no RAG, no vector store.
- **Golden set: inline JSON** in a `codeNode` inside the oracle flow. No seed
  flow, no Memory Store on the demo path. `seed-golden-set-flow.yaml` is kept
  only as the "productionize into Memory Store" reference.
- **Demo path = 2 flows:** `target-flow.yaml` + `oracle-eval-flow.yaml`.

## Complete initial surface — what was built

Two flows, ~7 inline golden pairs, one demo.

### Flow 1 — Target flow (system under test)
Plain LLM Q&A: `graphqlNode` trigger (`{"question":"string"}`) → `LLMNode`
(bundled model, terse system prompt) → `graphqlResponseNode`
(`{"answer": "{{LLMNode_1.output.generatedResponse}}"}`). It only needs a
fixed `question` in / `answer` out contract so the oracle can call it.

### Flow 2 — Oracle / eval flow (the core of the experiment)
1. `triggerNode` (`graphqlNode`) — `advance_schema` `'{"threshold": "number"}'`.
2. **Code Node** (`codeNode`) — returns `{ pairs: [ {input, expected_output}, … ] }`.
   The golden set as a literal. (Productionization: swap for a
   `memoryRetrieveNode` against a Memory Store collection.)
3. **Loop Node** (`forLoopNode` + `forLoopEndNode`) — iterate one golden pair
   at a time. Use Loop, not Batch, for predictable ordering; Batch is the
   obvious v2 speed-up.
4. **Execute Flow Node** (`flowNode`, inside the loop) — call Flow 1 by its
   `flowId` UUID with the golden input, capture `flowOutput`.
5. **Generate JSON Node** (`InstructorLLMNode`, inside the loop, the judge) —
   rubric prompt: given `expected_output` and `actual_output`, return
   `{score: 1-5, reasoning: string}`. The `schema` (a JSON string) forces a
   structured, parseable result instead of free text. No separate LLMNode
   needed — `InstructorLLMNode` is itself an LLM call.
6. **Loop End Node** (`forLoopEndNode`) — closes the loop; per-iteration
   outputs are collected on its `loopOutput[]`.
7. **Code Node** (`codeNode`, JS, after the loop) — aggregation lives here,
   **not** in a Variable node: `variablesNode` is a static mapper with no
   accumulate/average. The Code node reads `loopOutput[]`, computes the
   average score, builds `per_case_results`, filters `regressions`
   (`score < threshold`), and sets `pass = average >= threshold`.
8. **Condition Node** — OPTIONAL. `conditionNode` routes to branch targets;
   it does not hand a boolean to the response. Keep the decision in the Code
   node; add a Condition node only if you want the gate drawn on the canvas.
9. **Response Node** (`graphqlResponseNode`) — `outputMapping` returns
   `{pass, average_score, per_case_results, regressions}` from the Code node.

### Why inline JSON, not Memory Store
Memory Store (`memoryNode`/`memoryRetrieveNode`) is semantic/conversational —
it embeds entries and derives `extractedFacts`, and retrieval is a similarity
`searchQuery` with a `limit`, not a list-all. A regression oracle needs the
`expected_output` verbatim and *all* pairs every run, so the demo puts the
golden set as a literal array in `codeNode_270` "Golden Set". The reasoning:
inline for the demo so retrieval can't be the thing that fails; the
productionization is a `memoryRetrieveNode` (or a dedicated golden-set table)
so non-engineers can edit cases in the UI. `seed-golden-set-flow.yaml` sketches
that Memory Store version.

### Stretch (only if time remains)
Describe, don't necessarily wire up, how the oracle's fail condition could
gate a GitHub merge on the environment's mapped branch — the "deploy gate"
idea. A description is enough here; the CI integration itself was not built.

## Build order

All authoring is in Studio — there is no create-flow API, so these `.yaml`
files are reconciled against each flow's **Config tab**, not imported.

1. In Studio (project already exists: org `abhinav` / project `abhinav`):
   pick a bundled model to standardise on; find `PROJECT_ID` + GraphQL
   endpoint (Settings → API Keys / the "Connect" button on a flow); create an
   API key. No credential or vector store needed.
2. Build Flow 1 (`target-flow.yaml`) — `graphqlNode` → `LLMNode` →
   `graphqlResponseNode`. Reconcile against the Config tab, pick the model,
   Flow Test with one question, deploy. Copy its `flowId` (UUID) from the
   editor URL.
3. Build Flow 2 (`oracle-eval-flow.yaml`) node by node, pasting the `flowId`
   from step 2 into `flowNode_1`, running Flow Test after each node. Resolve
   the inline VERIFY flags as you go (loop-child nesting, Code node I/O
   mapping, exact judge output field).
4. Run the oracle flow with `{"threshold": 4}`; capture the pass/fail summary
   (screenshot or recording).
5. Optionally run it once more via the GraphQL API (`executeWorkflow`) to show
   it is CI-callable.
6. Write one paragraph: what it does, which nodes, why the gap exists, why
   it's aligned to their roadmap.

## One-paragraph summary

This experiment built a small target flow plus an oracle flow around it —
`forLoopNode` and `flowNode` to replay a golden set through the target flow,
and (in the shipped version) a deterministic Code-node grader that normalizes
and substring-matches each answer, then gates on a pass-count threshold. The
golden set is inline for the demo; a productionized version would use a
`memoryRetrieveNode` against a Memory Store collection so non-engineers could
edit cases. The motivation: Flow Testing covers deterministic execution and
output-match but nothing for semantic quality regression, and since a flow
exports as a YAML/JSON package and the whole thing is callable over the
GraphQL `executeWorkflow` API, the natural extension is running that oracle
flow as a CI gate before a deploy.
