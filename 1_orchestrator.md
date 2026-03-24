# Flow Migration Orchestrator: euler_txn -> connector-service

> **This is the master entry point.** Read this file first. It defines the execution
> sequence and tells you which sub-document to follow at each step.
>
> Source: `euler-api-txns` (Haskell/PureScript) -- `/Users/kanika.c/code/workflow/euler-api-txns/`
> Target: `connector-service` (Rust) -- `/Users/kanika.c/code/workflow/connector-service/`

---

## How to Trigger This Workflow

**For AI agents:** Paste one of the entry prompts from [agent_prompt.md](agent_prompt.md) into
a new session. The agent acts as an **orchestrator** that spawns 5 subagents:
1. **Analysis** (explore) — discover connectors, analyze Haskell code, identify pre-call dependencies, check connector capability
2. **Core Flow Infrastructure** (general) — create flow type infrastructure (7 layers), including pre-call flow types if missing
3. **Connector Integration** (general) — implement connector-specific code (transformers, URL, headers, auth) for target flow + pre-call dependencies
4. **Testing** (general) — execute pre-call chain via grpcurl, test target flow, MUST get HTTP 200. On failure → retry loop (back to Subagent 3). On capability mismatch → auto-switch connector.
5. **Push & PR** (general) — revert test-only config, push branch (commits already done by S2/S3), raise PR with full test evidence

The orchestrator runs a **retry loop** between Subagents 3 and 4 until HTTP 200 is received.
If a connector can't support the flow in test, it auto-switches to the next connector.

You only need to specify the **flow name** (e.g., "Migrate the VerifyOtpForWallet flow") —
the agent discovers which connectors implement it. See `agent_prompt.md` for variants:
dry-run (analysis only), gateway-centric (multiple flows for one gateway), and batch planning.

**For humans:** Follow the steps below sequentially, using each linked document.

## How to Use This Workflow

1. **Start here.** This file is the orchestrator.
2. **Follow steps sequentially.** Each step links to a detailed sub-document.
3. **Do not skip steps.** Each step has prerequisites from previous steps.
4. **Go/No-Go gates** exist between phases -- do not proceed until the gate criteria are met.
5. **Loop per gateway/flow.** Steps 4-8 repeat for every gateway or flow being migrated.

---

## Quick Path: "I just need to migrate one gateway fast"

If you already understand the architecture and just need to execute:

```
3.2_pre_migration_checklist.md  -->  Fill checklist for your gateway
        |
        v
4.1_create_connector.md  -->  Build connector in Rust (connector-service)
        |
        v
4.2_build_bridge.md  -->  Wire bridge in Haskell (euler_txn)
        |
        v
4.4_rollout.md  -->  Gradual rollout with feature flags
        |
        v
8.1_testing_strategy.md  -->  Shadow traffic + canary
        |
        v
9_gateway_migration_checklist.md  -->  Sign-off checklist
```

---

## Full Execution Sequence

### Step 1: Understand the Landscape

| Action | Document | What You Get |
|--------|----------|-------------|
| Read the migration strategy | [2.1_strategy.md](2_overview/2.1_strategy.md) | Why we're migrating, phased approach, key principles |
| Understand both architectures | [2.2_architecture_comparison.md](2_overview/2.2_architecture_comparison.md) | Side-by-side Haskell vs Rust layer mapping |

**Output:** You understand the bridge pattern and why euler_txn stays as orchestrator in Phase 1.

---

### Step 2: Plan the Migration

| Action | Document | What You Get |
|--------|----------|-------------|
| Review all flows & priorities | [3.1_flow_inventory.md](3_planning/3.1_flow_inventory.md) | Tiered gateway list, core flow priorities, async workflow priorities |
| Select your target gateway/flow | [3.3_decision_trees.md](3_planning/3.3_decision_trees.md) | Decision trees for "should I migrate this now?" |
| Complete pre-migration checklist | [3.2_pre_migration_checklist.md](3_planning/3.2_pre_migration_checklist.md) | Analysis, target state design, infrastructure readiness |

**Output:** You have a filled checklist and a clear target (gateway + flows to migrate).

---

### Step 3: Execute Phase 1 -- Gateway Connector Migration

> **Phase 1 scope:** Only the gateway communication layer moves to connector-service.
> euler_txn remains the orchestrator (auth, DB, routing, async workflows stay).

| Action | Document | What You Get |
|--------|----------|-------------|
| **3a.** Create/update connector in Rust | [4.1_create_connector.md](4_phase1_gateway_migration/4.1_create_connector.md) | ConnectorCommon, ConnectorIntegrationV2 traits, transformers, registration |
| **3b.** Build bridge in euler_txn | [4.2_build_bridge.md](4_phase1_gateway_migration/4.2_build_bridge.md) | Bridge dispatcher, request mapping, response conversion, feature flag |
| **3c.** Verify per-flow trait mapping | [4.3_per_flow_implementation.md](4_phase1_gateway_migration/4.3_per_flow_implementation.md) | Authorize/PSync/Capture/Void/Refund/RSync mapping table |
| **3d.** *(If needed)* Add new flow type | [4.5_new_flow_type.md](4_phase1_gateway_migration/4.5_new_flow_type.md) | Flow marker, sub-trait, proto, gRPC handler, default impls |
| **3e.** Gradual rollout | [4.4_rollout.md](4_phase1_gateway_migration/4.4_rollout.md) | 0% -> 1% -> 10% -> 50% -> 100% procedure |

**Reference docs to consult during implementation:**

