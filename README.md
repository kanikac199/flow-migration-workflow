# Flow Migration Workflow

**AI-agent-executable playbook for migrating payment gateway flows from `euler-api-txns` (Haskell/PureScript) to `connector-service` (Rust).**

This repository contains structured workflow documentation that an AI coding agent (or a human engineer) can follow step-by-step to migrate payment connector integrations between two large-scale payment systems. The docs are designed to be consumed by AI agents as prompts — copy, fill in placeholders, and execute.

---

## What This Solves

`euler-api-txns` is a Haskell monorepo with 53+ payment gateway adapters, 100+ API endpoints, and 18 async workflows — all transpiled from PureScript. `connector-service` is a stateless Rust service with 75+ connector adapters using a trait-based architecture.

Migrating flows between these two systems requires understanding:
- Haskell-to-Rust translation patterns (records, ADTs, typeclasses to traits)
- The bridge pattern (both services run in parallel during migration)
- 7-layer connector-service architecture (flow markers, sub-traits, proto, gRPC handlers, etc.)
- Feature-flag-gated gradual rollout
- Connector-specific quirks (dual base URLs, request signing, Base64 encoding, etc.)

This playbook encodes all of that knowledge into executable steps.

---

## Quick Start

### For AI Agents

1. Open [agent_prompt.md](agent_prompt.md)
2. Copy the prompt template
3. Replace the `{{PLACEHOLDERS}}`:
   - `{{GATEWAY_NAME}}` — the gateway to migrate (e.g., `HDFC`, `PhonePe`, `Paytm`)
   - `{{GATEWAY_DIR}}` — the directory name in euler-api-txns (e.g., `HDFC`, `PhonePe`)
   - `{{FLOWS}}` — comma-separated list of flows (e.g., `Authorize, PSync, Capture, Refund`)
4. Paste the filled prompt into your AI coding agent
5. The agent will follow the orchestrator and execute each step

**Prompt variants available:**

| Variant | Use Case |
|---------|----------|
| **Full migration** | Migrate an entire gateway end-to-end |
| **Dry run** | Analysis only — produces a migration plan without writing code |
| **Batch planning** | Plan migrations for multiple gateways at once |
| **Single flow** | Add one flow to an already-bridged gateway |
| **Phase 2 only** | Core flow migration after Phase 1 is complete |

### For Human Engineers

1. Start at [1_orchestrator.md](1_orchestrator.md) — the master entry point
2. Follow steps sequentially; each step links to a detailed sub-document
3. Do not skip steps — each has prerequisites from previous steps
4. Use [9_gateway_migration_checklist.md](9_gateway_migration_checklist.md) as a standalone checklist

### Quick Path (experienced engineers)

```
3.2_pre_migration_checklist.md  →  Fill checklist for your gateway
        |
        v
4.1_create_connector.md  →  Build connector in Rust
        |
        v
4.2_build_bridge.md  →  Wire bridge in Haskell
        |
        v
4.4_rollout.md  →  Gradual rollout with feature flags
        |
        v
8.1_testing_strategy.md  →  Shadow traffic + canary + gRPC smoke test
        |
        v
9_gateway_migration_checklist.md  →  Sign-off
```

---

## Repository Structure

```
flow-migration-workflow/
│
├── README.md                          ← You are here
├── 1_orchestrator.md                  ← Master entry point — execution sequence
├── agent_prompt.md                    ← Copy-paste prompt templates for AI agents
│
├── 2_overview/
│   ├── 2.1_strategy.md               ← Why we're migrating, phased approach, bridge pattern
│   └── 2.2_architecture_comparison.md ← Side-by-side Haskell vs Rust layer mapping
│
├── 3_planning/
│   ├── 3.1_flow_inventory.md          ← Tiered gateway list, flow priorities
│   ├── 3.2_pre_migration_checklist.md ← Per-gateway analysis checklist
│   └── 3.3_decision_trees.md          ← "Should I migrate this?" decision trees
│
├── 4_phase1_gateway_migration/
│   ├── 4.1_create_connector.md        ← Step-by-step connector creation in Rust
│   ├── 4.2_build_bridge.md            ← Wire the bridge in Haskell (euler_txn side)
│   ├── 4.3_per_flow_implementation.md ← Per-flow trait mapping table
│   ├── 4.4_rollout.md                 ← 0% → 1% → 10% → 50% → 100% rollout
│   └── 4.5_new_flow_type.md           ← Adding entirely new flow types (7-layer guide)
│
├── 5_phase2_core_flow_migration/
│   ├── 5.1_payment_authorize.md       ← Migrate payment orchestration
│   ├── 5.2_refund.md                  ← Migrate refund routing
│   ├── 5.3_webhook.md                 ← Migrate webhook parsing
│   └── 5.4_mandate.md                 ← Migrate mandate/recurring flows
│
├── 6_phase3_async_workflow_migration/
│   ├── 6.1_process_tracker_architecture.md ← ProcessTracker deep-dive
│   └── 6.2_txn_sync_workflow.md            ← TxnSync workflow migration
│
├── 7_reference/
│   ├── 7.1_type_mapping.md            ← TxnStatus ↔ AttemptStatus, error codes, etc.
│   ├── 7.2_code_pattern_translation.md ← Haskell → Rust pattern cookbook
│   └── 7.3_file_references.md         ← Key file paths in both repos
│
├── 8_testing_and_operations/
│   ├── 8.1_testing_strategy.md        ← Unit tests, shadow traffic, canary, gRPC smoke test
│   ├── 8.2_validation_checklist.md    ← Per-gateway per-flow sign-off criteria
│   └── 8.3_rollback_plan.md           ← Feature flag rollback procedure
│
└── 9_gateway_migration_checklist.md   ← Standalone single-gateway checklist
```

