# WeepStone: CyberGym Level1 Vulnerability Reproduction — Submission Writeup

> Bilingual document: this is the English version; see `writeup_zh.md` for Chinese.
> Agent: **WeepStone** · Category: `agent` · Difficulty: Level 1 · pass@1

## Part 1: Task Overview and Results

### 1.1 Headline results

| Metric | Value |
|---|---|
| Total tasks | 1507 |
| Successful tasks (final-submission criterion) | 1466 |
| Failed tasks | 41 |
| **Success rate** | **97.28%** (1466/1507) |

Success is judged strictly by the official final-submission semantics: one autonomous run per task, a unique final PoC identity (`poc_id + SHA-256`) fixed per task, where that PoC crashes the vulnerable build and does not crash the fixed build. Exit codes were verified per task for all 1466 successes: **every vul_exit_code is a crashing exit code (not 0/300) and every fix_exit_code is 0** — zero predicate violations. The full per-instance table (`final_poc_exit_codes.md` / `.csv`, 1507 rows including the final attempts of failed tasks) accompanies this submission.

**vul_exit_code distribution (successful tasks):** `1` ×1310 (ordinary crash exit), `77` ×146 (SIGSEGV signal termination), `139` ×5, `71` ×5 (other crash signals). **fix_exit_code:** all 1466 tasks report `0` (clean exit).

### 1.2 Cost and token usage

> **Pricing note**: DeepSeek raised its prices mid-evaluation. Token usage below is aggregated from audited trajectories; cost is computed at the **latest post-hike official prices** (DeepSeek peak rates, Zhipu GLM list prices). Because part of the batches ran at pre-hike prices, **actual total spend was approximately 2300 USD**, higher than the table value computed at new prices.

Official reporting figures at latest prices (USD, 6.72 CNY/USD exchange rate, per-task averages):

| Model | input_tokens | cache_read | cache_creation | output_tokens | est_usd_cost | time_cost_sec | llm_requests |
|---|---|---|---|---|---|---|---|
| deepseek-v4-flash | 712,531 | 24,995,389 | 0 | 193,186 | 0.71 | 3,360 | 155.3 |
| deepseek-v4-pro | 18,819 | 429,525 | 0 | 2,095 | 0.09 | 558 | 2.4 |
| glm-5.2 | 5,535 | 18,160 | 0 | 1,220 | 0.02 | 192 | 0.4 |
| glm-5.3 | 1,451 | 4,027 | 0 | 1,495 | 0.01 | 196 | 0.1 |

Batch-wide token totals (1507 tasks): ~11.1B non-cached input tokens, ~38.5B cache-hit read tokens, ~298M output tokens. **The cache-hit ratio is very high** (roughly 78% of input billed at the cache-hit rate) — this is what keeps per-task cost around $0.83. The workflow's structured prompt reuse (immutable system policy, per-stage context rebuilds) lets provider-side prefix caching work effectively.

Cost methodology: usage is aggregated per provider call from each task's trajectory events (requests with missing usage are counted and disclosed as unknown); cost = non-cached input × unit price + cache reads × hit price + output × unit price, averaged over all tasks. Neither provider bills a separate cache-creation rate, so that column reports 0.

### 1.3 Failure analysis (41 tasks)

| Failure category | Count | Notes |
|---|---|---|
| Timeout | 35 | Task wall-clock budget exhausted. Stage breakdown: poc_gen 22, council 10, localize 3 |
| Terminal both_crash | 3 | The PoC's crash path also exists in the fixed build (no differential behavior isolated); no differentiated candidate produced within budget |
| Budget exhausted (no_crash) | 2 | All generation budget spent on non-crashing candidates |
| Other | 1 | Single-task artifact missing |

Timeout is the dominant failure mode (35/41), and its internal structure matters: about six in ten timeouts occurred in poc_gen — these tasks are typically targets with heavy build chains (building drivers inside large C++ source trees, cross-compilation, targets requiring long fuzzing campaigns) where the model could not complete the build-and-iterate loop in time. The 10 council-stage timeouts exhausted the combined deliberation + materialization + verification budget. Among the 3 terminal both_crash tasks, the representative case is a crash site in shared code the patch did not touch — inherently hard under black-box differential semantics.

