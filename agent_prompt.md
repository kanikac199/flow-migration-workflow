# Agent Prompt: Batch Flow Migration Orchestrator

> **This is the entry point for migrating one or more flows from euler-api-txns
> to connector-service.**
>
> You (the main agent) act as a **Batch Orchestrator**. You do NOT implement code
> yourself and you do NOT directly spawn pipeline subagents (S1-S5).
> Instead, you spawn **one Per-Flow Orchestrator subagent per flow**, sequentially.
> Each Per-Flow Orchestrator internally manages its own 5-stage pipeline
> (S1 Analysis → S2 Core Flow → S3 Connector → S4 Testing → S5 Push & PR),
> and each stage further decomposes into parallel sub-subagents where possible.

---

## Architecture Overview

```
USER: "Migrate these 12 flows: VerifyVpa, ResendOtpForWallet, ..."
  │
  ▼
BATCH ORCHESTRATOR (you — the main agent)
  │
  │  Spawns Per-Flow Orchestrators SEQUENTIALLY:
  │
  ├─► Per-Flow Orchestrator: Flow 1 (VerifyVpa / Razorpay)
  │     │
  │     ├─► S1 Analysis ────────────────────────────────────────────┐
  │     │     ├─► S1.1 Search euler-api-txns (explore)    ─┐        │
  │     │     ├─► S1.2 Search connector-service (explore)  ┘ PARALLEL│
  │     │     └─► S1 merges results → ANALYSIS REPORT               │
  │     │                                                           │
  │     ├─► S2 Core Flow Infra ─────────────────────────────────────┤
  │     │     ├─► S2.1 Read workflow docs (explore) ─┐               │
  │     │     │   (7-layer guide, type mapping, etc.) ├ PARALLEL     │
  │     │     ├─► S2.2 Read file references (explore) ┘              │
  │     │     └─► S2.3 Implement 7 layers + build + commit (general) │
  │     │                                                           │
  │     ├─► S3 Connector Integration ───────────────────────────────┤
  │     │     ├─► S3.1 Read Haskell sources: pre-calls (explore) ─┐  │
  │     │     ├─► S3.2 Read Haskell sources: target (explore)     ├ P │
  │     │     ├─► S3.3 Read existing connector code (explore)     ┘  │
  │     │     └─► S3.4 Implement + build + commit (general)          │
  │     │                                              ↕ retry S3↔S4 │
  │     ├─► S4 Testing (general) ───────────────────────────────────┤
  │     │     Build → start server → pre-call chain → test → eval   │
  │     │                                                           │
  │     └─► S5 Push & PR (general) ─────────────────────────────────┘
  │         Revert config → clippy/fmt → push → gh pr create
  │
  │   ◄── Returns: PR URL or escalation
  │   Report Flow 1 result to user
  │
  ├─► Per-Flow Orchestrator: Flow 2 (ResendOtpForWallet / Razorpay)
  │     └─► S1 → S2 → S3 ↔ S4 → S5  (same nested structure)
  │
  ├─► ... Flow 3 through Flow N (sequentially)
  │
  ▼
  Report final batch summary to user
```

### Design principles

1. **Task descriptions, not templates.** The Per-Flow Orchestrator tells each
   subagent WHAT to accomplish. Subagents read workflow docs themselves and
   decide HOW to accomplish it.

2. **Maximum parallelism within stages.** Independent reads (3 repos, workflow
   docs, Haskell sources) run as parallel sub-subagents. Dependent work
   (implementation, testing) runs sequentially.

3. **Sequential between flows.** Later flows may depend on infrastructure from
   earlier flows, and only one gRPC server can run at a time.

4. **Clean boundaries.** Each agent returns a structured report. The parent
   agent merges or passes it to the next stage.

---

## CRITICAL: How Subagents Are Spawned

**Every agent at every level MUST be invoked using the Task tool.**

### Level 1: Batch Orchestrator → Per-Flow Orchestrators (sequential)

```
Task(description="Flow 1/12: VerifyVpa", subagent_type="general", prompt="...")
# wait for result
Task(description="Flow 2/12: ResendOtpForWallet", subagent_type="general", prompt="...")
# wait for result
...
```

### Level 2: Per-Flow Orchestrator → S1-S5 (sequential stages)

```
Task(description="S1 Analysis: VerifyVpa", subagent_type="general", prompt="...")
# wait, extract ANALYSIS REPORT
Task(description="S2 Core Flow: VerifyVpa", subagent_type="general", prompt="...")
# wait, extract CORE FLOW REPORT
Task(description="S3 Connector: VerifyVpa/Razorpay", subagent_type="general", prompt="...")
# wait, extract CONNECTOR REPORT
Task(description="S4 Testing: VerifyVpa/Razorpay", subagent_type="general", prompt="...")
# wait, extract TEST REPORT → retry S3↔S4 if needed
Task(description="S5 Push & PR: VerifyVpa", subagent_type="general", prompt="...")
```

### Level 3: S1/S2/S3 → parallel sub-subagents (independent reads)

```
# Inside S1:
Task(description="S1.1 euler-api-txns", subagent_type="explore", prompt="...")    ─┐
Task(description="S1.2 connector-service", subagent_type="explore", prompt="...") ─┘ PARALLEL
# S1 merges both results into ANALYSIS REPORT
```

---

## Batch Orchestrator Behavior

### What you do:
1. Parse the migration plan (list of flows, connectors, branch names)
2. For each flow, spawn a Per-Flow Orchestrator via Task tool
3. Wait for it to complete — it returns a PR URL or an escalation
4. Report the result to the user
5. Move to the next flow
6. After all flows, report batch summary

### What you do NOT do:
- Read source code, run cargo/git/grpcurl, edit files
- Spawn S1/S2/S3/S4/S5 directly — the Per-Flow Orchestrator does that
- Handle S3↔S4 retry logic — the Per-Flow Orchestrator handles that

### Pseudocode

```
completed_branches = []  # tracks {flow, connector, branch} for each completed flow

for i, (flow, connector, branch) in enumerate(migration_plan):
    result = Task(
        description=f"Flow {i+1}/{len(migration_plan)}: {flow}",
        subagent_type="general",
        prompt=build_per_flow_prompt(
            flow, connector, branch, i+1, len(migration_plan),
            completed_branches  # ← pass prior branches so subagent knows what exists
        )
    )

    if result.status == "SUCCESS":
        completed_branches.append({"flow": flow, "connector": connector, "branch": branch})
        report(f"Flow {i+1} ({flow}): PR at {result.pr_url}")
    elif result.status == "ESCALATION":
        report(f"Flow {i+1} ({flow}): ESCALATION — {result.reason}")
        ask_user("Continue with next flow or stop?")

report_batch_summary()
```

### Batch Summary Format

```
### Batch Migration Summary

| # | Flow | Connector | Branch | Status | PR |
|---|------|-----------|--------|--------|----|
| 1 | VerifyVpa | Razorpay | feat/razorpay-verify-vpa | SUCCESS | #NNN |
| 2 | ResendOtp | Razorpay | feat/razorpay-resend-otp | SUCCESS | #NNN |
| 3 | ... | ... | ... | ESCALATION | — |

**Completed:** X/N flows
**Escalations:** Y flows
```

---

## Per-Flow Orchestrator

**Spawned by:** Batch Orchestrator
**Type:** `general`

### What it receives:
- Flow name, connector, branch name
- Flow number and total count
- Completed branches from prior flows in this batch (list of {flow, connector, branch})

### What it does:
1. Spawns S1 (Analysis) and waits for ANALYSIS REPORT
2. Spawns S2 (Core Flow Infra) if flow type doesn't exist, waits for CORE FLOW REPORT
3. Spawns S3 (Connector Integration), waits for CONNECTOR REPORT
4. Spawns S4 (Testing), waits for TEST REPORT
5. On FAIL → re-spawns S3 with error context, then S4 (retry loop)
6. On CAPABILITY_MISMATCH → switches connector, re-spawns S3 then S4
7. On PASS → spawns S5 (Push & PR), waits for PR REPORT
8. Returns final result to Batch Orchestrator

### Retry logic (handled internally):

```
connector = primary_connector
connector_list = <from ANALYSIS REPORT>
attempt = 1

while True:
    s3_result = spawn S3(connector, retry_context if attempt > 1)
    s4_result = spawn S4(connector)

    if PASS → break, proceed to S5
    elif CAPABILITY_MISMATCH → connector = next(connector_list), or ESCALATE
    elif FAIL → attempt += 1, loop to S3 with error context
```

### Per-Flow Orchestrator Prompt Template

**Copy, fill `{{PLACEHOLDERS}}`, pass as `prompt` to Task tool.**