| Need | Document |
|------|----------|
| Type mapping (TxnStatus -> AttemptStatus, etc.) | [7.1_type_mapping.md](7_reference/7.1_type_mapping.md) |
| Haskell-to-Rust code translation patterns | [7.2_code_pattern_translation.md](7_reference/7.2_code_pattern_translation.md) |
| File paths in both repos | [7.3_file_references.md](7_reference/7.3_file_references.md) |

**Output:** Gateway is live on connector-service behind a feature flag.

> **MANDATORY PRE-COMMIT CHECK:** Before every commit, run:
> ```bash
> cargo clippy --all-targets --all-features   # zero errors/warnings
> cargo +nightly fmt --all                     # zero formatting changes
> ```
> Do not commit code that fails clippy or is unformatted.

---

### Step 4: Test & Validate

| Action | Document | What You Get |
|--------|----------|-------------|
| Run testing strategy | [8.1_testing_strategy.md](8_testing_and_operations/8.1_testing_strategy.md) | Unit tests, shadow traffic, canary deployment |
| gRPC smoke test | [8.1_testing_strategy.md § 8.1.5](8_testing_and_operations/8.1_testing_strategy.md) | End-to-end verification against connector preprod |
| **Commit, push & raise PR** | [8.1_testing_strategy.md § 8.1.6](8_testing_and_operations/8.1_testing_strategy.md) | PR raised with full test evidence (HTTP 200 required) |
| Complete validation checklist | [8.2_validation_checklist.md](8_testing_and_operations/8.2_validation_checklist.md) | Per-gateway per-flow sign-off criteria |
| Verify rollback readiness | [8.3_rollback_plan.md](8_testing_and_operations/8.3_rollback_plan.md) | Feature flag rollback, trigger conditions |

**Output:** Gateway is validated (HTTP 200 from connector), PR is raised with test evidence, rollback plan is ready.

---

### Step 5: Sign Off

| Action | Document |
|--------|----------|
| Complete the migration checklist | [9_gateway_migration_checklist.md](9_gateway_migration_checklist.md) |

**Output:** Gateway migration is complete. Repeat Steps 2-5 for the next gateway.

---

## Phase Gates

### Gate 1: Phase 1 -> Phase 2

**Do NOT start Phase 2 until ALL of these are true:**

- [ ] All Tier 2 gateways (HDFC, Paytm V2, Billdesk, Cybersource, PayU, CCAvenue V2) are migrated and stable at 100% traffic
- [ ] At least 4 weeks of production soak with zero rollbacks
- [ ] Latency p99 is within 50ms of old path for all migrated gateways
- [ ] Error rate delta is < 0.1% for all migrated gateways

**Phase 2 documents:**

| Action | Document |
|--------|----------|
| Payment authorize flow | [5.1_payment_authorize.md](5_phase2_core_flow_migration/5.1_payment_authorize.md) |
| Refund flow | [5.2_refund.md](5_phase2_core_flow_migration/5.2_refund.md) |
| Webhook flow | [5.3_webhook.md](5_phase2_core_flow_migration/5.3_webhook.md) |
| Mandate flow | [5.4_mandate.md](5_phase2_core_flow_migration/5.4_mandate.md) |

---

### Gate 2: Phase 2 -> Phase 3

**Do NOT start Phase 3 until ALL of these are true:**

- [ ] Core transaction flows (authorize, refund, capture, void) are migrated for all target gateways
- [ ] Webhook parsing is handled by connector-service for all migrated gateways
- [ ] At least 4 weeks of production soak on Phase 2
- [ ] A scheduler/task-queue solution has been designed and approved

**Phase 3 documents:**

| Action | Document |
|--------|----------|
| ProcessTracker architecture | [6.1_process_tracker_architecture.md](6_phase3_async_workflow_migration/6.1_process_tracker_architecture.md) |
| TxnSync workflow migration | [6.2_txn_sync_workflow.md](6_phase3_async_workflow_migration/6.2_txn_sync_workflow.md) |

---

## Document Map

```
1_orchestrator.md  (YOU ARE HERE)
│
├── 2_overview/
│   ├── 2.1_strategy.md
│   └── 2.2_architecture_comparison.md
│
├── 3_planning/
│   ├── 3.1_flow_inventory.md
│   ├── 3.2_pre_migration_checklist.md
│   └── 3.3_decision_trees.md
│
├── 4_phase1_gateway_migration/
│   ├── 4.1_create_connector.md
│   ├── 4.2_build_bridge.md
│   ├── 4.3_per_flow_implementation.md
│   ├── 4.4_rollout.md
│   └── 4.5_new_flow_type.md          ← adding flow types that don't exist yet (OTP, wallet, etc.)
│
├── 5_phase2_core_flow_migration/
│   ├── 5.1_payment_authorize.md
│   ├── 5.2_refund.md
│   ├── 5.3_webhook.md
│   └── 5.4_mandate.md
│
├── 6_phase3_async_workflow_migration/
│   ├── 6.1_process_tracker_architecture.md
│   └── 6.2_txn_sync_workflow.md
│
├── 7_reference/
│   ├── 7.1_type_mapping.md
│   ├── 7.2_code_pattern_translation.md
│   └── 7.3_file_references.md
│
├── 8_testing_and_operations/
│   ├── 8.1_testing_strategy.md
│   ├── 8.2_validation_checklist.md
│   └── 8.3_rollback_plan.md
│
├── 9_gateway_migration_checklist.md
└── agent_prompt.md              ← copy-paste prompt to trigger this workflow
```

---

*Last updated: 2026-03-24*