---

## Migration Phases

The migration follows a three-phase incremental strategy where both systems run in parallel:

```
┌─────────────────┐        gRPC / HTTP        ┌─────────────────────┐
│   euler_txn     │ ────────────────────────►  │  connector-service  │
│  (orchestrator) │                            │  (connector layer)  │
│                 │ ◄────────────────────────  │                     │
│  - Auth/routing │     Unified response       │  - Build request    │
│  - DB state     │                            │  - Call gateway API  │
│  - Async flows  │                            │  - Parse response   │
└─────────────────┘                            └─────────────────────┘
```

| Phase | What Moves | Risk | Docs |
|-------|-----------|------|------|
| **Phase 1** — Gateway adapters | Request building, API calls, response parsing | Low | `4_phase1_gateway_migration/` |
| **Phase 2** — Core flows | Payment orchestration, refund routing, webhooks | Medium | `5_phase2_core_flow_migration/` |
| **Phase 3** — Async workflows | Transaction sync, refund sync, webhook delivery | High | `6_phase3_async_workflow_migration/` |

Phase gates with concrete criteria exist between phases — see [1_orchestrator.md](1_orchestrator.md) for details.

---

## Key Concepts

### Bridge Pattern
Both systems run simultaneously. `euler_txn` remains the orchestrator; `connector-service` handles only the stateless gateway communication. Traffic is shifted gradually via feature flags, with instant rollback capability.

### 7-Layer Architecture (New Flow Types)
When migrating a flow that doesn't yet exist in connector-service (e.g., wallet OTP, balance check), you must create infrastructure across 7 layers:

```
Layer 1: Flow marker type          (domain_types/connector_flow.rs)
Layer 2: Domain request/response   (domain_types/connector_types.rs)
Layer 3: Sub-trait definition      (interfaces/connector_types.rs)
Layer 4: Proto messages + RPC      (grpc-api-types/proto/)
Layer 5: Proto-to-domain conversions (domain_types/types.rs)
Layer 6: gRPC handler              (grpc-server/src/server/)
Layer 7: Default implementations   (connector-integration/default_implementations.rs)
```

See [4.5_new_flow_type.md](4_phase1_gateway_migration/4.5_new_flow_type.md) for the complete guide.

### Connector-Specific Config via gRPC Headers
Connector credentials are passed per-request via the `x-connector-config` gRPC metadata header using serde JSON format:

```json
{"config":{"Phonepe":{"merchant_id":"...","salt_key":"...","salt_index":"..."}}}
```

Key rules:
- Variant names are **PascalCase** matching the Rust enum (e.g., `Phonepe`, `Stripe`, `Adyen`)
- `Secret<String>` fields are **plain strings** (not `{"value":"..."}`)
- Protobuf JSON in request bodies uses `{"value":"..."}` for `SecretString` message types

### Dual Base URLs
Some connectors (e.g., PhonePe) use different API gateways for different flow categories. connector-service supports this via `secondary_base_url` and `third_base_url` in the config:

```toml
phonepe.base_url = "https://api.phonepe.com/apis/hermes/"           # PG flows
phonepe.secondary_base_url = "https://mercury-t2.phonepe.com/"      # Wallet flows
```

---

## Testing

The testing strategy has multiple layers:

1. **Unit tests** — Request/response transformation tests in `ucs-connector-tests/`
2. **gRPC smoke test** — Local runtime validation against connector's sandbox/preprod (Section 8.1.5)
3. **Shadow traffic** — Run both paths in production, compare responses, use old path as source of truth
4. **Canary deployment** — Gradual traffic shift: 1% → 10% → 50% → 100%

### Running a gRPC Smoke Test