```
You are a Per-Flow Orchestrator. Your job is to migrate ONE flow from
euler-api-txns (Haskell) to connector-service (Rust) by managing a 5-stage
pipeline. Each stage is a subagent you spawn via the Task tool.

## Your Target

- **Flow:** {{FLOW_NAME}}
- **Primary Connector:** {{CONNECTOR_NAME}}
- **Branch:** {{BRANCH_NAME}}
- **Flow Number:** {{FLOW_NUMBER}} of {{TOTAL_FLOWS}} in this batch

## Prior Batch Branches (for pre-call dependencies)

These branches contain completed flow implementations from earlier in this batch.
They are NOT yet merged to main. If a pre-call dependency for your flow was
implemented on one of these branches, do NOT re-implement it on your branch.
Instead, pass the branch name to S3 and S4 so they can handle it correctly.

{{COMPLETED_BRANCHES}}

(If empty: "None — this is the first flow in the batch.")

## Repos
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
- connector-service: /home/kanikachaudhary/Kanika/connector-service/
- Credentials: /home/kanikachaudhary/Kanika/creds.json
- Workflow docs: /home/kanikachaudhary/Kanika/flow-migration-workflow/

## CRITICAL: You are an orchestrator. You do NOT write code, run builds,
edit files, or run tests yourself. You spawn subagents via Task tool for
ALL work. You read their results, pass handoff data, and handle retry logic.

## Your Pipeline: S1 → S2 → S3 ↔ S4 → S5

Execute these 5 stages in order. Each stage is a Task tool invocation.
Pass the structured report from each stage into the next stage's prompt.

---

### STAGE 1: ANALYSIS

Spawn a `general` subagent. Its job is to analyze the flow across all 3 repos
by spawning 3 PARALLEL `explore` sub-subagents, then merging results.

**Task for S1:**

Analyze the {{FLOW_NAME}} flow for migration to connector-service.

You must spawn 2 PARALLEL explore sub-subagents to search independently:

**S1.1 — Search euler-api-txns** (explore, PARALLEL):
- Repo: /home/kanikachaudhary/Kanika/euler-api-txns/
- Search euler-x/src-generated/Gateway/CommonGateway.hs for the dispatcher
  that routes {{FLOW_NAME}} to per-gateway implementations
- For each gateway found, read its Flow.hs and document:
  function name, file:line, endpoint URL, auth method, request fields (name,
  type, required/optional), response fields (success + failure shapes),
  encoding, special handling
- Also check Endpoints.hs / Env.hs for base URLs

**S1.2 — Search connector-service** (explore, PARALLEL):
- Repo: /home/kanikachaudhary/Kanika/connector-service/
- YOU MUST ACTUALLY READ EACH FILE BELOW. Do NOT guess, speculate, or
  infer what "probably" exists. If you cannot find something, report "NOT FOUND"
  with the exact grep/read command you ran.
- Check crates/types-traits/domain_types/src/connector_flow.rs — READ this file
  and grep for a struct matching {{FLOW_NAME}}. Report the exact line or "NOT FOUND".
- Check crates/types-traits/interfaces/src/connector_types.rs — READ this file
  and grep for a trait matching {{FLOW_NAME}}V2. Report the exact line or "NOT FOUND".
- Check crates/types-traits/grpc-api-types/proto/services.proto — READ this file
  and grep for an RPC matching {{FLOW_NAME}}. Report the exact line or "NOT FOUND".
- Check crates/types-traits/grpc-api-types/proto/payment.proto — grep for request/
  response message names related to {{FLOW_NAME}}. Report exact message names or "NOT FOUND".
- Check crates/integrations/connector-integration/src/connectors/{{CONNECTOR_NAME_LOWERCASE}}/
  — READ mod.rs and grep for `ConnectorIntegrationV2<{{FLOW_NAME}}`. If found, read the
  impl block and report whether it has real methods (get_url, get_request_body,
  handle_response_v2) or is empty. Report REAL/STUB/MISSING with file:line evidence.
- Check crates/common/common_utils/src/events.rs — grep for {{FLOW_NAME}} in FlowName enum.
- Check crates/types-traits/ucs_interface_common/src/flow.rs — grep for {{FLOW_NAME}} type_id.
- For EVERY finding, include the file:line reference where you found it.
  For EVERY "NOT FOUND", include the exact command you ran.

After both sub-subagents return, MERGE their results into a single
ANALYSIS REPORT with these exact sections:

- **Flow:** name
- **Connectors that implement this flow:** [list from S1.1]
- **Flow type exists in connector-service:** YES/NO (from S1.2)
- **Per-Connector Analysis:** (from S1.1, for each connector)
  - Haskell file, function, endpoint, auth, request/response fields, special handling
- **Field Naming Map:** Haskell name → connector-service name (from S1.2)
- **Flow Infrastructure Needed:** if flow type doesn't exist (from S1.2)
  - Flow marker struct, sub-trait, proto RPC, request/response messages
- **Pre-call Dependency Chain:** trace what inputs this flow needs (IDs,
  tokens) and which prior flows produce them. For each dependency, state
  whether it exists in connector-service (real/stub/missing). Build ordered
  execution chain.
- **Connector Capability for Testing:** can {{CONNECTOR_NAME}} test this
  flow in sandbox? If not, identify fallback connector.

---

### STAGE 2: CORE FLOW INFRASTRUCTURE

**Skip if Analysis Report says "Flow type exists: YES".** Set core_flow_report
to "Skipped — flow type already exists" and proceed to S3.

Otherwise, spawn a `general` subagent. Its job is to create the flow type
infrastructure by first reading docs in parallel, then implementing.

**Task for S2:**

Create the flow type infrastructure for {{FLOW_NAME}} in connector-service.

First, spawn PARALLEL `explore` sub-subagents to read reference material:

**S2.1 — Read implementation guides** (explore, PARALLEL):
- Read /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.5_new_flow_type.md
  (the 7-layer guide — this is the primary reference)
- Read /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.3_per_flow_implementation.md
- Summarize the 7 layers and rules

**S2.2 — Read reference material** (explore, PARALLEL):
- Read /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.1_type_mapping.md
- Read /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.2_code_pattern_translation.md
- Read /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.3_file_references.md
- Summarize exact file paths for each layer and naming conventions

After the reads complete, spawn a `general` sub-subagent to implement:

**S2.3 — Implement core flow infrastructure** (general, SEQUENTIAL after S2.1 + S2.2):
- Analysis Report: {{PASTE ANALYSIS REPORT}}
- Guides summary: {{PASTE S2.1 RESULT}}
- File references: {{PASTE S2.2 RESULT}}
- Branch: {{BRANCH_NAME}}
- Create fresh branch from main
- Create all 7 layers (marker, domain types, sub-trait, proto, conversions,
  gRPC handler, default impls) for the target flow AND any pre-call flow
  types that are missing
- Update FlowName enum in events.rs and type_id mapping in flow.rs
- Run cargo clippy + fmt + build — fix until clean
- Commit: "feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} flow type infrastructure"
- Return CORE FLOW REPORT: branch, flow type created, pre-call types created,
  commit SHA, files modified, build/clippy/fmt status

RULES for S2.3:
- connector-service naming conventions, NOT euler-api-txns
- ResponseRouterData has exactly 2 type params
- Reuse PaymentsResponseData for payment-semantic flows; standalone struct
  for different domains (wallet, OTP, mandate lifecycle)
- Do NOT write connector-specific code — only flow type infrastructure
- BEFORE COMMIT: cargo clippy --all-targets --all-features && cargo +nightly fmt --all

---

### STAGE 3: CONNECTOR INTEGRATION

Spawn a `general` subagent. Its job is to implement connector-specific code
by first reading Haskell sources in parallel, then implementing.

**Task for S3:**

Implement {{CONNECTOR_NAME}} connector integration for {{FLOW_NAME}}.

First, spawn PARALLEL `explore` sub-subagents to read source code:

**S3.1 — Read Haskell sources for pre-call flows** (explore, PARALLEL):
(Skip if no pre-call dependencies in the Analysis Report)
- For each pre-call flow in the dependency chain, read the Haskell
  implementation in euler-api-txns for {{CONNECTOR_NAME}}
- Document: endpoint URL, request construction, response parsing, auth,
  headers, encoding
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/

**S3.2 — Read Haskell sources for target flow** (explore, PARALLEL):
- Read the {{CONNECTOR_NAME}} implementation of {{FLOW_NAME}} in
  euler-api-txns
- Document: endpoint URL, request construction (every field, how it's built),
  response parsing (every field, how status is determined), auth headers,
  checksum/signature algorithm, encoding, error handling

**S3.3 — Read existing connector code in connector-service** (explore, PARALLEL):
- Repo: /home/kanikachaudhary/Kanika/connector-service/
- Read the {{CONNECTOR_NAME}} module under
  crates/integrations/connector-integration/src/connectors/
- Find how existing flows are implemented (get_url, get_headers,
  get_request_body, handle_response_v2 patterns)
- Check which pre-call flows already have REAL implementations vs empty stubs
- Find create_all_prerequisites! and macro_connector_implementation! macros

After reads complete, spawn a `general` sub-subagent to implement:

**S3.4 — Implement connector integration** (general, SEQUENTIAL):
- Analysis Report: {{PASTE ANALYSIS REPORT}}
- Core Flow Report: {{PASTE CORE FLOW REPORT}}
- Haskell pre-call sources: {{PASTE S3.1 RESULT}}
- Haskell target sources: {{PASTE S3.2 RESULT}}
- Existing connector patterns: {{PASTE S3.3 RESULT}}
- If retrying: {{PASTE ERROR DETAILS FROM S4 TEST REPORT}}
- If switching connector: previous={{OLD}}, now={{NEW}}, reason={{ERROR}}

MANDATORY PRE-CALL GATE: Before writing ANY code for the target flow,
verify ALL pre-call dependencies have REAL connector implementations.
Empty stubs (impl ... {} with no methods) are NOT real. Implement all
missing pre-calls FIRST. NEVER plan to "use dummy values for testing".

Implementation tasks:
1. Implement pre-call connector integrations (if any missing/stub)
2. Implement target flow connector integration:
   - Request transformer, response transformer, get_url, get_headers,
     content type, error handling
   - Register in create_all_prerequisites! and macro_connector_implementation!
3. Update config TOMLs if new config fields needed (all 3: development,
   sandbox, production)
4. Update field-probe auth if ConnectorSpecificConfig changed
5. cargo clippy + fmt + build — fix until clean
6. Commit: "feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} connector integration"
   (or "fix(...): ..." if retry)

RULES:
- connector-service naming conventions
- ResponseRouterData has exactly 2 type params
- Do NOT modify existing working connector code
- Full authority to modify any layer needed
- ZERO TOLERANCE for dummy values

Return CONNECTOR REPORT: connector, target implemented, pre-calls implemented,
commit SHA, files modified, build/clippy/fmt status, retry info

---

### STAGE 4: TESTING

Spawn a `general` subagent. This is inherently sequential — no sub-subagents.

**Task for S4:**

Test the {{FLOW_NAME}} flow for {{CONNECTOR_NAME}} in connector-service.

- Analysis Report: {{PASTE ANALYSIS REPORT}}
- Connector Report: {{PASTE CONNECTOR REPORT}}
- connector-service: /home/kanikachaudhary/Kanika/connector-service/
- Credentials: /home/kanikachaudhary/Kanika/creds.json
- Testing strategy: /home/kanikachaudhary/Kanika/flow-migration-workflow/8_testing_and_operations/8.1_testing_strategy.md

MANDATORY PRE-TEST VALIDATION: Verify Connector Report confirms ALL pre-call
dependencies were implemented with REAL integrations. If any missing, return
FAIL immediately — never use dummy/placeholder values.

Steps (all sequential):
1. Build: cargo build -p grpc-server
2. Start: RUN_ENV=development nohup ./target/debug/grpc-server > /tmp/grpc-server.log 2>&1 &
3. Read creds.json, find {{CONNECTOR_NAME}} credentials, build x-connector-config header
4. Verify endpoint URLs against Haskell source (cross-reference Endpoints.hs/Env.hs)
   — if config/development.toml needs temp changes, record them
5. Execute pre-call dependency chain via grpcurl in order, capture output IDs/tokens
6. Test target flow using real pre-call outputs
7. Evaluate response:
   - PASS: HTTP 200 with correct business data
   - CAPABILITY_MISMATCH: connector says "not supported/enabled in test" (env limitation)
   - FAIL: any other non-200 (code bug — diagnose root cause, suggest fix)
8. Record test-only config changes (to be reverted by S5)
9. Stop server: kill $(pgrep -f grpc-server)

Return TEST REPORT: flow, connector, result (PASS/FAIL/CAPABILITY_MISMATCH),
attempt number, pre-call chain (commands + responses), target flow command,
target flow full response JSON, HTTP status, root cause diagnosis (if FAIL),
capability error (if MISMATCH), test-only config changes

---

### STAGE 5: PUSH & PR

Only spawn on PASS from S4.

Spawn a `general` subagent. This is inherently sequential.

**Task for S5:**

Finalize and raise PR for {{FLOW_NAME}} / {{CONNECTOR_NAME}}.

- Connector Report: {{PASTE CONNECTOR REPORT}}
- Test Report: {{PASTE TEST REPORT}}
- connector-service: /home/kanikachaudhary/Kanika/connector-service/
- Branch: {{BRANCH_NAME}}

Steps:
1. Revert test-only config changes (from Test Report)
2. Run cargo clippy + fmt — commit if changes produced
3. Verify commit history: git log --oneline main..HEAD
   (should show: S2 infra commit, S3 connector commit, any fix/chore commits)
4. Push: git push -u origin {{BRANCH_NAME}}
5. Create PR via gh pr create:
   - Title: feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} flow for {{CONNECTOR_NAME}}
   - Body: Summary bullets + test results section
   - Test section: grpcurl command (x-connector-config REDACTED, secrets in
     request body replaced with placeholders), FULL unredacted response JSON
   - Note any test-only config that was reverted

SECURITY: Redact x-connector-config header value entirely. Replace secrets
in request body with <PLACEHOLDER>. NEVER redact any part of response body.

Return PR REPORT: PR URL, branch, commits table, title, test status,
config reverted, pre-call flows included, files changed, lines +/-

---

### RETRY LOGIC (you handle this between S3 and S4):

```
connector = "{{CONNECTOR_NAME}}"
connector_list = <from ANALYSIS REPORT>
attempt = 1

