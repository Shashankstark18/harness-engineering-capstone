# Harness Engineering with Claude and Claude Code — Reflection Brief

**Name:** Shashank M  
**Date:** 24/09/2026

## Environment

The four capstone systems were run from their completed `solution/` directories. System 1 used Claude Haiku 4.5 (`claude-haiku-4-5-20251001`). The runs were performed in the provided Linux environment with Python 3.13 and separate virtual environments for the systems where required.

---

## 1. How does the stop_reason-driven loop control execution?

The `claim_05_auto_collision` trace provides a concrete example. Turns 1, 2, and 3 have `stop_reason: "tool_use"` and contain tool calls, while turn 4 has `stop_reason: "end_turn"` with no tool calls. The harness therefore continues execution while Claude requests another tool action and stops when Claude returns a normal completion. This makes the model's response state, rather than an arbitrary iteration count, the control signal for the loop.

**Evidence:** `capstone-evidence/system-1/traces/claim_05_auto_collision.jsonl`

---

## 2. What anti-pattern would make the loop unsafe?

An unsafe design would use an unconditional loop such as `while True` without making the model's `stop_reason` and tool-call state the termination condition. That could cause unnecessary iterations, repeated tool execution, or runaway API usage. The System 1 implementation instead uses the response state to determine whether another tool round is required.

**Evidence:** System 1 implementation and `capstone-evidence/system-1/tests.txt` — 29 tests passed.

---

## 3. Which two tools have overlapping inputs, and why do their descriptions matter?

`classify_claim` and `route_to_adjuster` operate on the same claim-type domain (`CLAIM_TYPES`). `classify_claim` produces the claim type and confidence, while `route_to_adjuster` consumes the corresponding queue. Their schemas are intentionally different because they perform different actions, so the tool descriptions need to make the distinction explicit. The system prompt also requires the queue to match the `claim_type`, preventing routing to an unrelated queue.

**Evidence:** System 1 `claims_intake/system_prompt.py` and `tests/test_tools.py`.

---

## 4. What did one representative claim cost?

For `claim_05_auto_collision`, the final run completed in 4 turns with 0 clarifications. The run recorded 14,848 input tokens and 917 output tokens, with an estimated cost of $0.0194. Across all eight claims, the total estimated cost was $0.1509.

**Evidence:** `capstone-evidence/system-1/summary.md`

---

## 5. What token reduction did the retail context system achieve?

The final System 2 run had a baseline context size of 38,708 tokens and an assembled context size of 16,774 tokens. This represents a 56.67% reduction. The assembled context retained the active context while compressing resolved information into smaller sections.

**Evidence:** `capstone-evidence/system-2/budget.json`

---

## 6. What was summarized versus preserved?

The resolved refund section was compressed from 12,334 input tokens to 386 output tokens, and the resolved subscription section was compressed from 11,475 input tokens to 400 output tokens. The assembled context also preserved a structured `case_facts` section of 204 tokens and an `active` section of 15,789 tokens. This separates information that can be summarized from information that remains directly useful to the current conversation.

**Evidence:** `capstone-evidence/system-2/budget.json`

---

## 7. What did the evaluation and control comparison show?

The normal evaluation passed all 6 of 6 cases. In the control comparison, Q6 produced the expected failure because the control context lacked the structured case record containing the exact status token. Q1 was an unexpected pass in the control condition. The comparison therefore demonstrates that structured context can provide information that is not reliably available from the control context alone.

**Evidence:** `capstone-evidence/system-2/eval.jsonl` and `eval_control.jsonl`

---

## 8. How are path-scoped rules configured in the monorepo?

The React rule uses the following path patterns:

`src/components/**/*` and `src/pages/**/*`.

The test rule uses:

`**/*.test.tsx` and `**/*.test.ts`.

This allows rules to apply only to the files for which they are relevant rather than applying one broad instruction set to the entire repository.

**Evidence:** System 3 `.claude/rules/react.md` and `.claude/rules/tests.md`

---

## 9. What is the purpose of the forked deploy-check skill?

The `deploy-check` skill uses `context: fork` and restricts its allowed tools to `Read`, `Grep`, `Glob`, and read-only Git commands (`git status`, `git diff`, and `git log`). Forking the context isolates the skill's work, while the allowlist limits what the skill can do. In particular, the skill does not receive unrestricted write or destructive command access.

**Evidence:** System 3 `.claude/skills/deploy-check/SKILL.md`

---

## 10. What is the difference between project and user Claude Code scope?

The project-level `./CLAUDE.md` is the shared project entry point and is appropriate for team-wide instructions. The user-level `~/.claude/CLAUDE.md` contains personal preferences and is not shared with the team. The project also uses `.claude/rules/` for path-scoped rules.

