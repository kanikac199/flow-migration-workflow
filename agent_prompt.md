# Agent Prompt: Migrate a Gateway

> **Copy everything below the line and paste it as your first message to an AI coding agent.**
> Replace the `{{PLACEHOLDERS}}` with real values before pasting.

---

## Prompt

```
You are executing a gateway migration from euler-api-txns (Haskell) to
connector-service (Rust). Follow the workflow documentation precisely.

## Target

- Gateway to migrate: {{GATEWAY_NAME}}          (e.g., HDFC, Paytm, Billdesk)
- euler_txn gateway dir: euler-x/src-generated/Gateway/{{GATEWAY_DIR}}/
- Flows to migrate: {{FLOWS}}                    (e.g., Authorize, PSync, Capture, Refund, RSync)

## Repos

- euler-api-txns:    /Users/kanika.c/code/euler-api-txns/
- connector-service: /Users/kanika.c/code/connector-service/

## Workflow

Read and follow the orchestrator at:

  /Users/kanika.c/code/workflow/flow-migration-workflow/1_orchestrator.md

Execute each step in order. At each step, read the linked document, then do
the work it describes. Do not skip steps. Do not proceed past a phase gate
until its criteria are met.

### Execution sequence

1. Read 2_overview/2.1_strategy.md and 2_overview/2.2_architecture_comparison.md
   to understand the architecture. No code changes yet.

2. Read 3_planning/3.1_flow_inventory.md to confirm the gateway tier.
   Read 3_planning/3.3_decision_trees.md — walk through each decision tree
   for {{GATEWAY_NAME}} and state the answers.

3. Complete every item in 3_planning/3.2_pre_migration_checklist.md for
   {{GATEWAY_NAME}}. For the "Analysis Phase" items, actually read the
   euler_txn source files listed and report your findings. Do not guess.

4. Check whether the flow types in {{FLOWS}} already exist in connector-service.
    Read 4_phase1_gateway_migration/4.3_per_flow_implementation.md — the Flow
    Mapping Table lists all existing flow types. If ANY flow in {{FLOWS}} is NOT
    in that table (e.g., TriggerOtp, VerifyOtp, CheckBalance, or any wallet/OTP
    flow), you MUST first follow 4_phase1_gateway_migration/4.5_new_flow_type.md
    to create the full flow infrastructure (flow marker, sub-trait, proto RPC,
    gRPC handler, default implementations) BEFORE proceeding to step 5.

 5. Implement the connector in connector-service following
    4_phase1_gateway_migration/4.1_create_connector.md:
    - Create connectors/{{CONNECTOR_NAME}}/mod.rs
    - Create connectors/{{CONNECTOR_NAME}}/transformers.rs
    - Register in connectors.rs, types.rs, connector_types.rs, payment.proto
    - Add config to config/*.toml
    - Use 7_reference/7.1_type_mapping.md for all status mappings
    - Use 7_reference/7.2_code_pattern_translation.md for Haskell->Rust patterns
    - Use 7_reference/7.3_file_references.md to find the right files

 6. Wire the bridge in euler_txn following
    4_phase1_gateway_migration/4.2_build_bridge.md:
    - Add to euler-x/src-generated/Gateway/ConnectorService/Flow.hs dispatcher
    - Create euler-x/src-generated/Gateway/ConnectorService/{{GATEWAY_DIR}}.hs handler if needed
    - Add feature flag gate via isConnectorServiceEnabledForFunction in CommonGateway.hs

 7. Implement each flow from {{FLOWS}} following
    4_phase1_gateway_migration/4.3_per_flow_implementation.md.
    Use macro_connector_implementation! for flows NOT in {{FLOWS}}.

 8. Write unit tests for every request/response transformation following
    8_testing_and_operations/8.1_testing_strategy.md section 8.1.1.

 9. Fill out the validation checklist from
    8_testing_and_operations/8.2_validation_checklist.md for each flow.

10. Report a summary of:
   - Files created/modified (with paths)
   - Flows implemented
   - Flows marked unsupported (macro)
   - Any type mappings that were ambiguous or missing
   - Any issues or decisions that need human input

## Rules

- Do NOT modify any existing connector in connector-service (only add new).
- Do NOT remove or modify existing euler_txn gateway adapters (the old path
  must remain functional).
- If a type mapping is ambiguous, flag it and ask — do not guess.
- If the gateway requires encryption (AES, JWE, RSA, etc.), flag it and ask
  for guidance before implementing.
- **Follow connector-service naming conventions, NOT euler-api-txns conventions.**
  All proto field names, Rust struct field names, enum variants, function names,
  and domain type names must match the patterns already established in
  connector-service. Do NOT carry over Haskell/PureScript naming verbatim.
  Before introducing ANY new field or type name:
  1. Grep the existing `.proto` files in `crates/types-traits/grpc-api-types/proto/`
     for the concept you need (e.g., phone, address, amount, status, token, etc.)
  2. Grep existing Rust domain types in `crates/types-traits/domain_types/src/`
  3. Use whatever name the codebase already uses for that concept
  4. Only invent a new name if the concept is genuinely new to connector-service
  Gateway-specific serialization names (e.g., `#[serde(rename = "mobileNumber")]`
  for PhonePe's API) are fine — they match the external API contract, not our
  internal naming.
- **BEFORE EVERY COMMIT**, run both of these commands and fix any issues:
  ```bash
  cargo clippy --all-targets --all-features   # must pass with zero errors/warnings
  cargo +nightly fmt --all                     # must produce no changes
  ```
  Do not commit code that fails clippy or is not formatted.