while True:
    s3 = spawn S3(connector, retry_context if attempt > 1)
    s4 = spawn S4(connector)

    if PASS → break, proceed to S5
    elif CAPABILITY_MISMATCH →
        connector = next(connector_list)
        if none left → return ESCALATION to Batch Orchestrator
    elif FAIL →
        attempt += 1
        retry_context = s4.error_details
        → loop to S3
```

### What you return to the Batch Orchestrator:

### PER-FLOW RESULT

**Flow:** {{FLOW_NAME}}
**Status:** SUCCESS / ESCALATION
**Connector:** <final connector used>
**PR URL:** <url> (if SUCCESS)
**Branch:** {{BRANCH_NAME}}
**Escalation reason:** <reason> (if ESCALATION)
**Attempts:** <count>
**Connectors tried:** [list]
```

---

## Rules (Shared Across All Subagents)

These rules apply at every level. Any subagent that writes code must follow them.

### Branching & Commits
- **ONE BRANCH PER FLOW.** Fresh from `main`. Each branch contains ONLY the
  commits for that specific flow.
- **Pre-call dependency handling (3 cases):**
  - **Case A — Pre-call already on main:** Do nothing. It's available automatically.
  - **Case B — Pre-call on a prior batch branch (not yet merged):** Do NOT
    re-implement it. S3 skips it. S4 will temporarily merge that branch for
    build+test, then `git reset --hard` after. Only the current flow's commits
    appear on the current branch.
  - **Case C — Pre-call truly missing (nowhere):** Implement it on the current
    branch. This is the ONLY case where pre-call code goes on the current branch.
- **Separate commits:** S2 commits core flow infra, S3 commits connector code.
  Never squash into one commit.
- **BEFORE EVERY COMMIT:** `cargo clippy --all-targets --all-features` +
  `cargo +nightly fmt --all` — zero errors, zero warnings, zero formatting changes.
- Do NOT amend commits or force push unless user explicitly asks.

### Naming & Types
- **connector-service naming conventions**, NOT euler-api-txns. Before adding any
  new field/type name: grep existing .proto files and Rust domain types. Use
  whatever name already exists for that concept.
- **ResponseRouterData** has exactly 2 type params (not 5).
- **Response type reuse:** `PaymentsResponseData` for payment-semantic flows.
  Standalone struct only for fundamentally different domains (wallet, OTP, mandate).

### Implementation Authority
- Subagents have full authority to modify any layer: core flow, connector
  integration, proto, config. Do not stop at partial implementation.
- Do NOT modify existing working connector code — only add new code.
- If type mapping is ambiguous or gateway needs encryption, flag and ask user.

### Testing Integrity — ZERO TOLERANCE for Dummy Values
- Every ID, token, and reference in testing MUST come from a real grpcurl call
  to a real implemented flow. Violations:
  - Hardcoded/placeholder IDs instead of implementing pre-calls
  - Empty stubs treated as "good enough"
  - 4xx accepted when the real problem is missing pre-call implementation
  - Test declared "passed" with faked pre-call outputs
- **Iterate until HTTP 200.** The only exit conditions:
  (a) HTTP 200 with valid business data → Push & PR
  (b) ALL connectors exhausted with environment limitations → escalate to user
- **Connector fallback** on CAPABILITY_MISMATCH: auto-switch to next connector
  from the analysis report.

### Config & Security
- Revert test-only config changes before committing.
- In PRs: REDACT entire x-connector-config header, PLACEHOLDER secrets in
  request body, NEVER redact response body.

---

## Variants

### Single Flow

```
User: "Migrate the VerifyVpa flow"
Batch Orchestrator → Task(Per-Flow Orchestrator: VerifyVpa) → report
```

### Dry Run (Analysis Only)

Run only S1. Skip S2-S5.

```
User: "Analyze CheckBalanceForWallet. Do NOT write code."
Batch Orchestrator → Task(S1 Analysis only) → report to user
```

### Batch Migration (primary use case)

```
User: "Migrate these 12 flows: [table]"
Batch Orchestrator:
  → Task(Flow 1/12: VerifyVpa / Razorpay) → report
  → Task(Flow 2/12: ResendOtp / Razorpay) → report
  → ... (all 12 sequentially)
  → Batch summary
```

---

## Quick Start Examples

**Migrate one flow:**
```
Migrate the VerifyOtpForWallet flow to connector-service.

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```

**Batch migrate 12 flows:**
```
Migrate these flows to connector-service:

| # | Flow | Connector | Branch |
|---|------|-----------|--------|
| 1 | VerifyVpa | Razorpay | feat/razorpay-verify-vpa |
| 2 | ResendOtpForWallet | Razorpay | feat/razorpay-resend-otp-for-wallet |
| 3 | VerifyOtpForWallet | PhonePe | feat/phonepe-verify-otp-for-wallet |
| 4 | DelinkWallet | PhonePe | feat/phonepe-delink-wallet |
| 5 | InitiateTopup | PhonePe | feat/phonepe-initiate-topup |
| 6 | VerifyTopupWebhook | Paytm | feat/paytm-verify-topup-webhook |
| 7 | CreditToWallet | Razorpay | feat/razorpay-credit-to-wallet |
| 8 | CreateSubscription | PhonePe | feat/phonepe-create-subscription |
| 9 | MandateRevoke | Razorpay | feat/razorpay-mandate-revoke |
| 10 | CancelRecurring | Cashfree | feat/cashfree-cancel-recurring |
| 11 | UpdateMandateToken | Payu | feat/payu-update-mandate-token |
| 12 | SplitSettlement | Razorpay | feat/razorpay-split-settlement |

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```

---

## Implementation Notes for Batch Orchestrator

### Filling the Per-Flow Orchestrator Prompt

Replace these placeholders:

| Placeholder | Source |
|---|---|
| `{{FLOW_NAME}}` | Migration plan |
| `{{CONNECTOR_NAME}}` | Migration plan |
| `{{BRANCH_NAME}}` | Migration plan |
| `{{FLOW_NUMBER}}` | Sequential index |
| `{{TOTAL_FLOWS}}` | Total count |
| `{{COMPLETED_BRANCHES}}` | Accumulated list from batch orchestrator: `[{flow, connector, branch}, ...]` or "None" |

All `{{PASTE ...}}` placeholders inside S1-S5 tasks are filled by the Per-Flow
Orchestrator as it extracts handoff data between stages.

### Error Handling at Batch Level

- **ESCALATION** from a Per-Flow Orchestrator: report to user, ask whether to
  continue or stop.
- **Crash/hang**: report last known state, ask user.
- Track successes and failures for final summary.

### Sequential Execution Rationale

1. Later flows may reuse flow types from earlier flows
2. Git branches from `main` — parallel would cause conflicts
3. One gRPC server at a time for testing
4. User sees progress and can intervene

---

## Appendix A: Full Subagent Prompt Templates

> **These are the verbatim, copy-paste-ready prompts** that the Per-Flow
> Orchestrator passes to each subagent via the Task tool. Replace
> `{{PLACEHOLDERS}}` with real values before spawning.
>
> Each template is self-contained — the subagent receiving it has everything it
> needs to execute without reading this master document.

---

### TEMPLATE: S1 — Analysis Agent