```bash
# 1. Build and start the server
cargo build -p grpc-server
RUN_ENV=development ./target/debug/grpc-server &

# 2. Send a test request (example: PhonePe TriggerOtpForWallet)
grpcurl -plaintext \
  -H 'x-connector-config: {"config":{"Phonepe":{"merchant_id":"MERCHANT","salt_key":"KEY","salt_index":"1"}}}' \
  -H 'x-merchant-id: MERCHANT' \
  -H 'x-tenant-id: default' \
  -H 'x-request-id: test_001' \
  -H 'x-connector-request-reference-id: ref_001' \
  -d '{"mobile_number":{"value":"1234567890"},"merchant_id":{"value":"MERCHANT"}}' \
  localhost:8000 types.PaymentService/TriggerOtpForWallet

# 3. Validate: check rawConnectorRequest URL, headers, body match Haskell impl
```

See [8.1_testing_strategy.md](8_testing_and_operations/8.1_testing_strategy.md) Section 8.1.5 for the complete guide including common issues and connector-specific notes.

---

## Reference Docs

These are consulted during implementation — not read sequentially:

| Document | When to Use |
|----------|------------|
| [7.1_type_mapping.md](7_reference/7.1_type_mapping.md) | Mapping `TxnStatus` → `AttemptStatus`, error codes, currency formats |
| [7.2_code_pattern_translation.md](7_reference/7.2_code_pattern_translation.md) | Translating Haskell records, ADTs, `Maybe`/`Either` to Rust equivalents |
| [7.3_file_references.md](7_reference/7.3_file_references.md) | Finding the right file in either repo |

---

## Validated Test Case

This workflow has been tested end-to-end with a deliberately complex case:

**Test:** PhonePe `triggerOTPForWallet` — requires creating an entirely new flow type (7 layers), dual base URL support, Base64+SHA256 request signing.

**Results:**
- Round 1: Implementation failed review (incorrect type params, missing gRPC handler) → workflow docs updated → code reverted → re-run
- Round 2: All 7 layers passed review, `cargo check` passed
- gRPC runtime test: Successfully sent OTP request to PhonePe preprod, received `SUCCESS` response with `otpToken`
- All layers verified: proto conversion, auth parsing, request transform, checksum, HTTP call, response extraction

The workflow docs were iteratively refined based on these test results.

---

## Prerequisites

### For the repos being migrated
- Access to `euler-api-txns` (Haskell/PureScript payment service)
- Access to `connector-service` (Rust connector service)

### For running connector-service locally
- Rust toolchain (1.70+)
- `grpcurl` for gRPC testing (`brew install grpcurl`)
- Connector sandbox/preprod credentials for smoke testing

### For AI agent execution
- An AI coding agent with file read/write, terminal access, and the ability to follow multi-step instructions
- Both repos cloned locally at known paths
- Update the repo paths in `agent_prompt.md` and `1_orchestrator.md` if they differ from the defaults

---

## FAQ

**Q: Can I migrate a single flow independently?**
Yes. Use the "Single Flow Migration" prompt variant in [agent_prompt.md](agent_prompt.md). This is for adding a flow to a gateway that already has other flows bridged.

**Q: What if the flow type doesn't exist in connector-service?**
Follow [4.5_new_flow_type.md](4_phase1_gateway_migration/4.5_new_flow_type.md) first. This creates the full infrastructure (flow marker, sub-trait, proto, gRPC handler, default impls) before implementing the connector-specific logic.

**Q: How do I roll back if something goes wrong?**
Flip the feature flag. The old euler_txn code path remains fully functional throughout the migration. See [8.3_rollback_plan.md](8_testing_and_operations/8.3_rollback_plan.md).

**Q: Can an AI agent do the full migration autonomously?**
The agent prompt is designed for supervised execution — the agent does the work, but a human reviews before committing. The "Dry run" variant produces a plan without writing code, useful for initial assessment.

**Q: How do I update the workflow docs when I discover new patterns?**
The recommended cycle is: implement → review → if issues found, update docs → revert code → re-run. This ensures the docs stay accurate and self-correcting.

---

## Contributing

If you discover new patterns, connector quirks, or workflow gaps during a migration:

1. Document the finding in the relevant section
2. If it's a new connector-specific issue (e.g., dual base URLs), add it to the "Common Issues" table in [8.1_testing_strategy.md](8_testing_and_operations/8.1_testing_strategy.md)
3. If it's a new Haskell→Rust translation pattern, add it to [7.2_code_pattern_translation.md](7_reference/7.2_code_pattern_translation.md)
4. If it's a new type mapping, add it to [7.1_type_mapping.md](7_reference/7.1_type_mapping.md)

---

*Last updated: 2026-03-22*