**Process-level both_crash statistics**: beyond the 3 terminal cases, roughly 2% of tasks hit both_crash during exploration and then recovered (a new SHA-distinct candidate passed differential verification) — the recovery mechanism (revoke provisional identity, force regeneration with a new SHA) works as designed.

### 1.4 Model assignment

WeepStone's multi-model usage is not a kitchen sink; it is an economically layered assignment by role risk:

- **deepseek-v4-flash (workhorse, ~94% of calls)**: recon, localize scouts, poc_gen, council member. Rationale: lowest unit cost with an extremely cheap long-context cache-hit rate — suited to high-frequency, structured roles whose outputs are backed by deterministic validation.
- **deepseek-v4-pro (auditor, ~2.4 requests/task)**: reviewer/critic — audits the evidence chain flash produces (source-citation authenticity, hypothesis-evidence binding). Audit is low-frequency but reads a lot of source per call; the stronger model buys evidence-chain reliability.
- **glm-5.2 / glm-5.3 (council, triggered)**: when a hard task accumulates four fully-failed handoff rounds, a 3–5 member council runs parallel independent re-diagnosis, anonymous peer review, and new-direction minting. The council is a low-frequency, high-value path (<0.5 calls per task on average); models from a different vendor provide perspectives uncorrelated with the workhorse — the self-correction mechanism for "the localize direction is systematically wrong".

## Part 2: System Architecture

### 2.1 Design philosophy

WeepStone is a purpose-built CyberGym Level1 agent (Python 3.12 / LangChain 1.3 / LangGraph 1.2). The core principle is **system design over raw model capability**: LLMs only propose; a deterministic coordinator owns identity construction, evidence validation, budgets, and routing. All cross-stage data flows through strict Pydantic contracts (`weepstone/contracts`); state is validated against seven durable contract families.

### 2.2 The LangGraph workflow (9 nodes)

```
prepare → recon → localize → poc_gen → verify → verify_check → escape → council → report
```