```
You are S1 — the Analysis Agent for a flow migration.

## Target
- Flow: {{FLOW_NAME}}
- Primary Connector: {{CONNECTOR_NAME}}
- Branch: {{BRANCH_NAME}}

## Repos
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Your Job

Analyze {{FLOW_NAME}} across both repos by spawning 2 PARALLEL `explore`
sub-subagents, then merge their results into a single ANALYSIS REPORT.

You do NOT write code. You only read, search, and analyze.

## CRITICAL RULES FOR ALL SUB-SUBAGENTS:
- **READ FILES, DO NOT GUESS.** Every finding must come from actually reading
  a file or running a grep. If a sub-subagent cannot find something, it must
  report "NOT FOUND" with the exact search it ran.
- **INCLUDE file:line EVIDENCE.** Every claim about what exists must include
  the file path and line number where it was found.
- **DO NOT HALLUCINATE.** Do not generate plausible-sounding code/struct/trait
  names that you haven't actually seen in the codebase. Do not say "probably
  exists" or "likely named". Either you found it or you didn't.
- **DO NOT ASK QUESTIONS.** Read the files and return results. If something is
  ambiguous, report what you actually found and flag the ambiguity.

---

### Sub-subagent S1.1 — Search euler-api-txns (explore, PARALLEL)

Spawn with: Task(description="S1.1 euler-api-txns: {{FLOW_NAME}}", subagent_type="explore", prompt=<below>)

Search /home/kanikachaudhary/Kanika/euler-api-txns/ for all implementations
of {{FLOW_NAME}}:

1. Open euler-x/src-generated/Gateway/CommonGateway.hs
2. Find the dispatcher function that routes {{FLOW_NAME}} to per-gateway
   implementations. Note: flow names in Haskell are camelCase (e.g.,
   triggerOTPForWallet, verifyVpa, mandateRevoke). Search for variations.
3. For EACH gateway that implements this flow, read its Flow.hs and document:

   | Field | Value |
   |-------|-------|
   | Gateway name | e.g., RAZORPAY |
   | Haskell file | e.g., euler-x/src-generated/Gateway/Razorpay/Flow.hs |
   | Function name | e.g., razorpayVerifyVpa |
   | Line number | e.g., :412 |
   | HTTP method | GET / POST / PUT |
   | Endpoint URL | Full URL or URL pattern |
   | Auth method | Header / query param / body / basic auth |
   | Request content type | JSON / form-encoded / XML |
   | Request fields | Name, type, required/optional for each |
   | Response success shape | JSON structure with field names and types |
   | Response failure shape | Error structure |
   | Special handling | Checksums, signatures, Base64, retries, etc. |

4. Check the gateway's Endpoints.hs and Env.hs for base URLs.
5. Check the gateway's Types.hs for request/response type definitions.
6. Check the gateway's Transforms.hs for request construction logic.

Return ALL findings as structured text. Include file:line references.

---

### Sub-subagent S1.2 — Search connector-service (explore, PARALLEL)

Spawn with: Task(description="S1.2 connector-service: {{FLOW_NAME}}", subagent_type="explore", prompt=<below>)

## ╔══════════════════════════════════════════════════════════════╗
## ║  CRITICAL: READ FILES. DO NOT GUESS. DO NOT HALLUCINATE.    ║
## ╚══════════════════════════════════════════════════════════════╝

You MUST use the Read tool or Grep tool to actually read each file listed below.
DO NOT speculate about what "probably" exists. DO NOT generate plausible-sounding
content that you haven't actually read from a file. If you cannot find something,
report "NOT FOUND" with the EXACT search command/tool call you used.

Every finding MUST include `file:line` evidence (the actual line number where
you saw it). Every "NOT FOUND" MUST include what you searched for and where.

Search /home/kanikachaudhary/Kanika/connector-service/ for existing
infrastructure related to {{FLOW_NAME}}:

**1. Flow marker struct:**
- READ the file: crates/types-traits/domain_types/src/connector_flow.rs
- Grep for: `{{FLOW_NAME}}` (PascalCase, e.g., VerifyOtpForWallet)
- Report: FOUND at file:line with the exact struct definition, OR "NOT FOUND —
  grepped for '{{FLOW_NAME}}' in connector_flow.rs, no match"

**2. Sub-trait:**
- READ the file: crates/types-traits/interfaces/src/connector_types.rs
- Grep for: `{{FLOW_NAME}}` (the trait is usually named {{FLOW_NAME}}V2)
- Report: FOUND at file:line with exact trait signature, OR "NOT FOUND"

**3. Proto RPC:**
- READ the file: crates/types-traits/grpc-api-types/proto/services.proto
- Grep for: `{{FLOW_NAME}}` (RPC names are PascalCase)
- Report: FOUND at file:line with exact RPC definition including service name,
  OR "NOT FOUND"

**4. Proto messages:**
- READ the file: crates/types-traits/grpc-api-types/proto/payment.proto
- Grep for: request/response messages related to {{FLOW_NAME}}
  (e.g., `{{FLOW_NAME}}Request`, `{{FLOW_NAME}}Response`, or variations)
- Also grep for the proto message names used by the RPC (if found in step 3)
- Report: FOUND with exact message names and field lists, OR "NOT FOUND"

**5. Connector implementation for {{CONNECTOR_NAME}}:**
- READ the directory listing:
  crates/integrations/connector-integration/src/connectors/{{CONNECTOR_NAME_LOWERCASE}}/
- READ mod.rs in that directory
- Grep mod.rs for: `ConnectorIntegrationV2<{{FLOW_NAME}}`
- If found, READ the entire impl block and classify:
  - **REAL:** Has actual method bodies (get_url returns a URL, get_request_body
    builds a request, handle_response_v2 parses a response)
  - **STUB:** Impl block exists but methods are empty or unimplemented!()
  - Include the file:line range of the impl block
- If NOT found in mod.rs, also check other .rs files in the connector directory
- Report: REAL at file:line-line / STUB at file:line-line / MISSING (not found
  anywhere — include the grep commands you ran)

**6. Field naming map:**
- For common payment/wallet/mandate field names, grep the proto files and
  Rust domain types to find connector-service's naming:
  ```
  Fields to search for (grep each one):
  - mobile_number / phone / phoneNumber
  - merchant_id / merchantId
  - otp / otpCode
  - token / walletToken / mandateToken
  - amount / orderAmount
  - currency / currencyCode
  - vpa / vpaAddress
  - mandate_id / mandateId
  - subscription_id / subscriptionId
  - transaction_id / transactionId / txnId
  - order_id / orderId
  - customer_id / customerId
  - wallet_id / walletId
  ```
- Search in: proto/*.proto files, crates/types-traits/domain_types/src/
- Build a mapping table with file:line evidence for each match:

  | Haskell Name | connector-service Name | Found In (file:line) |
  |---|---|---|

**7. FlowName enum:**
- READ the file: crates/common/common_utils/src/events.rs
- Grep for: `{{FLOW_NAME}}`
- Report: FOUND at file:line with the exact enum variant, OR "NOT FOUND"

**8. type_id mapping:**
- READ the file: crates/types-traits/ucs_interface_common/src/flow.rs
- Grep for: `{{FLOW_NAME}}`
- Report: FOUND at file:line with the exact mapping, OR "NOT FOUND"

**9. Default implementations:**
- READ the file: crates/integrations/connector-integration/src/default_implementations.rs
- Grep for: `{{FLOW_NAME}}`
- Report: FOUND at file:line (shows which connectors use default/stub), OR "NOT FOUND"

## MANDATORY: Your return message MUST follow this structure:

```
### S1.2 Results: {{FLOW_NAME}} in connector-service

1. Flow marker struct: FOUND at <file:line> / NOT FOUND
   Evidence: <exact line content or grep command>

2. Sub-trait: FOUND at <file:line> / NOT FOUND
   Evidence: <exact line content or grep command>

3. Proto RPC: FOUND at <file:line> / NOT FOUND
   Evidence: <exact line content or grep command>

4. Proto messages: FOUND (<list>) / NOT FOUND
   Evidence: <exact message definitions or grep command>

5. Connector implementation ({{CONNECTOR_NAME}}): REAL / STUB / MISSING
   Evidence: <file:line range, method summary, or grep command>

6. Field naming map:
   | Haskell Name | connector-service Name | Found In (file:line) |
   |---|---|---|

7. FlowName enum: FOUND at <file:line> / NOT FOUND
   Evidence: <exact line content or grep command>

8. type_id mapping: FOUND at <file:line> / NOT FOUND
   Evidence: <exact line content or grep command>

9. Default implementations: FOUND at <file:line> / NOT FOUND
   Evidence: <exact line content or grep command>
```

DO NOT return anything else. DO NOT ask questions. DO NOT speculate.
Read the files, report what you found with evidence, and return.

---

### After both return, MERGE into ANALYSIS REPORT:

You MUST produce this exact structure:

### ANALYSIS REPORT

**Flow:** {{FLOW_NAME}}

**Connectors that implement this flow in Haskell:**
| # | Gateway | File | Function | Line |
|---|---------|------|----------|------|
(from S1.1)

**Flow type exists in connector-service:** YES / NO
(from S1.2 — marker struct, sub-trait, proto RPC all exist = YES)

**Per-Connector Analysis:**

For each connector found in S1.1:

#### <ConnectorName>
- **Source:** euler-api-txns
- **File:** <path>:<line>
- **Function:** <name>
- **Endpoint:** <method> <url>
- **Auth:** <method>
- **Content-Type:** <type>
- **Request Fields:**
  | Field | Type | Required | Notes |
  |-------|------|----------|-------|
- **Response (Success):**
  | Field | Type | Notes |
  |-------|------|-------|
- **Response (Error):** <structure>
- **Special Handling:** <checksums, signatures, retries, etc.>

**Field Naming Map (Haskell → connector-service):**
(from S1.2)
| Haskell Name | connector-service Name | Found In |
|---|---|---|

**Flow Infrastructure Needed:**
(from S1.2 — only if flow type does NOT exist)
- [ ] Flow marker struct in connector_flow.rs
- [ ] Domain request/response types in connector_types.rs
- [ ] Sub-trait in interfaces/connector_types.rs
- [ ] Proto messages in payment.proto
- [ ] Proto RPC in services.proto
- [ ] Proto-to-domain conversions in types.rs
- [ ] gRPC handler in grpc-server
- [ ] Default implementations for all connectors
- [ ] FlowName enum variant in events.rs
- [ ] type_id mapping in flow.rs

**Pre-call Dependency Chain:**

Trace what inputs {{FLOW_NAME}} needs. For each input (ID, token, reference):
| Input Needed | Produced By Flow | That Flow Exists in UCS? | Status |
|---|---|---|---|
| e.g., otp_token | TriggerOtpForWallet | YES | REAL impl for {{CONNECTOR_NAME}} |
| e.g., mandate_id | SetupMandate | YES | STUB for {{CONNECTOR_NAME}} |

**Ordered execution chain:** Flow_A → Flow_B → ... → {{FLOW_NAME}}

**Connector Capability for Testing:**
Can {{CONNECTOR_NAME}} test {{FLOW_NAME}} in sandbox/preprod?
- If YES: note any sandbox-specific behavior (synthetic values, etc.)
- If NO or UNKNOWN: identify fallback connectors from the list above
- Fallback connector priority: [list]

---
END ANALYSIS REPORT. Return this report as your final message.
```

---

### TEMPLATE: S2 — Core Flow Infrastructure Agent