- Commit nothing. Stage changes only. I will review before committing.
```

---

## Variants

### Dry Run (Analysis Only, No Code Changes)

Replace the "Execution sequence" section with:

```
### Execution sequence

1-3. Same as above (read docs, walk decision trees, complete checklist).

4. STOP. Do not write any code. Instead, produce a migration plan:
   - List every file you would create in connector-service (with purpose)
   - List every file you would modify in euler_txn (with the change)
   - List all type mappings needed (with any ambiguities)
   - List all feature toggles that affect this gateway
   - Estimate complexity (low / medium / high) per flow
   - Flag any blockers or unknowns
```

### Multiple Gateways (Batch Planning)

```
You are planning migrations for multiple gateways. For each gateway below,
complete steps 1-3 of the workflow (analysis only, no code). Then produce
a prioritized migration plan.

Gateways: {{GATEWAY_1}}, {{GATEWAY_2}}, {{GATEWAY_3}}

Read the orchestrator at:
  /Users/kanika.c/code/workflow/flow-migration-workflow/1_orchestrator.md

For each gateway:
1. Walk through the decision trees in 3_planning/3.3_decision_trees.md
2. Complete the analysis items in 3_planning/3.2_pre_migration_checklist.md
3. Summarize: flows to migrate, complexity, blockers, dependencies

Then rank the gateways by migration priority and explain your reasoning.
```

### Single Flow Migration (Add One Flow to an Already-Bridged Gateway)

Use this when the gateway already has at least one flow migrated to
connector-service and you want to add another flow (e.g., add Refund to a
gateway that already has Authorize).

```
You are adding a single flow to an existing gateway bridge. The gateway
already has some flows migrated to connector-service.

## Target

- Gateway: {{GATEWAY_NAME}}
- New flow to add: {{FLOW_NAME}}       (e.g., Refund, PSync, Capture, Void, RSync, Webhook)
- Already migrated flows: {{EXISTING_FLOWS}}  (e.g., Authorize, PSync)

## Repos

- euler-api-txns:    /Users/kanika.c/code/euler-api-txns/
- connector-service: /Users/kanika.c/code/connector-service/

## Workflow

Read the orchestrator at:
  /Users/kanika.c/code/workflow/flow-migration-workflow/1_orchestrator.md

### Execution sequence

1. Read 4_phase1_gateway_migration/4.3_per_flow_implementation.md to
   understand how per-flow traits map between euler_txn and connector-service.

2. Analyze the existing euler_txn implementation of {{FLOW_NAME}} for
   {{GATEWAY_NAME}}:
   - Read euler-x/src-generated/Gateway/{{GATEWAY_NAME}}/Flow.hs
   - Identify the function that handles {{FLOW_NAME}}
   - Document all request/response fields, error handling, and special cases
   - Check 7_reference/7.1_type_mapping.md for status mappings

3. Implement the flow trait in connector-service:
   - Add the {{FLOW_NAME}} impl block in
     backend/connector-integration/src/connectors/{{CONNECTOR_NAME}}/mod.rs
   - Add request/response transformers in
     backend/connector-integration/src/connectors/{{CONNECTOR_NAME}}/transformers.rs
   - Remove the macro_connector_implementation! stub for this flow if present

4. Wire the bridge call site in euler_txn:
   - Read 4_phase1_gateway_migration/4.2_build_bridge.md section
     "Adding a New Flow to an Existing Bridge"
   - Add a new dispatcher function in
     euler-x/src-generated/Gateway/ConnectorService/Flow.hs
     (e.g., callConnectorService{{FLOW_NAME}})
   - Add the call site in euler-x/src-generated/Gateway/CommonGateway.hs
     (or the relevant orchestration module) with the
     isConnectorServiceEnabledForFunction gate using a new function name
   - Add the per-gateway handler in
     euler-x/src-generated/Gateway/ConnectorService/{{GATEWAY_NAME}}.hs
     (update existing file)

5. Write unit tests following
   8_testing_and_operations/8.1_testing_strategy.md section 8.1.1.

6. Fill out the validation checklist row for this flow from
   8_testing_and_operations/8.2_validation_checklist.md.

7. Report a summary of:
   - Files created/modified (with paths)
   - The new flow implemented
   - Type mappings used (with any ambiguities)
   - The feature flag function name added to enabledFunctions
   - Any issues or decisions that need human input

## Rules

- Do NOT modify any existing flows — only add the new one.
- The old euler_txn path for {{FLOW_NAME}} must remain functional.
- The new flow must be gated behind isConnectorServiceEnabledForFunction
  with a distinct function name so it can be rolled out independently.
- If a type mapping is ambiguous, flag it and ask — do not guess.
- Commit nothing. Stage changes only. I will review before committing.
```

### Phase 2 Only (Core Flow Migration)

```
Phase 1 is complete for {{GATEWAY_NAME}}. Now execute Phase 2.

Read the Phase 2 documents:
  - 5_phase2_core_flow_migration/5.1_payment_authorize.md
  - 5_phase2_core_flow_migration/5.2_refund.md
  - 5_phase2_core_flow_migration/5.3_webhook.md
  - 5_phase2_core_flow_migration/5.4_mandate.md

For each flow, analyze the current euler_txn implementation and describe
what would need to change. Do not write code — produce a migration plan
with risks, dependencies, and estimated effort.
```

---

## Quick Start Example

To migrate HDFC with Authorize, PSync, Capture, Void, Refund, and RSync flows,
copy the main prompt above and fill in:

```
Gateway to migrate: HDFC
euler_txn gateway dir: euler-x/src-generated/Gateway/HDFC/
Flows to migrate: Authorize, PSync, Capture, Void, Refund, RSync
```

Then paste the filled prompt into your AI agent.
