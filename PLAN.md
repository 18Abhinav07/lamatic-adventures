# Lamatic Oracle Flow — Build Plan

## Verification status

`lamatic.ai` is egress-blocked from this environment, so nothing below comes
from a direct page fetch. It comes from targeted web searches against
Lamatic's own doc titles/snippets (`lamatic.ai/docs/...`) across several
queries. That is good enough to confirm node names and category structure
exist, but not exact YAML field names or a page-by-page read. Before
building, cross-check field names in the Flow Editor / Flow AI Assistant —
treat the YAML scaffolds in `flows/` as drafts, not verified schema.

## Confirmed facts

- **Node categories** (`docs/nodes`): Integration, AI, Logic, Data.
- **AI nodes**: LLMNode (Generate Text), Generate JSON Node, RAG Node,
  Supervisor Node (agentNode), Agent Classifier Node, Multi Modal Agent.
- **Logic nodes** (`docs/nodes/node-logic`): Code Node, Condition Node,
  Variable Node, Branch Node, Execute Flow Node, Loop Node, Batch Node,
  Wait Node, End Node.
- **Data nodes**: VectorDB Node, Memory Add Node, Memory Retrieve Node,
  Chunking Node, Extract from File Node, Hybrid Search Node, Keyword
  Search Node.
- **Testing** (`docs/tests/flow-testing`, `flow-debugging`): Flow Testing,
  Flow Debugging, Prompt Testing, Code Testing — all validate execution
  and output-match, no semantic quality scoring.
- **Benchmarks** (`docs/models/benchmark`): compares raw model performance
  across providers to help pick a model. Not a check on your own flow's
  output quality.
- **Feedback API**: monitoring/analytics, not a pre-deploy quality gate.
- **Flow Config** (`docs/flows/flow-config`): flows are defined as YAML
  (`triggerNode`, `nodes`, `responseNode`; each node has `nodeId`,
  `nodeType`, `nodeName`, `values`, `needs`). Flows export as a package —
  YAML, JSON, and a README. Each environment maps to a GitHub branch via
  their VCS integration.
- **No eval/judge feature found**: no "Eval Node," "Judge Node," golden-set
  regression testing, or LLM-as-judge anywhere in Lamatic's docs or 2026
  changelog search results. The gap the earlier analysis assumed is real,
  not fabricated. Search absence isn't absolute proof — confirm it directly
  in the interview rather than asserting it as fact.

## Why this is genuinely aligned, not just a plausible-sounding pitch

1. **No new primitive required.** Loop/Batch, Execute Flow, an LLM node as
   judge, Generate JSON Node for structured scoring, Memory Store/VectorDB
   for the golden set, Condition Node for the gate — every piece already
   exists. That's the strongest part of the pitch: it's "extend what you
   have," not "please build me infrastructure."
2. **It sits on their own roadmap seam.** Flows already export as YAML and
   sync to GitHub, one branch per environment. The natural extension of the
   oracle flow is a CI gate on that branch — score drops, merge blocks.
   That's a feature idea Lamatic can ship, not just a demo.

## Complete initial surface — build this before the interview

Two throwaway flows plus one real one, 5–10 golden pairs, one demo.

### Flow 1 — Target flow (system under test)
Small RAG Q&A flow: trigger → RAG/VectorDB retrieval → LLMNode answer →
response node. Doesn't need to be sophisticated. It only needs a fixed
input/output contract so another flow can call it.

### Flow 2 — Golden set seeding
One-off flow (or manual Memory Add Node calls) writing 5–10
`{input, expected_output}` pairs into a Memory Store collection, e.g.
`golden-set-target-flow`.

### Flow 3 — Oracle / eval flow (the actual pitch)
1. `triggerNode` — manual or webhook, "run eval."
2. **Memory Retrieve Node** — pull all records from the golden-set
   collection.
3. **Loop Node** — iterate one golden pair at a time. Use Loop, not Batch,
   for interview-day predictability; mention Batch as the obvious v2
   speed-up.
4. **Execute Flow Node** — call Flow 1 with the golden input, capture
   `actual_output`.
5. **LLMNode + Generate JSON Node (judge)** — rubric prompt: given
   `expected_output` and `actual_output`, return
   `{score: 1-5, reasoning: string}`. Generate JSON Node forces a
   structured, parseable result instead of free text.
6. **Variable Node** — accumulate scores/reasoning across loop iterations
   into a running array or average.
7. **Condition Node** — pass/fail gate, e.g. average score ≥ 4/5.
8. **Response/End Node** — return
   `{pass, average_score, per_case_results, regressions}`.

### Stretch (only if time remains)
Describe, don't necessarily wire up, how the oracle's fail condition could
gate a GitHub merge on the environment's mapped branch — the "deploy gate"
framing. Credible description is enough; you don't need to fully implement
CI integration for the interview.

## Build order

1. Sign up, create a workspace.
2. Build Flow 1 first — throwaway, exists only to be called.
3. Seed the golden set (Flow 2) — 5–10 pairs, don't over-invest.
4. Build Flow 3 node by node, running Flow Testing after each addition.
5. Run the oracle flow, capture the pass/fail summary output
   (screenshot or recording).
6. Write one paragraph: what it does, which nodes, why the gap exists,
   why it's aligned to their roadmap.

## Interview answer, one version

"I tried Lamatic and built a small target flow plus an oracle flow around
it — Loop and Execute Flow to replay a golden set through the target flow,
Generate JSON as a structured judge, Memory Store for the golden pairs,
Condition Node as the pass/fail gate. I built this because Flow Testing
covers deterministic execution and output-match but nothing for semantic
quality regression, and since flows already export as YAML and sync to
GitHub per environment, the natural extension is treating that oracle flow
as a CI gate on the branch."