```
You are S2 — the Core Flow Infrastructure Agent.

## Target
- Flow: {{FLOW_NAME}}
- Primary Connector: {{CONNECTOR_NAME}}
- Branch: {{BRANCH_NAME}}
- connector-service: /home/kanikachaudhary/Kanika/connector-service/
- Workflow docs: /home/kanikachaudhary/Kanika/flow-migration-workflow/

## Context from S1
{{PASTE FULL ANALYSIS REPORT HERE}}

## Your Job

Create the flow type infrastructure for {{FLOW_NAME}} (and any missing
pre-call flow types) in connector-service. You do this by:
1. Spawning 2 PARALLEL `explore` sub-subagents to read reference docs
2. Spawning 1 `general` sub-subagent (S2.3) to implement

---

### Sub-subagent S2.1 — Read implementation guides (explore, PARALLEL)

Spawn with: Task(description="S2.1 Read guides", subagent_type="explore", prompt=<below>)

Read these files and return a structured summary:

1. /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.5_new_flow_type.md
   — This is the 7-layer guide. Summarize each layer: what file to edit,
   what to add, exact code patterns.

2. /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.3_per_flow_implementation.md
   — Summarize the per-flow implementation steps.

Return the summary organized by layer number (1-7).

---

### Sub-subagent S2.2 — Read reference material (explore, PARALLEL)

Spawn with: Task(description="S2.2 Read references", subagent_type="explore", prompt=<below>)

Read these files and return a structured summary:

1. /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.1_type_mapping.md
   — Haskell-to-Rust type mappings.

2. /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.2_code_pattern_translation.md
   — Code pattern translation rules, especially ResponseRouterData (2 params!).

3. /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.3_file_references.md
   — Exact file paths for each component.

NOTE: The file references doc may use old backend/ paths. The repo has been
restructured — use crates/ prefix instead:
- backend/domain_types/ → crates/types-traits/domain_types/
- backend/interfaces/ → crates/types-traits/interfaces/
- backend/grpc-api-types/ → crates/types-traits/grpc-api-types/
- backend/grpc-server/ → crates/grpc-server/grpc-server/
- backend/connector-integration/ → crates/integrations/connector-integration/
- backend/composite-service/ → crates/integrations/composite-service/

Return: exact file paths (with crates/ prefix) for each of the 7 layers,
naming conventions, and the ResponseRouterData 2-param rule.

---

### Sub-subagent S2.3 — Implement core flow infrastructure (general, SEQUENTIAL)

Wait for S2.1 and S2.2 to complete, then spawn:
Task(description="S2.3 Implement: {{FLOW_NAME}} infra", subagent_type="general", prompt=<below>)

You are S2.3 — the implementation agent for core flow infrastructure.

## Target
- Flow: {{FLOW_NAME}}
- Branch: {{BRANCH_NAME}}
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Context
**Analysis Report:**
{{PASTE FULL ANALYSIS REPORT}}

**Implementation Guide Summary (from S2.1):**
{{PASTE S2.1 RESULT}}

**File References & Naming (from S2.2):**
{{PASTE S2.2 RESULT}}

## Your Job

Create ALL flow type infrastructure for {{FLOW_NAME}} and any pre-call flow
types that are missing (from the Analysis Report's "Flow Infrastructure
Needed" and "Pre-call Dependency Chain" sections).

### Step 1: Create branch
```bash
export PATH="$HOME/.cargo/bin:$PATH"
git checkout main && git pull
git checkout -b {{BRANCH_NAME}}
```

### Step 2: Implement 7 layers for EACH missing flow type

For {{FLOW_NAME}} (and each missing pre-call flow type), create:

**Layer 1 — Flow marker** (crates/types-traits/domain_types/src/connector_flow.rs):
- Add `pub struct {{FLOW_NAME}};`
- Add variant to `FlowName` enum

**Layer 2 — Domain types** (crates/types-traits/domain_types/src/connector_types.rs):
- Add `{{FLOW_NAME}}Data` request struct with fields from Analysis Report
- Add `{{FLOW_NAME}}ResponseData` response struct
- Decision: Reuse `PaymentsResponseData` if flow is payment-semantic;
  create standalone struct if different domain (wallet, OTP, mandate lifecycle)

**Layer 3 — Sub-trait** (crates/types-traits/interfaces/src/connector_types.rs):
- Add trait `{{FLOW_NAME}}V2` extending `ConnectorIntegrationV2<...>`
- Add to `ConnectorServiceTrait` bounds

**Layer 4 — Proto messages + RPC**:
- crates/types-traits/grpc-api-types/proto/payment.proto: Add request/response messages
- crates/types-traits/grpc-api-types/proto/services.proto: Add RPC to appropriate service
- Use connector-service naming conventions (grep existing protos first!)

**Layer 5 — Proto-to-domain conversions** (crates/types-traits/domain_types/src/types.rs):
- ForeignTryFrom for request: proto → domain
- ForeignTryFrom for PaymentFlowData from new request
- generate_{{flow_name}}_response function: domain → proto

**Layer 6 — gRPC handler** (crates/grpc-server/grpc-server/src/server/payments.rs or new file):
- Use implement_connector_operation! macro in utils.rs
- Add handler method to service impl
- Register service in app.rs if new service

**Layer 7 — Default implementations** (crates/integrations/connector-integration/src/default_implementations.rs):
- Add default_impl_{{flow_name}}_v2! macro
- Invoke for ALL connectors EXCEPT {{CONNECTOR_NAME}}

**Also update:**
- FlowName enum in crates/common/common_utils/src/events.rs
- type_id mapping in crates/types-traits/ucs_interface_common/src/flow.rs

### Step 3: Build and fix

```bash
cargo clippy --all-targets --all-features   # Fix ALL errors and warnings
cargo +nightly fmt --all                     # Fix ALL formatting
cargo build                                  # Must succeed
```

Iterate until all three pass cleanly.

### Step 4: Commit

```bash
git add -A
git commit -m "feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} flow type infrastructure"
```

## RULES — STRICTLY ENFORCED

1. **connector-service naming conventions**, NOT euler-api-txns. Before adding
   ANY new field/type: grep existing .proto files and Rust domain types. Use
   whatever name the codebase already uses.

2. **ResponseRouterData has exactly 2 type params** (Response, Self). NOT 5.
   Check crates/integrations/connector-integration/src/types.rs line 109.

3. **Response type reuse:** `PaymentsResponseData` for payment-semantic flows.
   Standalone struct only for fundamentally different domains.

4. **Do NOT write connector-specific code.** Only flow type infrastructure.
   Connector integration is S3's job.

5. **BEFORE COMMIT:** `cargo clippy --all-targets --all-features` AND
   `cargo +nightly fmt --all` — zero errors, zero warnings.

6. **Do NOT amend commits or force push.**

## Return CORE FLOW REPORT

Return this exact structure:

### CORE FLOW REPORT

**Branch:** {{BRANCH_NAME}}
**Flow types created:** [list]
**Pre-call flow types created:** [list] (or "None")
**Commit SHA:** <sha>
**Commit message:** <message>
**Files modified:**
| File | Change |
|------|--------|
**Build status:** PASS / FAIL (with error)
**Clippy status:** PASS / FAIL (with error)
**Fmt status:** PASS / FAIL (with changes)

---
END CORE FLOW REPORT.
```

---

### TEMPLATE: S3 — Connector Integration Agent

```
You are S3 — the Connector Integration Agent.

## Target
- Flow: {{FLOW_NAME}}
- Connector: {{CONNECTOR_NAME}}
- Branch: {{BRANCH_NAME}}
- Attempt: {{ATTEMPT_NUMBER}} (1 = first try, >1 = retry after test failure)
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Context from prior stages
**Analysis Report:**
{{PASTE FULL ANALYSIS REPORT}}

**Core Flow Report:**
{{PASTE CORE FLOW REPORT OR "Skipped — flow type already exists"}}

{{IF RETRY}}
**Previous Test Failure (from S4):**
{{PASTE S4 TEST REPORT WITH ERROR DETAILS}}
{{END IF}}

{{IF CONNECTOR SWITCH}}
**Connector Switch:**
Previous connector: {{OLD_CONNECTOR}} — failed with: {{ERROR}}
Now implementing: {{CONNECTOR_NAME}} (next in fallback list)
{{END IF}}

## Your Job

Implement the {{CONNECTOR_NAME}} connector integration for {{FLOW_NAME}} by:
1. Spawning 3 PARALLEL `explore` sub-subagents to read source code
2. Spawning 1 `general` sub-subagent (S3.4) to implement

---

### Sub-subagent S3.1 — Read Haskell pre-call sources (explore, PARALLEL)

Skip if Analysis Report shows no pre-call dependencies.

Spawn with: Task(description="S3.1 Haskell pre-calls: {{CONNECTOR_NAME}}", subagent_type="explore", prompt=<below>)

Read the Haskell source for EACH pre-call flow in the dependency chain
for {{CONNECTOR_NAME}}.

Pre-call chain from Analysis Report:
{{PASTE PRE-CALL DEPENDENCY CHAIN}}

For each pre-call flow, find the {{CONNECTOR_NAME}} implementation in:
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
  Path: euler-x/src-generated/Gateway/<GatewayName>/Flow.hs

For each pre-call flow, document:
| Field | Value |
|-------|-------|
| Flow name | |
| File:line | |
| Endpoint URL | Full URL including base |
| HTTP method | |
| Auth headers | Exact header names and how values are constructed |
| Request content type | |
| Request construction | How each field is built (transformations, defaults) |
| Response parsing | How each response field is extracted |
| Status determination | How success/failure is determined |
| Error handling | |

Return all findings as structured text.

---

### Sub-subagent S3.2 — Read Haskell target flow sources (explore, PARALLEL)

Spawn with: Task(description="S3.2 Haskell target: {{FLOW_NAME}}/{{CONNECTOR_NAME}}", subagent_type="explore", prompt=<below>)

Read the {{CONNECTOR_NAME}} implementation of {{FLOW_NAME}} in Haskell.

Search in:
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
  Path: euler-x/src-generated/Gateway/<GatewayName>/Flow.hs

Find the implementation function and document EVERYTHING:

| Field | Value |
|-------|-------|
| File:line | |
| Function name | |
| Endpoint URL | Full URL including base domain |
| HTTP method | |
| Auth method | How credentials are used (header, body, query) |
| Auth header names | Exact names (e.g., X-VERIFY, Authorization) |
| Auth value construction | How the value is built (e.g., SHA256 of payload + salt) |
| Request content type | JSON / form-encoded / XML / raw |
| Request encoding | Any Base64, URL-encoding, etc. |

**Request fields (EVERY field):**
| Field Name | Source | Type | Required | Construction Logic |
|------------|--------|------|----------|--------------------|
(How is each field value obtained? Direct from input? Computed? Default?)

**Response parsing (EVERY field):**
| Response Field | Maps To | Type | Extraction Logic |
|----------------|---------|------|-----------------|
(How is each field extracted from the raw response?)

**Status determination:**
- How is success determined? (HTTP code? Response field? Both?)
- What response values map to success vs failure?
- What are the known error codes and their meanings?

**Special handling:**
- Checksum/signature algorithm (step by step if present)
- Base64 encoding/decoding steps
- Multi-step flows (if response triggers another call)
- Retry logic
- Timeout configuration

Return all findings as structured text. Include EXACT file:line references.

---

### Sub-subagent S3.3 — Read existing connector code (explore, PARALLEL)

Spawn with: Task(description="S3.3 UCS connector: {{CONNECTOR_NAME}}", subagent_type="explore", prompt=<below>)

Read the existing {{CONNECTOR_NAME}} connector module in connector-service:
/home/kanikachaudhary/Kanika/connector-service/crates/integrations/connector-integration/src/connectors/{{CONNECTOR_NAME_LOWERCASE}}/

1. Read mod.rs — document:
   - How get_url() constructs URLs (base_url, secondary_base_url usage)
   - How get_headers() builds auth headers
   - How get_content_type() is set
   - How get_request_body() transforms domain → connector request
   - How handle_response_v2() parses connector → domain response
   - Pattern for error handling (get_error_response)

2. Read transformers.rs — document:
   - Request type definitions (what fields exist)
   - Response type definitions
   - TryFrom implementations (domain → connector, connector → domain)
   - How ResponseRouterData is used (should be 2 type params)

3. Check which flows already have REAL implementations vs empty stubs:
   - For each flow in the pre-call chain, check if it has a real
     ConnectorIntegrationV2 impl (with actual get_url, get_request_body,
     handle_response_v2 methods) or an empty stub (impl block with no methods)
   - Report: REAL / STUB / MISSING for each

4. Find these macros and document what's registered:
   - create_all_prerequisites! macro invocation
   - macro_connector_implementation! macro invocation

Return all findings as structured text.

---

### Sub-subagent S3.4 — Implement connector integration (general, SEQUENTIAL)

Wait for S3.1, S3.2, S3.3 to complete, then spawn:
Task(description="S3.4 Implement: {{FLOW_NAME}}/{{CONNECTOR_NAME}}", subagent_type="general", prompt=<below>)

You are S3.4 — the connector implementation agent.

## Target
- Flow: {{FLOW_NAME}}
- Connector: {{CONNECTOR_NAME}}
- Branch: {{BRANCH_NAME}}
- Attempt: {{ATTEMPT_NUMBER}}
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Context

**Analysis Report:**
{{PASTE FULL ANALYSIS REPORT}}

**Core Flow Report:**
{{PASTE CORE FLOW REPORT}}

**Haskell Pre-call Sources (from S3.1):**
{{PASTE S3.1 RESULT OR "No pre-calls needed"}}

**Haskell Target Flow Sources (from S3.2):**
{{PASTE S3.2 RESULT}}

**Existing Connector Patterns (from S3.3):**
{{PASTE S3.3 RESULT}}

**Prior Batch Branches (from orchestrator):**
{{COMPLETED_BRANCHES}}

{{IF RETRY}}
**Previous Test Failure — YOU MUST FIX THIS:**
{{PASTE S4 TEST REPORT}}

Root cause from S4: {{ROOT_CAUSE}}
Suggested fix from S4: {{SUGGESTED_FIX}}
{{END IF}}

## ╔══════════════════════════════════════════════════════════════╗
## ║  MANDATORY PRE-CALL GATE — READ THIS BEFORE WRITING CODE   ║
## ╚══════════════════════════════════════════════════════════════╝

Before writing ANY code for {{FLOW_NAME}}, classify EVERY pre-call dependency
from the Analysis Report into one of three cases:

Check the "Pre-call Dependency Chain" from the Analysis Report, the
"existing flow status" from S3.3, AND the "Prior Batch Branches" list:

**Case A — Pre-call already on main (has REAL impl in S3.3 results):**
  → Do nothing. It's available on your branch automatically.

**Case B — Pre-call implemented by a prior flow in this batch:**
  → Check {{COMPLETED_BRANCHES}} for a matching flow+connector.
  → Do NOT re-implement it. Do NOT cherry-pick or merge that branch.
  → Record the branch name in your Connector Report so S4 can temporarily
    merge it for testing.
  → The pre-call code does NOT go on this branch.

**Case C — Pre-call is STUB or MISSING and NOT on any prior batch branch:**
  → This is the ONLY case where you implement the pre-call on this branch.
  → Implement it fully (mod.rs + transformers.rs + registration).

NEVER plan to "use dummy values for testing later". Every pre-call must be
a working, real integration that produces real output when called via grpcurl.

Log your classification for EACH pre-call dependency in the Connector Report:
```
Pre-call: <FlowName> — Case <A/B/C>
  Reason: <on main / on branch feat/xxx / truly missing>
  Action: <skip / record branch for S4 / implement>
