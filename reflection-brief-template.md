# Reflection Brief — Harness Engineering Capstone

**Name:Shaik Humail**
**Date:September 24, 2026**

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):Claude-3.5 Sonnet / Claude 3 Haiku
- OS / Python:Linux(Vocareum code-server) / Python 3.11.8
- Approx. API spend:$0.00 (verified using recorded response fixtures)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In `claims_intake/loop.py` inside `run_agent_loop` (lines 42–65), execution continues while `response.stop_reason == "tool_use"`. When tools are called, the loop executes the tool and appends results to conversation state. It terminates when `response.stop_reason == "end_turn"`. In our run output trace (`runs/`), the first turn called `lookup_policy` on `tool_use`, and the final turn returned `end_turn` with the terminal routing disposition.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_antipatterns.py` validates that loop control never relies on raw string pattern matching (such as checking if `"DONE"` or `"ESCALATE"` appears in response text) or arbitrary iteration caps. If raw string matching were used, any conversational mention of "done" during intermediate multi-step reasoning would prematurely terminate the intake loop before tool execution finished.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `lookup_policy` and `verify_coverage` share claim context, but `lookup_policy` requires `policy_id` while `verify_coverage` requires `claim_type` and `incident_date`. Structured tool errors return typed JSON dictionaries (`{"error": "POLICY_NOT_FOUND", "retryable": false}`) which enable programmatic branching, whereas generic error strings cause the LLM to hallucinate retry loops.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → In `summary.md`, claims processed through `claims_intake.run --all` completed within 2 to 4 turns. For example, Claim #CLM-10491 completed in 3 turns using 1,420 tokens at an estimated cost of ~$0.0042. This closely matches the README sample, with small variations due to dynamic prompt assembly tokens.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → In `budget.json`, baseline conversation tokens were 43,000, and assembled tokens were 19,400, achieving a 54.9% token reduction (exceeding the >= 50% rubric requirement). The persistent facts block (3,200 tokens) and the active conversation segment (8,000 tokens) dominate the assembled prompt. The persistent facts block must remain verbatim because customer identifiers and order IDs cannot tolerate lossy summarization.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Historical conversational turns (turns 1–18) and raw 57-field courier payloads were summarized into structured key-value progress summaries (reducing 28,000 tokens to 8,200). In contrast, the persistent facts block and the latest 8 active turns were preserved byte-exact to maintain immediate dialogue continuity and exact entity recall.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → Comparing `eval.jsonl` (answered 6/6 correctly) with `eval_control.jsonl` (where persistent facts were stripped), Questions 3 and 5 regressed from correct to failed. Without persistent facts, the model hallucinated a legacy SKU and dropped the shipping address, proving that the persistent facts block is strictly load-bearing.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → In `.claude/rules/react.md`, the YAML frontmatter defines:
```yaml
---
paths:
  - "src/components/**/*.tsx"
  - "src/pages/**/*.tsx"
---

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → In validator_output.txt, project-level configurations reside in .claude/ (committed to git for the entire monorepo team), while user-level configurations reside in ~/.claude/ (developer-specific aliases, terminal theme preferences, and private API keys).

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → In shift_output.txt, the SQL query pre-filtered 3 active defects from 180 total records in data/warm.sqlite. The indexed query is SELECT * FROM defects WHERE shift = 'C' AND status = 'open'. The model never ingests the entire history, which keeps prompt tokens bounded, predictable, and fast.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In recovery.py, a 30-minute staleness threshold governs the recovery mode (test_threshold_constant_is_30_minutes). If restart occurs at 29 minutes, the system resumes directly from hot_state.json. If restart occurs at 31 minutes, operating conditions have drifted; the system discards hot state and initializes fresh with warm database summaries.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → hot_state_size.txt shows data/hot_state.json occupies 643 bytes, well below the 5 KB limit. Keeping hot state strictly bounded ensures the orchestrator can run indefinite 8-hour shifts without suffering from token window expansion or memory leaks.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model:claims_intake/client.py handles direct Anthropic API calls, structured tool schemas, and stop_reason parsing.
    → Harness:retail_context/assemble.py and .claude/rules/react.md manage token budgeting, prompt layout, and path-scoped rules.
    → Orchestration:shift_monitor/orchestrator.py manages multi-shift SQLite persistence, hot-state serialization, and crash recovery.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Deterministic enforcement is used in System 4 (crash_recovery.py) where code evaluates timestamps and enforces recovery modes. Prompt guidance is used in System 3 (.claude/rules/react.md) to guide code styling. Deterministic code is required for safety invariants; prompt guidance is suitable for stylistic and creative tasks.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 manages intra-session context (compressing conversation turns down by 54.9% within a single dialogue). System 4 manages cross-session context (offloading logs to SQLite and maintaining hot state at 643 bytes across shifts).

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → Automated test suites guarantee edge-case coverage—such as crash recovery at 29 vs 31 minutes, or missing API fields—that never trigger during a single successful happy-path run.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In System 4, the blast radius of an errant model is constrained by read-only SQLite queries, Pydantic schema validation before state updates, and human supervisor sign-off gates.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → Direct shell pipelines initially had environment path mismatches when executing across independent systems. Resolved by running the solutions from their respective directories within their dedicated virtual environments.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → Implement atomic file writes (writing to a tempfile and renaming) for hot_state.json in System 4 to prevent state corruption during ungraceful process termination.
