# WeepStone

WeepStone is a self-built CyberGym Level 1 vulnerability-reproduction agent
(Python 3.12 / LangChain 1.3 / LangGraph 1.2). Given a vulnerability
description and the pre-patch source tree, it autonomously produces a PoC that
crashes the vulnerable build and does not crash the fixed build — under the
official final-submission semantics (one autonomous run per task, one unique
final PoC identity).

**Leaderboard result: 97.28% (1466/1507 tasks, pass@1).**

## Contents

| File | Description |
|---|---|
| [writeup.md](writeup.md) | Submission writeup (English) |
| [writeup_zh.md](writeup_zh.md) | Submission writeup (Chinese) |
| [writeup-graph.png](writeup-graph.png) | The 9-node LangGraph workflow (LangGraph Studio) |
| [weepstone-icon.png](weepstone-icon.png) | System icon |

## Headline numbers

- 1466 / 1507 tasks successful (97.28%), final-submission criterion
- All 1466 successes verified: vul_exit_code is a crashing code (never 0/300),
  fix_exit_code is 0 — zero predicate violations
- ~$0.83 per task at latest prices (high cache-hit ratio ≈78%); actual total
  spend ≈ $2300 due to a mid-evaluation price hike
- Models: deepseek-v4-flash (workhorse), deepseek-v4-pro (evidence-chain
  auditor), glm-5.2 / glm-5.3 (council members for hard tasks)

## Design principle

System design over raw model capability: LLMs only propose; a deterministic
coordinator owns identity construction, evidence validation, budgets, and
routing. Fix-side information is structurally unreachable — the only fix-side
signal is the bounded, credential-redacted verdict of the official black-box
probe.