```

## Implementation Steps

### Step 1: Implement pre-call connector integrations (Case C only)

For each pre-call classified as **Case C** (truly missing) for {{CONNECTOR_NAME}}:

**Skip this step entirely if all pre-calls are Case A or Case B.**

1. In mod.rs: Add `impl ConnectorIntegrationV2<PreCallFlow, ...>` with:
   - get_url() — from Haskell Endpoints.hs (S3.1 data)
   - get_headers() — from Haskell Flow.hs auth pattern (S3.1 data)
   - get_content_type() — from Haskell request encoding (S3.1 data)
   - get_request_body() — transform domain request → connector request
   - handle_response_v2() — parse connector response → domain response

2. In transformers.rs: Add:
   - Connector request struct + TryFrom<domain → connector>
   - Connector response struct + TryFrom<connector → domain>
   - Use ResponseRouterData with 2 type params

3. Register in create_all_prerequisites! and macro_connector_implementation!

4. Remove {{CONNECTOR_NAME}} from the default_impl macro for this flow
   (in default_implementations.rs)

### Step 2: Implement target flow connector integration

Same pattern as Step 1, but for {{FLOW_NAME}}:

1. **mod.rs** — Add ConnectorIntegrationV2 impl:
   - get_url(): Construct from base_url or secondary_base_url + endpoint path
     (from S3.2 Haskell endpoint data). Match the existing connector's URL
     pattern from S3.3.
   - get_headers(): Build auth headers exactly as Haskell does (from S3.2).
     Follow existing header patterns from S3.3.
   - get_content_type(): Match Haskell's request encoding
   - get_request_body(): Transform domain → connector request
   - handle_response_v2(): Parse connector → domain response
     Map response statuses: Haskell success criteria → AttemptStatus enum

2. **transformers.rs** — Add types and conversions:
   - Connector-specific request struct with serde attributes matching the
     connector's external API field names (use #[serde(rename = "...")])
   - Connector-specific response struct
   - TryFrom<domain → connector request>: map every field per S3.2 logic
   - TryFrom<connector response → domain>: map every field, determine status

3. **Register** in create_all_prerequisites! and macro_connector_implementation!

4. **Remove** {{CONNECTOR_NAME}} from default_impl macro for {{FLOW_NAME}}
   in default_implementations.rs

### Step 3: Config updates (if needed)

If the connector needs new config fields (e.g., secondary_base_url):
- Update crates/types-traits/domain_types/src/router_data.rs (ConnectorSpecificConfig)
- Update config/development.toml, config/sandbox.toml, config/production.toml
- Update crates/internal/field-probe/src/auth.rs if ConnectorSpecificConfig changed

### Step 4: Build and fix

```bash
export PATH="$HOME/.cargo/bin:$PATH"
cargo clippy --all-targets --all-features   # Fix ALL errors and warnings
cargo +nightly fmt --all                     # Fix ALL formatting
cargo build                                  # Must succeed
```

Iterate until all three pass cleanly. Do NOT proceed with errors.

### Step 5: Commit

```bash
git add -A
git commit -m "feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} connector integration"
```

If this is a RETRY (attempt > 1):
```bash
git commit -m "fix({{CONNECTOR_NAME}}): fix {{FLOW_NAME}} — {{ONE_LINE_FIX_DESCRIPTION}}"
```

## RULES — STRICTLY ENFORCED

1. **connector-service naming conventions**, NOT euler-api-txns. Grep existing
   code before adding ANY new name.
2. **ResponseRouterData has exactly 2 type params.**
3. **Do NOT modify existing working connector code** — only ADD new code.
4. **ZERO TOLERANCE for dummy values** — every field must map to real data.
5. **BEFORE COMMIT:** `cargo clippy --all-targets --all-features` AND
   `cargo +nightly fmt --all`.
6. **Do NOT amend commits or force push.**
7. Follow the EXISTING patterns from S3.3 for this connector. Don't invent
   new patterns when the connector already has established conventions.

## Return CONNECTOR REPORT

Return this exact structure:

### CONNECTOR REPORT

**Connector:** {{CONNECTOR_NAME}}
**Flow:** {{FLOW_NAME}}
**Attempt:** {{ATTEMPT_NUMBER}}
**Branch:** {{BRANCH_NAME}}

**Pre-call implementations added/fixed:**
| Pre-call Flow | Status Before | Status After | Files Modified |
|---|---|---|---|
(or "No pre-calls needed")

**Target flow implementation:**
| Component | File | Change |
|---|---|---|
| get_url | mod.rs | <endpoint URL used> |
| get_headers | mod.rs | <auth pattern> |
| get_request_body | mod.rs + transformers.rs | <field count, encoding> |
| handle_response_v2 | mod.rs + transformers.rs | <status mapping> |
| config | <files> | <fields added> |

**Commit SHA:** <sha>
**Commit message:** <message>
**Files modified:** [list with change summary]
**Build status:** PASS / FAIL
**Clippy status:** PASS / FAIL
**Fmt status:** PASS / FAIL

{{IF RETRY}}
**Fix applied:** <what was wrong and what was changed>
**Root cause confirmed:** YES / NO
{{END IF}}

---
END CONNECTOR REPORT.
```

---

### TEMPLATE: S4 — Testing Agent

```
You are S4 — the Testing Agent.

## Target
- Flow: {{FLOW_NAME}}
- Connector: {{CONNECTOR_NAME}}
- Branch: {{BRANCH_NAME}}
- Attempt: {{ATTEMPT_NUMBER}}
- connector-service: /home/kanikachaudhary/Kanika/connector-service/
- Credentials: /home/kanikachaudhary/Kanika/creds.json
- Testing guide: /home/kanikachaudhary/Kanika/flow-migration-workflow/8_testing_and_operations/8.1_testing_strategy.md

## Context from prior stages

**Analysis Report:**
{{PASTE FULL ANALYSIS REPORT}}

**Connector Report:**
{{PASTE CONNECTOR REPORT}}

**Prior Batch Branches (from orchestrator):**
{{COMPLETED_BRANCHES}}

## ╔══════════════════════════════════════════════════════════════╗
## ║  MANDATORY PRE-TEST VALIDATION                              ║
## ╚══════════════════════════════════════════════════════════════╝

Before running ANY test, verify the Connector Report confirms:
1. ALL pre-call dependencies are accounted for:
   - Case A (on main): available automatically
   - Case B (on prior batch branch): you will temporarily merge in Step 0
   - Case C (implemented on this branch): already here
2. Build, clippy, and fmt all PASS
3. The target flow's connector integration is complete

If ANY of these are false, return FAIL immediately with:
- result: FAIL
- root_cause: "Pre-test validation failed: <specific issue>"
- suggested_fix: "<what S3 needs to fix>"

Do NOT proceed to build/test with known missing implementations.

## Testing Steps (ALL sequential — no sub-subagents)

### Step 0: Temporarily incorporate prior batch branches (Case B only)

Check the Connector Report for any pre-call dependencies classified as Case B
(on a prior batch branch, not yet merged to main).

If there are Case B pre-calls:

```bash
# Record current HEAD so we can reset later
CLEAN_HEAD=$(git rev-parse HEAD)

