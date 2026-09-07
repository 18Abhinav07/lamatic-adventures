# oracle-eval-flow — end-to-end demo proof

**Date:** 2026-09-07
**Project:** AbhinavsOrganization694 / AbhinavsProject977
**Flow:** `oracle-eval-flow` — ID `86799705-51b6-45d7-881f-8240140eea46`
**Depends on:** `target-qa-flow` — ID `4f1f1bd3-9f6d-4634-aa46-637a6726c5a5`

Both flows are **built in Lamatic Studio and deployed to the edge** ("Project
Deployed"). `oracle-eval-flow` runs the 7-case golden set through
`target-qa-flow` (Groq `gpt-oss-20b`), grades each answer with a deterministic
normalize + substring matcher, and returns a pass/fail summary gated on a
caller-supplied `threshold`.

## How it was run

Lamatic's GraphQL API is **execute-only** (`executeWorkflow` / `checkStatus`;
no flow-definition CRUD). The flow was invoked with:

```bash
# API_KEY and PROJECT_ID are read from ./.env (git-ignored, never printed)
set -a && . ./.env && set +a && curl -s -X POST \
  "https://abhinavsorganization694-abhinavsproject977.lamatic.dev/graphql" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "x-project-id: ${PROJECT_ID}" \
  -H "Content-Type: application/json" \
  -d '{"query":"query ExecuteWorkflow($workflowId: String!, $threshold: Int) { executeWorkflow(workflowId: $workflowId, payload: { threshold: $threshold }) { result status } }","variables":{"workflowId":"86799705-51b6-45d7-881f-8240140eea46","threshold":4}}' \
  | python3 -m json.tool
```

`threshold` must be a GraphQL `Int`. `executeWorkflow` returns a
`WorkflowResponse`, so the selection set `{ result status }` is required.

## Result (threshold = 4)

```json
{
  "data": {
    "executeWorkflow": {
      "status": "success",
      "result": {
        "pass": true,
        "passed": 7,
        "failed": 0,
        "total": 7,
        "pass_rate": 1,
        "threshold": 4,
        "regressions": [],
        "per_case_results": [
          { "question": "What is the capital of France?",                    "accept": "paris",       "got": "Paris.",                                      "pass": true },
          { "question": "What is 12 times 12?",                              "accept": "144",         "got": "144.",                                        "pass": true },
          { "question": "Who wrote the play Romeo and Juliet?",              "accept": "shakespeare", "got": "William Shakespeare wrote Romeo and Juliet.",  "pass": true },
          { "question": "What is the chemical symbol for the element gold?", "accept": "au",          "got": "Au",                                          "pass": true },
          { "question": "How many continents are there on Earth?",           "accept": "seven|7",     "got": "There are seven continents on Earth.",         "pass": true },
          { "question": "Which planet is known as the Red Planet?",          "accept": "mars",        "got": "Mars.",                                       "pass": true },
          { "question": "In what year did World War 2 end?",                 "accept": "1945",        "got": "World War II ended in 1945.",                  "pass": true }
        ]
      }
    }
  }
}
```

**7 / 7 passed, 0 regressions, `pass: true` (>= threshold 4).**

The `got` strings vary slightly run to run (the target LLM is not deterministic)
but the grader — lowercase, strip punctuation, collapse whitespace, pipe-split
the `accept` string, substring match — matches every case on every run.

## Pipeline

```
API Request (trigger, threshold:Int)
  -> Golden Set    codeNode_270      7 {question, accept} pairs
  -> Loop          forLoopNode_193   over the 7 pairs
       -> Case Input    codeNode_050   expose currentValue.{input,accept}
       -> Execute Flow  flowNode_379   call target-qa-flow with the question
       -> Grade Case    codeNode_500   normalize + substring match -> {pass}
  -> Loop End      forLoopEndNode_711
  -> Aggregate     codeNode_600      count passes, compare to threshold, build table
  -> API Response  graphqlResponseNode
```

Full node YAML: `flows/oracle-eval-flow.yaml`. Target flow: `flows/target-flow.yaml`.