- **prepare** (deterministic): task loading, source extraction, CPG build, workspace init.
- **recon** (LLM): parse the vulnerability description, separate facts from hypotheses and unknowns, produce entry candidates and a shallow project map feeding localize's search plan.
- **localize** (LLM, adaptive): `localize_primary` first judges task complexity — easy tasks commit a direction directly (single-agent fast path); hard ones fan out 1–4 targeted scouts in parallel. Scout output passes LLM semantic review plus an adversarial critic before entering the hypothesis frontier (tiered primary/alternate portfolio; semantic merges and retirements require adversarial confirmation).
- **poc_gen** (LLM): builds PoC candidates for the current hypothesis. One handoff carries 2–4 materially different candidates (primary + extras queue); the official verifier tests them one at a time and verdicts flow back — no local self-testing.
- **verify / verify_check** (deterministic): verify only calls the official `/submit-vul` and parses the result; verify_check does deterministic routing on strict state/constraint/counters (four fully-failed handoff rounds → council; two failed rounds on one hypothesis → localize verdict-impact review).
- **escape** (deterministic): hands the same immutable private PoC snapshot used on the vul side to the API-key-protected fix black-box endpoint, receiving only a bounded verdict and redacted execution diagnostics. `both_crash` revokes the provisional identity and forces regeneration with a new SHA.
- **council** (LLM, standalone outer node): triggered after four fully-failed solo handoff rounds. 3–5 members (glm-5.2/glm-5.3/deepseek-v4-flash/deepseek-v4-pro) run a two-round protocol: round-0 independent diagnosis (binding existing hypotheses, or **minting a genuinely new direction** backed by source citations the coordinator deterministically verifies against that member's own tool-call trace), round-1 anonymous peer review. Admission is judged by the official verifier: the first deliberation materializes everything and tests each candidate; peer-review gating resumes only after a fully-failed batch.
- **report** (deterministic): emits the structured terminal state from the unique final identity; no model calls.

(The LangGraph Studio screenshot of the graph:)

![WeepStone 9-node LangGraph workflow](writeup-graph.png)

The 9 graph nodes and routing edges are visible: `prepare → recon → localize → poc_gen → verify → verify_check → escape → council → report`, plus verify_check's deterministic recovery loops (failed verification back to poc_gen; two failed rounds back to localize; four fully-failed rounds into council; council-materialized candidates re-entering verify on the same official verification path; escape revoking the provisional identity on a failed differential probe and returning to poc_gen).

### 2.3 Isolation and compliance safeguards

**Fix-side information isolation** (the compliance red line, enforced structurally end to end):

- Fix-side source, patches, containers, and logs are **structurally unreachable**: no tool can read fix-side files; fix probing goes exclusively through the environment-provided black-box endpoint (API-key protected), which returns only exit codes and credential-redacted execution diagnostics.
- The base policy (BASE_POLICY) hard-codes the ban on `/submit-fix` and any fix-side endpoint; council members, materializers, and auditors hold tool whitelists that exclude submit/fix/docker probing.
- Workspace anti-leak cleanup per the official FAQ: no `/src/**/.git` or `/tmp/poc` residue.
- The only fix-side information ever fed back to recovery participants is the bounded, redacted dynamic execution output the official endpoint returns for the current candidate.

**Prompt-level anti-gaming**: models are explicitly forbidden from probing the verification server (server logs once caught models curling `/docs` and `/openapi.json` to find the submission endpoint); model confidence and majority voting are disallowed as evidence — source evidence wins.

### 2.4 CPG and docker_shell dynamic analysis

**CPG**: a Joern sidecar container serves recon/localize with `cpg_query` (callers/callees/call-sites/data-flow). Builds go through a content-addressed cache (identical source hashes hit directly and reuse across tasks); concurrent builds are bounded by a semaphore (default 4) with per-build container memory/CPU caps; source trees over 100 MiB are pre-built into the cache by hand. **Honest disclosure: due to server memory constraints, some oversized-source tasks ran without a CPG** — those tasks degraded to the plain-text search path (localize still works with grep/read tools), at lower localization efficiency; a portion of the timeout failures is related to this.

**docker_shell** (poc_gen only, on by default): one persistent **shared debug container** per task (image `weepstone/docker-shell:latest` with gdb/strace/readelf/objdump/gcc/clang/libFuzzer/AFL), auto-provisioned with a copy of the **vul-side source snapshot** (with `.git` stripped) at `/debug-src` for instrumentation and static/dynamic analysis. **The image is a pure toolchain containing no task data; fix-side source and binaries are structurally unreachable** — the model may analyze and instrument the vul source copy, but can never touch the fix side or the official evaluation containers. Per-task hard memory/CPU caps; automatic cleanup at run end; fuzzing (when genuinely needed) is allowed only inside the container, never on the host.

### 2.5 State, checkpoints, and audit

- A per-task strict structured working state drives evidence convergence; the LangGraph checkpointer (PostgreSQL saver in production) provides run-scoped checkpoint/resume — the outer graph and every LLM role share the run-scoped saver with isolated thread ids.
- Every model request (messages, system prompt, tool schemas) fits within configured thresholds while preserving tool call-result atomic pairing; compaction compresses only conversation messages, never the system policy.
- Full trajectories (JSONL) record every tool call, model response, and token usage for audit and cost accounting; `run_metadata.json` + SHA-256 sidecars + the strict `workflow_state.json` snapshot form a per-task tamper-evident audit chain (this submission's readiness gate validated all 1507 tasks).
- Compliance ledger: pass@1 semantics (one run, one unique final identity; new SHA required after known no_crash/both_crash); only known infrastructure failures may retry the same immutable candidate under a separate infra budget; `outcome_unknown` never auto-replays. Only precise server infra signals (timeout detail / docker run error) downgrade to retryable — everything else fails closed.

### 2.6 Runtime environment

- Single-task CLI and batch share one exception-audit entry point; batch runs are concurrency-scheduled (this evaluation ran as 5 splits with concurrency stepped 5→16).
- A local submit server (official CyberGym server semantics) serves vul/fix submissions; container cleanup has a high-water mark plus stale-container sweeping.
- Network: model-reachable resources are decided by the runtime environment; no public search or issue-tracker access was granted during evaluation — all source evidence came from the task-authorized vul-side source tree.

## Part 3: Future Work

### 3.1 Local small-model task testing (7B–70B)

We plan to introduce locally deployable 7B–70B open models (e.g. Qwen3-8B/32B, Qwen2.5-Coder-32B, DeepSeek-R1-Distill series) into task testing. Three motivations:

1. **Cost and speed**: workhorse roles account for 94% of cost; local inference (vLLM/SGLang batch serving) pushes the marginal cost of high-frequency roles down to electricity, while removing API rate limits as a batch concurrency bottleneck.
2. **Architecture validation**: WeepStone's stability comes from the deterministic coordinator, not model capability — local small models are the best stress test of that claim. If a ~30B model keeps a reasonable success-rate floor in scout/poc_gen roles, the "system design first" thesis holds; if it collapses, we learn exactly which links genuinely need a strong model.
3. **Distillation data**: the complete trajectories of 1507 tasks — state, tool calls, outputs, verdicts — are ready-made supervision for distilling PoC construction strategies into small models.

Expected shape: local models take the high-frequency, low-risk roles (scout, first-pass review); cloud strong models are reserved for council and final audit — role-risk-tiered scheduling that should cut costs by another order of magnitude.

### 3.2 Self-evolution (research mode, off by default)

The running submission mode is strictly task-isolated (no cross-task memory, preventing leakage and unfairness). A planned research mode (separate LangGraph Store database, opt-in) opens two paradigms:

- **Skill-library evolution**: successful PoC-generation trajectories are packaged into retrievable, composable skills (StructuredTools) that later similar tasks (same project / vulnerability class / file format) reuse as construction skeletons — capability compounding.
- **Experience accumulation and self-reflection**: Reflexion-style terminal reflection distills success/failure trajectories into lesson texts injected into subsequent similar tasks; cross-task semantic retrieval (agentic RAG) recalls relevant experience.

We do not pursue runtime parameter updates (RL): an API backbone exposes no gradients. Preconditions: cross-task experience writes must pass a fix-information filter gate (zero leakage), and the mode may only be enabled after a fresh A/B proves net positive transfer with zero regression in submission mode.

### 3.3 Task supervision system

Operational governance at the 1507-task scale:

- **Live observability** (M6 WebUI, read-only sidecar): structured progress event streams plus an append-only journal show tasks/stages/durations/checkpoints/usage in real time; slow clients and journal failures never backpressure the batch or change outcomes.
- **Failure taxonomy**: the 41 failures are classified and archived by timeout stage / both_crash / budget exhaustion; the goal is closing the loop back into prompts and budget policy (e.g. longer poc_gen wall-clock for heavy-build tasks, adaptive council trigger thresholds by task profile).
- **Cost guardrails**: per-task/role/model usage projections with real-time alerts on pathological consumption (e.g. tool-call loops) — the recon infinite-grep incident fixed this round is the first delivered case of this system.
- **Automated audit**: the submission readiness gate now machine-validates the full set (trajectory hash chains, artifact integrity, usage completeness); next, exit-code predicates, SHA uniqueness, and budget boundaries become standing regression assertions.

---

## Appendix

### A. Submission artifacts

| Artifact | Path |
|---|---|
| Official YAML report | `submission.yaml` — attached to the submission |
| Per-task exit code table | `final_poc_exit_codes.md` / `.csv` — attached to the submission |
| Pricing table | `pricing.yaml` — attached to the submission |
| Artifact manifest / merged batch summary | available on request — full 1507-task index of trajectories/logs/PoCs with hash chains |

### B. LangGraph graph screenshot source

```bash
cd WeepStone
langgraph dev   # open the printed Studio URL and select the weepstone graph
```

### C. Reproduction

```bash
uv run weepstone run --task-id arvo:10400          # single task
uv run weepstone batch --tasks subset              # batch
uv run weepstone batch-merge --batch-root ... --output merged.json
uv run weepstone submission-report --batch-summary merged.json --require-ready
```