# For each Case B pre-call branch:
git merge --no-ff --no-edit <prior_branch_name>
# Example: git merge --no-ff --no-edit feat/phonepe-trigger-otp
```

Record which branches were merged. You MUST undo this at the end of testing
(after Step 9) regardless of whether tests pass or fail:

```bash
# After ALL testing is complete (pass or fail):
git reset --hard $CLEAN_HEAD
```

**CRITICAL:** This temporary merge is ONLY for build+test. The merged code
must NOT appear in the final branch. Step 9 (cleanup) will verify this.

If there are no Case B pre-calls, skip this step entirely.

### Step 1: Build the server

```bash
export PATH="$HOME/.cargo/bin:$PATH"
cargo build -p grpc-server
```

If build fails, return FAIL with the build error.

### Step 2: Start the server

```bash
RUN_ENV=development nohup ./target/debug/grpc-server > /tmp/grpc-server.log 2>&1 &
SERVER_PID=$!
echo "Server PID: $SERVER_PID"
sleep 5
ps -p $SERVER_PID -o pid,command || { echo "FAILED TO START"; cat /tmp/grpc-server.log; }
```

If server fails to start, return FAIL with the log contents.

### Step 3: Build x-connector-config header

Read /home/kanikachaudhary/Kanika/creds.json and find the entry for
{{CONNECTOR_NAME}}.

Construct the header in serde JSON format:
```
{"config":{"<ConnectorVariant>":{<fields from creds.json>}}}
```

Rules:
- Variant name is PascalCase and matches the Rust enum variant EXACTLY
  (e.g., Phonepe, Razorpay, Payu, Cashfree, Paytm)
- Secret<String> fields are plain strings (NOT {"value":"..."})
- The outer key is always "config"

To find the exact variant name:
```bash
grep -n "ConnectorSpecificConfig" crates/types-traits/domain_types/src/router_data.rs | head -20
```

### Step 4: HARD GUARDRAIL — Verify endpoints against Haskell source

## ╔══════════════════════════════════════════════════════════════╗
## ║  DO NOT SKIP THIS STEP. DO NOT PROCEED WITHOUT IT.          ║
## ╚══════════════════════════════════════════════════════════════╝

Before making ANY grpcurl call, you MUST cross-reference EVERY endpoint
(pre-calls AND target flow) against the Haskell codebase.

**For each flow you will test (pre-calls + target):**

1. **Read the Haskell endpoint directly** — do NOT rely on the Analysis Report alone.
   Open the actual Haskell source file and find the endpoint URL:
   ```bash
   # Example for PhonePe:
   grep -n "url\|endpoint\|baseUrl\|path" /home/kanikachaudhary/Kanika/euler-api-txns/euler-x/src-generated/Gateway/<GatewayName>/Endpoints.hs | head -30
   grep -n "url\|endpoint\|path" /home/kanikachaudhary/Kanika/euler-api-txns/euler-x/src-generated/Gateway/<GatewayName>/Flow.hs | head -30
   ```

2. **Read the connector-service get_url()** for the same flow:
   ```bash
   grep -A 10 "fn get_url" /home/kanikachaudhary/Kanika/connector-service/crates/integrations/connector-integration/src/connectors/<connector>/mod.rs
   ```

3. **Compare:**
   - Does the PATH match? (e.g., `/apis/hermes/v3/merchant/otp/send` vs `/apis/pg/v1/otp/send`)
   - Does the base URL in config/development.toml match the connector's preprod/sandbox domain?
   - Are there environment-specific URL patterns? (UAT vs preprod vs sandbox)

4. **Record the comparison in a table:**
   ```
   | Flow | Haskell Endpoint (file:line) | UCS get_url() (file:line) | Base URL (config) | MATCH? |
   |------|----------------------------|--------------------------|-------------------|--------|
   ```

5. **If endpoints DON'T match:**
   - Fix get_url() in the connector code, rebuild, and re-commit BEFORE testing
   - Or if the base URL in config is wrong, update config/development.toml temporarily

6. **If base URL seems wrong for the test environment:**
   - Check if Haskell uses a different preprod URL than what's in config
   - Try the Haskell preprod URL temporarily

**DO NOT proceed to Step 5 until EVERY endpoint is verified and matches.**

### Step 5: Execute pre-call dependency chain

For each pre-call flow in the dependency chain (from Analysis Report),
execute in order:

```bash
grpcurl -plaintext \
  -H 'x-connector-config: <JSON from Step 3>' \
  -H 'x-merchant-id: <merchant_id from creds>' \
  -H 'x-tenant-id: default' \
  -H 'x-request-id: test_precall_{{FLOW_NAME}}_<N>' \
  -H 'x-connector-request-reference-id: test_ref_precall_<N>' \
  -d '<request body in protobuf JSON format>' \
  localhost:8000 types.<ServiceName>/<PreCallRpcName>
```

For each pre-call:
- Record the FULL command and FULL response
- Extract output IDs/tokens needed by the next call
- If a pre-call fails, this is a FAIL for the entire test

**Protobuf JSON rules:**
- SecretString fields use `{"value":"..."}` format (this is proto, not serde)
- Enum fields are string names
- Standard protobuf JSON encoding

### Step 6: Test the target flow

Using real outputs from Step 5 as inputs:

```bash
grpcurl -plaintext \
  -H 'x-connector-config: <JSON from Step 3>' \
  -H 'x-merchant-id: <merchant_id from creds>' \
  -H 'x-tenant-id: default' \
  -H 'x-request-id: test_{{FLOW_NAME}}_{{ATTEMPT_NUMBER}}' \
  -H 'x-connector-request-reference-id: test_ref_{{FLOW_NAME}}_{{ATTEMPT_NUMBER}}' \
  -d '<request body using real pre-call outputs>' \
  localhost:8000 types.<ServiceName>/{{FLOW_NAME}}
```

Record the FULL command and FULL response (including rawConnectorRequest,
rawConnectorResponse, all fields).

### Step 7: Evaluate response

Evaluate the response against these criteria:

**PASS** — ALL of the following:
- HTTP 200 from gRPC (no gRPC error code)
- The connector returned a well-formed response
- Business data is present and correct (e.g., status, IDs, tokens)
- rawConnectorRequest shows correct URL, headers, body
- rawConnectorResponse shows the connector processed the request
- Note: Sandbox may return synthetic values (e.g., PhonePe returns
  "MERCHANT123", "OTP6789") — this is expected and counts as PASS

**CAPABILITY_MISMATCH** — VERY NARROW definition, ALL of these must be true:
- The connector explicitly says this feature is NOT SUPPORTED or NOT AVAILABLE
  (e.g., "recurring not supported", "feature not available in test mode",
  "subscription APIs not enabled for this merchant")
- The endpoint URL is VERIFIED CORRECT (matches Haskell exactly — from Step 4)
- The credentials are VERIFIED CORRECT (same merchant/keys as other working flows)
- The request DID reach the connector's business logic (not a routing/auth error)
- You have ALREADY attempted the fixes in Step 7a below and none worked
- This is an inherent environment limitation, NOT fixable by changing URL/creds/body

**FAIL** — ANY of these:
- gRPC error (unknown service, proto mismatch) → registration bug
- **"Api Mapping Not Found"** → WRONG ENDPOINT or WRONG MERCHANT CONFIGURATION.
  This is NEVER acceptable as a pass. It means either:
  (a) The URL path is wrong (compare against Haskell source EXACTLY), or
  (b) The merchant credentials don't have this API enabled, or
  (c) The preprod/sandbox base URL is wrong
- **"Bad Request"** from connector → malformed request body, wrong encoding,
  missing required fields — compare field-by-field against Haskell
- **"Invalid merchant" / "Authentication failed" / credential errors** →
  wrong config format, wrong keys, or wrong merchant ID
- Connection error → code bug (panic, timeout, parse failure)
- 4xx/5xx that indicates our request was malformed → code bug
- Any non-200 where pre-calls were available but skipped → INVALID test
- Any non-200 where dummy values were used → INVALID test

### Step 7a: MANDATORY fix attempts before returning FAIL

If Step 7 result is FAIL, you MUST attempt fixes before returning to the
orchestrator. DO NOT return FAIL after the first attempt without trying these:

**Fix attempt checklist (try ALL applicable, in order):**

1. **Endpoint mismatch fix:** Re-read the Haskell source (not the Analysis Report —
   the ACTUAL file). Compare the full URL path character-by-character against
   what rawConnectorRequest shows. If different:
   - Fix get_url() in the connector code
   - Rebuild: `cargo build -p grpc-server`
   - Restart server and re-test
   - Record: "Fix attempt 1: Changed endpoint from X to Y"

2. **Base URL fix:** Check if the connector's base_url in config/development.toml
   is correct for the test environment. Compare against what Haskell uses for
   preprod/UAT. If different:
   - Update config/development.toml temporarily
   - Restart server and re-test
   - Record: "Fix attempt 2: Changed base_url from X to Y"

3. **Credential fix:** If the error suggests merchant/auth issues:
   - Re-read /home/kanikachaudhary/Kanika/creds.json
   - Verify the variant name matches EXACTLY (e.g., "Phonepe" not "PhonePe")
   - Verify all required fields are present and correctly formatted
   - Check if there are MULTIPLE credential entries (different merchants) —
     try each one
   - Record: "Fix attempt 3: Changed creds from X to Y"

4. **Request body fix:** If the error suggests malformed request:
   - Compare every field name and type against the Haskell request builder
   - Check encoding (JSON vs form-encoded)
   - Check required vs optional fields
   - Fix transformers.rs if needed, rebuild, re-test
   - Record: "Fix attempt 4: Fixed request body field X"

5. **Alternative endpoint paths:** Some connectors have multiple URL patterns
   for the same logical operation (v1 vs v2 vs v3, different path prefixes).
   Check the Haskell source for ANY alternate paths. Try each one.

**Only after exhausting ALL applicable fixes above**, return FAIL with the
full diagnosis. The test report must include what fixes were attempted:

```
Fix Attempts:
| # | What was tried | Result |
|---|---------------|--------|
| 1 | Changed endpoint path to /v2/... | Still 400 |
| 2 | Changed base_url to preprod.example.com | Still 400 |
| 3 | Tried alternate merchant creds | Still auth error |
```

If NONE of the fixes work and the error is clearly "this API is not enabled
for this merchant in the test environment" (NOT a code bug), then and ONLY
then classify as CAPABILITY_MISMATCH instead of FAIL.

### Step 8: Record test-only config changes

List any changes made to config files for testing:
| File | Field | Original Value | Test Value | Must Revert |
|------|-------|---------------|------------|-------------|

### Step 9: Stop the server

```bash
kill $(pgrep -f grpc-server) 2>/dev/null || true
```

### Step 10: Undo temporary merges from Step 0

If you merged any prior batch branches in Step 0, undo them NOW:

```bash
# Reset to the clean HEAD recorded in Step 0
git reset --hard $CLEAN_HEAD
```

Verify the branch is clean — `git log --oneline main..HEAD` should show
ONLY commits belonging to {{FLOW_NAME}}, with no commits from other flows.

If you did NOT merge any branches in Step 0, skip this step.

## Return TEST REPORT

Return this exact structure:

### TEST REPORT

**Flow:** {{FLOW_NAME}}
**Connector:** {{CONNECTOR_NAME}}
**Attempt:** {{ATTEMPT_NUMBER}}
**Branch:** {{BRANCH_NAME}}

**Result:** PASS / FAIL / CAPABILITY_MISMATCH

**Temporary Branch Merges (Step 0):**
- Branches merged: [list, or "None"]
- Reset completed: YES / NO

**Pre-call Chain Execution:**
| # | Flow | RPC | Command | Response (summary) | Output Captured |
|---|------|-----|---------|--------------------|-----------------|
(or "No pre-calls needed")

**Target Flow Test:**

**grpcurl command:**
```bash
<full command with all headers and body>
```

**Full response JSON:**
```json
<COMPLETE untruncated response>
```

**HTTP status:** <status>

{{IF PASS}}
**Endpoint Verification (Step 4):**
| Flow | Haskell Endpoint (file:line) | UCS get_url() (file:line) | Base URL (config) | MATCH? |
|------|----------------------------|--------------------------|-------------------|--------|

**Validation:**
- URL correct: YES (verified against Haskell in Step 4)
- Headers correct: YES
- Request body correct: YES
- Response parsed correctly: YES
- Business data present: YES
- Sandbox synthetic values: <list any, or "None">
{{END IF}}

{{IF FAIL}}
**Endpoint Verification (Step 4):**
| Flow | Haskell Endpoint (file:line) | UCS get_url() (file:line) | Base URL (config) | MATCH? |
|------|----------------------------|--------------------------|-------------------|--------|

**Root Cause Diagnosis:**
- Symptom: <what went wrong>
- rawConnectorRequest URL: <URL used>
- rawConnectorRequest shows: <summary>
- rawConnectorResponse shows: <summary>
- Haskell comparison: <what's different>
- Root cause: <specific diagnosis>
- Suggested fix: <actionable fix for S3>

**Fix Attempts (Step 7a):**
| # | What was tried | Result |
|---|---------------|--------|
| 1 | <description> | <outcome> |
(Must have at least 1 entry. If no fixes were applicable, explain why.)
{{END IF}}

{{IF CAPABILITY_MISMATCH}}
**Capability Error:**
- Connector response: <exact error message>
- Endpoint verified correct: YES (must be YES to classify as CAPABILITY_MISMATCH)
- Credentials verified correct: YES (must be YES)
- Fix attempts exhausted: YES (must be YES — list attempts from Step 7a)
- This is an environment limitation, not a code bug
- Recommendation: Try next connector from analysis report
{{END IF}}

**Test-only Config Changes:**
| File | Field | Original | Test Value | Reverted? |
|------|-------|----------|------------|-----------|
(or "None")

**Server Log Excerpts:** <any relevant errors from /tmp/grpc-server.log>

---
END TEST REPORT.
```