**Evidence:** System 3 `.CLAUDE.md`

---

## 11. How does the shift-monitoring system use the warm database?

The fresh warm database contained 40 defect rows. The retrieval path uses a SQL query of the form:

`SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`

The recorded Shift C run returned 0 new defects and then reported the previously stored Shift C findings: 3 high and 2 medium defects on `capacitor-bank-C-7`, all from lot `2026-0430-B`, plus one low VP-4 vent squeal.

**Evidence:** `capstone-evidence/system-4/run.txt` and the warm-store implementation.

---

## 12. How does crash recovery decide whether to resume?

The recovery logic uses a 30-minute threshold. If the existing state is recent enough, the system resumes; if it is older than the threshold, it starts fresh. The threshold is explicitly represented by `STALE_RESUME_THRESHOLD_MINUTES = 30`.

**Evidence:** System 4 recovery implementation and tests.

---

## 13. What is the hot-state size constraint?

The generated `hot_state.json` from the fresh Shift C run was 643 bytes. The test suite also verifies the hot-state size behavior and rejects state containing more than 20 hashes. This keeps the hot state bounded rather than allowing it to grow indefinitely.

**Evidence:** `capstone-evidence/system-4/hot_state.json` and `tests.txt`

---

## 14. What is the relationship between the model, harness, and orchestration?

The model provides reasoning and requests tool actions. The harness controls the execution loop, validates and executes tools, and determines when another model turn is required. The orchestration layer manages the surrounding workflow, including state, context assembly, queues, recovery, and recorded/offline execution. Keeping these responsibilities separate reduces the amount of authority given directly to the model.

---

## 15. How is the system's blast radius constrained?

The systems constrain model actions through explicit tool schemas, read-only or narrow tool permissions where appropriate, structured state, and application-side control logic. For example, System 3's deploy-check skill has a narrow allowlist, while System 4 keeps durable state in controlled storage and limits hot-state size. These boundaries reduce the consequences of an incorrect model decision.

---

## 16. What contrast between systems was most useful?

System 2 demonstrates context efficiency at the token level: the final assembled context reduced 38,708 baseline tokens to 16,774 tokens, a 56.67% reduction. System 4 demonstrates bounded operational state: its fresh hot state was only 643 bytes. The two systems optimize different resources—large conversational context versus persistent operational state—but both use explicit boundaries instead of allowing state to grow without control.

---

## 17. What did the tests reveal beyond basic functionality?

The tests checked architectural properties rather than only happy-path output. System 4's tests cover SQL index usage, hot-state round trips and size limits, recovery truth-table behavior, the 30-minute threshold, resumed prompts, and fork scratchpad behavior. These tests make implementation constraints executable and provide evidence that the harness behavior is deliberate.

**Evidence:** `capstone-evidence/system-4/tests.txt` — 33 tests passed.

---

## 18. Which system had the smallest operational blast radius?

The shift-monitoring system keeps the model interaction bounded by a recorded response option, a warm SQLite store, bounded hot state, and a scratchpad. The run can therefore be reproduced without giving the model unrestricted access to external systems. The architecture separates the model's reasoning from the application's durable state and database operations.

**Evidence:** System 4 fresh run and `hot_state.json`.

---

## 19. What implementation issue did you encounter and how was it resolved?

The first System 1 execution failed before the application could make its intended API request because `anthropic==0.39.0` was incompatible with the installed `httpx` version: the client initialization raised `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'`. The environment was corrected by pinning `httpx==0.27.2` to match the expected Anthropic client behavior. After the fix, the fixture run succeeded and the full eight-claim run completed successfully with a total estimated cost of $0.1509.

**Evidence:** System 1 run history and final `summary.md`.

---

## 20. How would you reduce dependency on the live model?

I would keep the model interaction behind a narrow interface and use recorded responses for deterministic tests and offline verification. System 4 already demonstrates this pattern through its `--recorded-response` option for the Shift C run. This allows orchestration, state management, recovery, and tool behavior to be verified without requiring a live model call for every test or validation run.

**Evidence:** `capstone-evidence/system-4/run.txt` and the recorded response fixture.

---

# Final Assessment

The four systems demonstrate different harness-engineering techniques: stop-reason-driven control, context assembly and compression, scoped Claude Code configuration, and bounded multi-shift orchestration. The fresh runs produced independent evidence for the reflection: System 1 passed 29 tests, System 2 passed 30 tests, System 3 passed 35 tests with validator output `OK`, and System 4 passed 33 tests. The most important practical lesson from the capstone was that reliable AI systems require explicit boundaries around model actions, context, state, tools, and recovery rather than relying on the model alone.