---

### TEMPLATE: S5 — Push & PR Agent

```
You are S5 — the Push & PR Agent.

## Target
- Flow: {{FLOW_NAME}}
- Connector: {{CONNECTOR_NAME}}
- Branch: {{BRANCH_NAME}}
- connector-service: /home/kanikachaudhary/Kanika/connector-service/
- Git remote: git@github.com:juspay/connector-service.git

## Context from prior stages

**Connector Report:**
{{PASTE CONNECTOR REPORT}}

**Test Report:**
{{PASTE TEST REPORT}}

## Your Job

Finalize the branch, push, and create a pull request. This is sequential —
no sub-subagents needed.

### Step 1: Revert test-only config changes

Check the Test Report's "Test-only Config Changes" section.

For each change that was NOT reverted:
```bash
# Check current state
git diff config/

# Revert test-only changes
git checkout config/development.toml    # or whichever file
```

If there were no test-only config changes, skip this step.

### Step 2: Final pre-commit checks

```bash
export PATH="$HOME/.cargo/bin:$PATH"
cargo clippy --all-targets --all-features
cargo +nightly fmt --all
```

If either produces changes or errors, fix them and commit:
```bash
git add -A
git commit -m "chore({{CONNECTOR_NAME}}): lint and format fixes for {{FLOW_NAME}}"
```

### Step 3: Verify commit history

```bash
git log --oneline main..HEAD
```

Expected commits (in order, newest first):
- chore: lint/format fixes (if Step 2 produced changes)
- feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} connector integration (from S3)
- feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} flow type infrastructure (from S2, if applicable)
- fix(...): ... (if there were S3 retries)

Verify there are no accidental commits, no config-only commits, no
work-in-progress commits.

**CRITICAL: Verify NO commits from other flows' branches are present.**
Every commit in `git log --oneline main..HEAD` must belong to THIS flow
({{FLOW_NAME}}). If you see commits mentioning other flow names (e.g.,
from a temporary merge that wasn't properly reset), STOP and report the
contamination — do NOT push. This would mean S4's Step 10 cleanup failed.

### Step 4: Push

```bash
git push -u origin {{BRANCH_NAME}}
```

Do NOT force push. Do NOT amend any commits.

### Step 5: Create PR

```bash
gh pr create --title "feat({{CONNECTOR_NAME}}): add {{FLOW_NAME}} flow for {{CONNECTOR_NAME}}" --body "$(cat <<'PREOF'
## Summary

- Added {{FLOW_NAME}} flow type infrastructure (marker, domain types, sub-trait, proto, conversions, gRPC handler, default impls)
- Implemented {{CONNECTOR_NAME}} connector integration for {{FLOW_NAME}} (request transformer, response transformer, URL, headers, error handling)
{{IF PRE_CALLS}}- Implemented pre-call dependency flows: {{PRE_CALL_LIST}}{{END IF}}
{{IF CONFIG}}- Updated config for {{CONFIG_CHANGES}}{{END IF}}

## How did you test this?

### gRPC smoke test against {{CONNECTOR_NAME}} preprod — PASS

{{IF PRE_CALLS}}
**Pre-call chain:**
{{FOR EACH PRE_CALL}}
- {{PRE_CALL_FLOW}}: Executed via grpcurl, captured {{OUTPUT_FIELD}}={{OUTPUT_VALUE}}
{{END FOR}}
{{ELSE}}
**Pre-calls:** No pre-call dependencies for this flow.
{{END IF}}

**Request:**
```bash
grpcurl -plaintext \
  -H 'x-connector-config: <REDACTED>' \
  -H 'x-merchant-id: {{MERCHANT_ID}}' \
  -H 'x-tenant-id: default' \
  -H 'x-request-id: test_{{FLOW_NAME}}_{{ATTEMPT}}' \
  -H 'x-connector-request-reference-id: test_ref_{{FLOW_NAME}}_{{ATTEMPT}}' \
  -d '{{REQUEST_BODY_WITH_SECRETS_REPLACED}}' \
  localhost:8000 types.{{SERVICE_NAME}}/{{FLOW_NAME}}
```

**Response (HTTP 200 — SUCCESS):**
```json
{{FULL_UNREDACTED_RESPONSE_JSON}}
```

{{IF TEST_CONFIG}}
**Note:** During testing, `config/development.toml` had `{{FIELD}}` set to
`{{TEST_VALUE}}` to reach the {{ENVIRONMENT}} endpoint. This has been reverted
to the standard value before commit. To reproduce the test locally, set this
URL in your development config.
{{END IF}}
PREOF
)"
```

## ╔══════════════════════════════════════════════════════════════╗
## ║  SECURITY RULES FOR PR DESCRIPTION                          ║
## ╚══════════════════════════════════════════════════════════════╝

**x-connector-config header:**
- Replace the ENTIRE value with `<REDACTED>`
- This header contains API keys, salt keys, merchant secrets
- NEVER include any part of it in the PR

**Request body (-d argument):**
- Replace ONLY actual secrets (API keys, salt keys, auth tokens) with
  `<PLACEHOLDER_DESCRIPTION>` (e.g., `<SALT_KEY>`, `<API_KEY>`)
- Keep non-secret identifiers: merchant_id, phone numbers, reference IDs
  are OK to include
- SecretString values in protobuf JSON: replace the inner value only
  (e.g., `{"value":"<SALT_KEY>"}`)

**Response body:**
- Include the FULL response JSON EXACTLY as returned
- Do NOT redact ANY part of the response — reviewers need the complete
  raw output to verify correctness
- This includes: rawConnectorRequest, rawConnectorResponse, base64 payloads,
  checksums, headers, field values — EVERYTHING

**Examples:**

CORRECT (header redacted, body secrets replaced, response intact):
```
-H 'x-connector-config: <REDACTED>' \
-d '{"mobile_number":{"value":"8178392166"},"merchant_id":{"value":"JUSPAYUAT"}}' \
```
Response: <full JSON as-is>

WRONG (response redacted — NEVER do this):
```
Response: { "otpToken": "<REDACTED>", ... }
```

WRONG (header partially shown — NEVER do this):
```
-H 'x-connector-config: {"config":{"Phonepe":{"merchant_id":"JUSPAYUAT",...}}}' \
```

### Step 6: Verify PR was created

```bash
gh pr view --json number,url,title,state
```

## Return PR REPORT

Return this exact structure:

### PR REPORT

**PR URL:** <url>
**PR Number:** #<number>
**Branch:** {{BRANCH_NAME}}
**Title:** <title>

**Commits:**
| # | SHA (short) | Message |
|---|-------------|---------|
(from git log --oneline main..HEAD)

**Test status:** PASS (attempt {{ATTEMPT_NUMBER}})
**Config reverted:** YES / NO / N/A

**Pre-call flows included on this branch:**
| Flow | Connector | New/Existing |
|------|-----------|-------------|
(or "None — no pre-calls")

**Files changed:** <count>
**Lines:** +<added> / -<removed>

---
END PR REPORT.
```

---

### End of Appendix A
