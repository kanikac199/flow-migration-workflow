# Agent Prompt: Flow Migration Orchestrator

> **This is the entry point for migrating a flow from euler-api-txns to connector-service.**
>
> You (the main agent) act as an **orchestrator**. You do NOT implement code yourself.
> Instead, you spawn subagents for each phase and pass structured handoffs between them.

---

## Orchestrator Behavior

When you receive a flow migration request:

1. Read this file (`agent_prompt.md`) to understand the subagent pipeline.
2. Read `1_orchestrator.md` to understand the overall workflow structure.
3. **When spawning each subagent, include its full prompt template from this file
   as the subagent's context.** Each subagent section below contains a "Prompt
   Template" — copy the appropriate template and fill in the `{{PLACEHOLDERS}}`
   with the handoff data from the previous subagent. The subagent needs this
   context to know what to do.
4. Execute **Subagent 1 → 2 → 3 → 4 → 5** in order, passing the handoff data between them.
5. If any subagent fails or needs user input, surface it to the user immediately.
6. Do NOT proceed to the next subagent until the current one completes successfully.
7. **One branch per flow (with dependency exception).** If the user requests multiple
   flows, run the full pipeline **separately for each flow**. Each flow gets its own
   fresh branch from `main`. Exception: pre-call dependency flows required for testing
   the target flow go on the SAME branch — they are prerequisites, not independent features.
8. **Retry loop on failure.** If Subagent 4 (Testing) returns FAIL:
   - Re-spawn Subagent 3 (Connector Integration) with the error details to fix the code
   - Re-spawn Subagent 4 (Testing) to retest
   - Repeat until HTTP 200 is received
9. **Connector fallback on capability mismatch.** If Subagent 4 returns
   CAPABILITY_MISMATCH, pick the next connector from the analysis report's
   connector list, re-spawn Subagent 3 for the new connector, then Subagent 4.
   Subagent 2 (Core Flow) is NOT re-run — the flow type already exists.
10. **Never stop at partial results.** The pipeline is NOT complete until a
    connector returns HTTP 200 with valid business data, or ALL connectors from
    the analysis report have been exhausted (escalate to user with evidence).

---

## Subagent Pipeline

```
User: "Migrate the VerifyOtpForWallet flow"
  │
  ▼
┌──────────────────────────────────────┐
│  Subagent 1: ANALYSIS (explore)      │
│  - Find connectors in euler-api-txns │
│  - Analyze Haskell code              │
│  - Identify pre-call dependencies    │
│  - Check connector capability        │
│  - Identify fallback connector       │
│  - Check if flow type exists in CS   │
└──────────────┬───────────────────────┘
               │ Handoff: Analysis Report
               ▼
┌──────────────────────────────────────┐
│  Subagent 2: CORE FLOW (general)    │
│  - Create FRESH branch from main    │
│  - Flow marker, domain types,       │
│    sub-trait, proto, conversions,    │
│    gRPC handler, default impls      │
│  - Also for pre-call flow types     │
│    if they don't exist              │
│  - clippy + fmt + build             │
│  (Skip if flow type already exists) │
└──────────────┬───────────────────────┘
               │ Handoff: Core Flow Report
               ▼
┌──────────────────────────────────────┐
│  Subagent 3: CONNECTOR (general)    │◄────────────────────┐
│  - Connector-specific transformers, │                     │
│    URL, headers, auth, error        │                     │
│  - Pre-call connector impls too     │                     │
│  - clippy + fmt + build             │  Retry on FAIL or   │
└──────────────┬───────────────────────┘  CAPABILITY_MISMATCH│
               │ Handoff: Connector Report                   │
               ▼                                             │
┌──────────────────────────────────────┐                     │
│  Subagent 4: TESTING (general)      │                     │
│  - Execute pre-call chain via grpcurl│                     │
│  - Feed outputs into target flow    │                     │
│  - MUST get HTTP 200                │                     │
│  - On FAIL → back to Subagent 3    │─────────────────────►│
│  - On CAPABILITY_MISMATCH →         │                     │
│    orchestrator picks next connector │─────────────────────┘
└──────────────┬───────────────────────┘
               │ Handoff: Test Report (PASS)
               ▼
┌──────────────────────────────────────┐
│  Subagent 5: PUSH & PR              │
│  - Revert test-only config          │
│  - Push branch (commits already     │
│    done by S2 and S3)               │
│  - Raise PR with full test evidence │
└──────────────┬───────────────────────┘
               │
               ▼
         Return PR URL to user
```

### Orchestrator Retry Flow

```
         S1 (Analysis)
              │
              ▼
         S2 (Core Flow Infra)   ← runs once, unless new flow types needed
              │
              ▼
    ┌──► S3 (Connector Integration) ◄──────────────────┐
    │         │                                         │
    │         ▼                                         │
    │    S4 (Testing)                                   │
    │         │                                         │
    │         ├── PASS (200) ──► S5 (Push & PR)          │
    │         │                                         │
    │         ├── FAIL (code bug) ─────────────────────►│
    │         │   (pass error details to S3 for fix)    │
    │         │                                         │
    │         └── CAPABILITY_MISMATCH ──► Orchestrator  │
    │              picks next connector ───────────────►│
    │              (S3 for new connector)               │
    │                                                   │
    └── repeat until 200 or all connectors exhausted ──►│
```

---

## Subagent 1: Analysis

**Type:** `explore`

**Purpose:** Discover which connectors implement the requested flow in euler-api-txns,
analyze the Haskell implementation, identify pre-call dependencies, check connector
capabilities, and determine whether the flow type already exists in connector-service.

### Prompt Template

```
You are analyzing a flow migration from euler-api-txns (Haskell) to connector-service (Rust).

## Target Flow: {{FLOW_NAME}}

## Repos
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Workflow Docs
- Architecture comparison: /home/kanikachaudhary/Kanika/flow-migration-workflow/2_overview/2.2_architecture_comparison.md
- Flow inventory: /home/kanikachaudhary/Kanika/flow-migration-workflow/3_planning/3.1_flow_inventory.md
- Per-flow mapping: /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.3_per_flow_implementation.md
- New flow type guide: /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.5_new_flow_type.md
- Code pattern translation: /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.2_code_pattern_translation.md

## Tasks

1. **Find which connectors implement this flow in euler-api-txns.**
   - Search euler-x/src-generated/Gateway/ for functions matching {{FLOW_NAME}}
     (e.g., grep for "triggerOTP", "verifyOTP", "checkBalance", etc.)
   - Search euler-x/src-generated/Gateway/CommonGateway.hs for the dispatcher
     that routes to per-gateway implementations
   - List every gateway that has an implementation

2. **Analyze the Haskell implementation for each gateway found.**
   For each gateway, read the implementation and document:
   - File path and line numbers (e.g., Gateway/PhonePe/Flow.hs:3194)
   - Function name and signature
   - Request type: all fields, their types, which are required vs optional
   - Response type: success shape, failure shape, all fields
   - API endpoint URL (and which base URL / domain it uses)
   - Authentication method (headers, signature, checksum algorithm)
   - Any encoding (Base64, JSON, form-encoded, XML)
   - Any special handling (caching, retries, error mapping)

3. **Check if the flow type already exists in connector-service.**
   - Read connector-service crates/types-traits/domain_types/src/connector_flow.rs
     for flow marker structs
   - Read crates/types-traits/interfaces/src/connector_types.rs for sub-traits
   - Read crates/types-traits/grpc-api-types/proto/services.proto for existing RPCs
   - State clearly: "Flow type exists: YES/NO"
   - If YES, list the existing marker struct, sub-trait, and RPC name
   - If NO, state what needs to be created (referencing 4.5_new_flow_type.md)

4. **Check naming conventions.**
   - For every field in the Haskell request/response types, grep the existing
     connector-service .proto files and Rust types to find what connector-service
     calls that concept
   - Example: Haskell uses "mobileNumber" → connector-service uses "phone_number"
   - Produce a field mapping table: Haskell name → connector-service name

5. **Identify pre-call dependencies.**
   - Trace the flow's requirements: does it need inputs (IDs, tokens, auth tokens)
     produced by a prior flow?
     Examples: status-check flows need the entity created by a setup flow;
     wallet debit needs an auth token from OTP verification; sync flows need
     a transaction to have been initiated.
   - For each dependency, identify:
     - Which flow produces the required input
     - Whether that flow exists in connector-service for this connector
       (real implementation vs empty stub vs doesn't exist at all)
     - If missing or stub: it must be implemented as a pre-call
   - Build a dependency chain: list flows in execution order needed to get a
     200 from the target flow in test.

6. **Validate connector capability for testing.**
   - Check the connector's test/sandbox environment: does it support this flow?
   - Look for known limitations (production-only features, bank approval required,
     recurring not enabled in test mode)
   - Check the Haskell code for test-mode branching or feature flags
   - If the primary connector is likely unsupported in test, identify a fallback
     connector from the list that CAN test this flow

## Return Format

Return a structured analysis report with these exact sections:

### ANALYSIS REPORT

**Flow:** {{FLOW_NAME}}
**Connectors that implement this flow:** [list]
**Flow type exists in connector-service:** YES/NO

**Per-Connector Analysis:**
(for each connector)
- Haskell file: <path>:<lines>
- Function: <name>
- Endpoint: <URL>
- Auth method: <type>
- Request fields: [field: type (required/optional)]
- Response fields (success): [field: type]
- Response fields (failure): [field: type]
- Special handling: [notes]

**Field Naming Map:**
| Haskell name | connector-service name | Source |
|...|...|...|

**Flow Infrastructure Needed:** (if flow type doesn't exist)
- Flow marker struct: <name>
- Sub-trait: <name>
- Proto RPC: <name>
- Request message: <name>
- Response message: <name>

**Pre-call Dependency Chain:**
| Order | Flow | Exists in connector-service? | Produces |
|-------|------|------------------------------|----------|
| 1 | <pre-call flow> | YES (real) / YES (stub) / NO | <output IDs/tokens> |
| 2 | <target flow> | ... | ... |
(or "None — flow has no pre-call dependencies")

**Connector Capability for Testing:**
- Primary connector: <name> — SUPPORTED / LIKELY UNSUPPORTED (reason: ...)
- Fallback connector: <name> or "N/A — all connectors have same limitation"
```

### Handoff → Subagent 2

Pass the full analysis report as context to Subagent 2.

---

## Subagent 2: Core Flow Infrastructure

**Type:** `general`

**Purpose:** Create the flow type infrastructure in connector-service — all 7 layers
(flow marker, domain types, sub-trait, proto messages, proto-to-domain conversions,
gRPC handler, default implementations). Also creates flow types for any pre-call
dependencies that don't exist yet. This subagent does NOT write connector-specific code.

If the flow type already exists in connector-service (per the analysis report), this
subagent is skipped — proceed directly to Subagent 3.

### Prompt Template

```
You are creating flow type infrastructure in connector-service (Rust).

## Analysis Report
{{PASTE FULL ANALYSIS REPORT FROM SUBAGENT 1}}

## Repos
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Workflow Docs (read these before writing code)
- New flow type (7-layer guide): /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.5_new_flow_type.md
- Per-flow implementation: /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.3_per_flow_implementation.md
- Type mapping reference: /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.1_type_mapping.md
- Code pattern translation: /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.2_code_pattern_translation.md
- File references: /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.3_file_references.md

## Tasks

0. **Create a fresh branch from main:**
   ```bash
   git checkout main && git pull origin main
   git checkout -b feat/<connector>-<flow-name>
   ```
   Every flow MUST get its own branch. Never reuse an existing feature branch.

1. **Create the target flow type** (if it does NOT exist in connector-service):
   Follow 4.5_new_flow_type.md to create all 7 layers:
   - Layer 1: Flow marker struct + FlowName enum variant (connector_flow.rs)
   - Layer 2: Request/response domain types (connector_types.rs)
   - Layer 3: Sub-trait definition (interfaces/connector_types.rs)
   - Layer 4: Proto messages + RPC (payment.proto + services.proto)
   - Layer 5: Proto-to-domain conversions (types.rs)
   - Layer 6: gRPC handler (payments.rs)
   - Layer 7: Default implementations for ALL connectors

2. **Create pre-call flow types** (if any dependencies in the analysis report
   are missing their flow type in connector-service):
   - For each pre-call flow that doesn't exist, create all 7 layers
   - These go on the SAME branch as the target flow

3. **Update flow marker mappings** in:
   - crates/common/common_utils/src/events.rs (FlowName enum + as_str)
   - crates/types-traits/ucs_interface_common/src/flow.rs (type_id mapping)

4. **Run lint, format, and build:**
   ```bash
   cargo clippy --all-targets --all-features
   cargo +nightly fmt --all
   cargo build
   ```
   Fix any errors and repeat until all three pass cleanly.
   Only pre-existing future-incompat warnings from upstream deps are acceptable.

5. **Commit the core flow infrastructure changes:**
   ```bash
   git add -A
   git commit -m "feat(<connector>): add <FlowName> flow type infrastructure"
   ```
   This commit contains ONLY the flow type infrastructure (marker, types, traits,
   proto, handler, defaults). Connector-specific code goes in a separate commit
   by Subagent 3. This separation makes the PR easier to review and allows
   reverting connector code without touching the flow infrastructure.

## Rules

- Follow connector-service naming conventions, NOT euler-api-txns conventions.
  Before introducing ANY new field or type name:
  1. Grep existing .proto files in crates/types-traits/grpc-api-types/proto/
  2. Grep existing Rust domain types in crates/types-traits/domain_types/src/
  3. Use whatever name the codebase already uses for that concept
  4. Only invent a new name if the concept is genuinely new to connector-service
  Gateway-specific serde renames (matching the external API) are fine.

- ResponseRouterData has exactly 2 type params: ResponseRouterData<Response, RouterData>
  (defined at crates/integrations/connector-integration/src/types.rs). NOT 5.

- Do NOT write connector-specific integration code in this subagent. Only create
  the flow type infrastructure (traits, types, proto, handlers, defaults).

- Use the field naming map from the analysis report for all proto/domain fields.

- **Response/request data type reuse rule:**
  Reuse `PaymentsResponseData` (add a variant if needed) when the flow is semantically
  a payment transaction (auth, sync, capture, void, 3DS). Create a standalone struct
  only when the flow is a fundamentally different operation (wallet balance, OTP, mandate
  lifecycle, order creation) whose response fields have zero semantic overlap with payment
  concepts. See `4.5_new_flow_type.md` Layer 2 for the full decision guide.

## Return Format

Return a core flow report with:

### CORE FLOW REPORT

**Flow:** <name>
**Flow type created:** YES / NO (already existed)
**Pre-call flow types created:** [list] or "None needed"
**Commit SHA:** <sha>

**Files modified/created:**
| File | What was changed |
|...|...|

**Build status:** PASS/FAIL
**Clippy status:** PASS/FAIL (zero warnings)
**Fmt status:** PASS/FAIL (no changes)

**Any issues or decisions that need human input:** [list or "None"]
```

### Handoff → Subagent 3

Pass the core flow report (especially the flow types created and files modified)
as context to Subagent 3.

---

## Subagent 3: Connector Integration

**Type:** `general`

**Purpose:** Implement connector-specific integration for the target flow AND all
pre-call dependencies. This subagent writes the request/response transformers, URL
construction, headers, auth, error handling — everything specific to how this
connector talks to the payment gateway. Runs clippy + fmt + build.

This subagent may be re-spawned by the orchestrator if:
- Testing fails (with error details to fix)
- A connector capability mismatch requires switching to a different connector

### Prompt Template

```
You are implementing connector-specific integration in connector-service (Rust).

## Analysis Report
{{PASTE FULL ANALYSIS REPORT FROM SUBAGENT 1}}

## Core Flow Report
{{PASTE CORE FLOW REPORT FROM SUBAGENT 2}}

## If retrying after a test failure:
{{PASTE ERROR DETAILS AND DIAGNOSIS FROM SUBAGENT 4 TEST REPORT}}
(Delete this section if this is the first attempt)

## If switching connector after capability mismatch:
Previous connector: {{NAME}} — failed with: {{ERROR}}
Now implementing: {{NEW CONNECTOR NAME}}
(Delete this section if this is the first connector)

## Repos
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Workflow Docs (read these before writing code)
- New flow type (7-layer guide): /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.5_new_flow_type.md
- Per-flow implementation: /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.3_per_flow_implementation.md
- Create connector: /home/kanikachaudhary/Kanika/flow-migration-workflow/4_phase1_gateway_migration/4.1_create_connector.md
- Type mapping reference: /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.1_type_mapping.md
- Code pattern translation: /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.2_code_pattern_translation.md
- File references: /home/kanikachaudhary/Kanika/flow-migration-workflow/7_reference/7.3_file_references.md

## Tasks

**MANDATORY PRE-CALL GATE (before ANY implementation):**
Before writing a single line of connector code for the target flow, check the
analysis report's pre-call dependency chain. For EVERY pre-call flow listed:
- Does it have a REAL (non-empty, non-stub) connector implementation for this
  connector in connector-service?
- If NO — you MUST implement it first. An empty stub (`impl ... {}`) is NOT
  a real implementation.
- If the pre-call's flow type doesn't exist either, go back to the orchestrator
  to have Subagent 2 create it.
**Do NOT proceed to implementing the target flow until ALL pre-call connector
integrations are real, buildable, and functional.** Skipping a pre-call and
planning to "use a dummy value for testing" is a VIOLATION of this workflow.
There are NO exceptions.

1. **Implement pre-call connector integrations first** (if any dependencies
   identified in the analysis report):
   - For each pre-call flow in the dependency chain, implement the connector's:
     - Request transformer (Haskell request → Rust request)
     - Response transformer (PG response → Rust domain response)
     - URL construction (get_url)
     - Headers (get_headers) including checksums/signatures
     - Content type
     - Error response handling
   - Register each flow in create_all_prerequisites! and macro_connector_implementation!
   - These must produce the outputs needed by the target flow (IDs, tokens, etc.)
   - **Verify each pre-call is NOT an empty stub.** If the connector has
     `impl ConnectorIntegrationV2<...> for Connector<T> {}` with no method
     bodies, that is an empty stub — you must add the real get_url,
     get_request_body, handle_response_v2 methods.

2. **Implement the target flow's connector integration:**
   - Request transformer (Haskell request → Rust request)
   - Response transformer (gateway response → Rust domain response)
   - URL construction (get_url)
   - Headers (get_headers) — including any checksums/signatures
   - Content type
   - Error response handling
   - Register in create_all_prerequisites! and macro_connector_implementation!

3. **Update config files** if the connector needs new config fields
   (e.g., secondary_base_url). Update all 3 TOML files:
   - config/development.toml
   - config/sandbox.toml
   - config/production.toml

4. **Update field-probe auth** if you added new fields to ConnectorSpecificConfig.
   File: crates/internal/field-probe/src/auth.rs

5. **Run lint, format, and build:**
   ```bash
   cargo clippy --all-targets --all-features
   cargo +nightly fmt --all
   cargo build
   ```
   Fix any errors and repeat until all three pass cleanly.

6. **Commit the connector integration changes:**
   ```bash
   git add -A
   git commit -m "feat(<connector>): add <FlowName> connector integration for <Connector>"
   ```
   This commit contains ONLY connector-specific code (transformers, URL, headers,
   auth, error handling, config changes). The core flow infrastructure was already
   committed by Subagent 2 in a separate commit.

   If this is a retry (fixing a test failure), commit the fix:
   ```bash
   git add -A
   git commit -m "fix(<connector>): fix <brief description of what was wrong>"
   ```

## If this is a retry after test failure:
- Read the error details from the test report
- Diagnose: wrong URL? bad auth? missing field? parse error? wrong status mapping?
- Fix the specific issue in the connector code
- Rebuild and verify
- Do NOT rewrite unrelated code — focus on the specific failure

## Rules

- Follow connector-service naming conventions, NOT euler-api-txns conventions.
  Before introducing ANY new field or type name:
  1. Grep existing .proto files in crates/types-traits/grpc-api-types/proto/
  2. Grep existing Rust domain types in crates/types-traits/domain_types/src/
  3. Use whatever name the codebase already uses for that concept
  4. Only invent a new name if the concept is genuinely new to connector-service
  Gateway-specific serde renames (matching the external API) are fine.

- ResponseRouterData has exactly 2 type params: ResponseRouterData<Response, RouterData>
  (defined at crates/integrations/connector-integration/src/types.rs). NOT 5.

- Do NOT modify any existing connector's working code — only add new code for the
  new flow.

- Use the field naming map from the analysis report for all proto/domain fields.

- You have full authority to modify any layer of code needed: core flow logic,
  connector integrations, pre-call implementations, proto definitions, config.
  Do not stop at partial implementation. Do not rely on manual intervention.
  Automatically implement all missing dependencies.

## Return Format

Return a connector report with:

### CONNECTOR REPORT

**Connector:** <name>
**Target flow implemented:** YES
**Pre-call flows implemented:** [list] or "None needed"
**Commit SHA:** <sha>

**Files modified/created:**
| File | What was changed |
|...|...|

**Build status:** PASS/FAIL
**Clippy status:** PASS/FAIL (zero warnings)
**Fmt status:** PASS/FAIL (no changes)

**Is this a retry?** YES (attempt #N, fixing: <issue>) / NO (first attempt)
**Is this a connector switch?** YES (from <old> to <new>, reason: <mismatch>) / NO

**Any issues or decisions that need human input:** [list or "None"]
```

### Handoff → Subagent 4

Pass the connector report (especially the files modified list, connector name,
and pre-call flows implemented) as context to Subagent 4.

---

## Subagent 4: Testing

**Type:** `general`

**Purpose:** Execute the full pre-call chain via grpcurl, then test the target flow.
MUST achieve HTTP 200 with valid business data. Returns PASS, FAIL, or
CAPABILITY_MISMATCH to the orchestrator.

The task is NOT complete until the connector returns HTTP 200. On failure, the
orchestrator will re-run Subagent 3 → Subagent 4 in a loop until success.

### Prompt Template

```
You are testing a newly implemented flow in connector-service via gRPC.

## Analysis Report
{{PASTE FULL ANALYSIS REPORT FROM SUBAGENT 1 — especially the dependency chain}}

## Connector Report
{{PASTE CONNECTOR REPORT FROM SUBAGENT 3}}

## Repos
- connector-service: /home/kanikachaudhary/Kanika/connector-service/
- euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/

## Workflow Docs
- Testing strategy (§8.1.5 Smoke Test): /home/kanikachaudhary/Kanika/flow-migration-workflow/8_testing_and_operations/8.1_testing_strategy.md

## Tasks

**MANDATORY PRE-TEST VALIDATION (before building the server):**
Before running ANY test, verify that the Connector Report from Subagent 3
confirms ALL pre-call dependencies have been implemented with REAL connector
integrations (not empty stubs). Check:
- Does the connector report list "Pre-call flows implemented: [list]"?
- Does the list match the analysis report's dependency chain?
- If ANY pre-call is missing or was listed as "None needed" when the analysis
  report says otherwise, STOP. Return FAIL to the orchestrator immediately
  with: "Pre-call dependency <flow> was not implemented by Subagent 3.
  Cannot test with dummy values. Re-run Subagent 3 to implement <flow> first."
**NEVER substitute a dummy/hardcoded/placeholder value for an ID, token, or
reference that should come from a pre-call flow. This is the #1 cause of
false test results and is an absolute violation of this workflow.**

Follow section 8.1.5 of the testing strategy doc precisely:

1. **Build and start the gRPC server:**
   ```bash
   export PATH="$HOME/.cargo/bin:$PATH"
   cargo build -p grpc-server
   RUN_ENV=development nohup ./target/debug/grpc-server > /tmp/grpc-server.log 2>&1 &
   sleep 5
   ```

2. **Get connector credentials from creds.json.**
   Credentials for all supported connectors are stored at:
   `/home/kanikachaudhary/Kanika/creds.json`
   - Read this file and find the credentials for <CONNECTOR_NAME>
   - Extract the fields needed for the `x-connector-config` header (e.g.,
     api_key, api_secret, merchant_id, salt_key, salt_index — varies by connector)
   - Build the `x-connector-config` header in serde JSON format:
     `{"config":{"<ConnectorVariant>":{...fields...}}}`
   - If the connector is NOT found in creds.json, THEN ask the user for credentials.
   Do NOT proceed without real credentials. Do NOT use placeholder values.

3. **Verify the correct endpoint URLs BEFORE testing.**
   Before sending any request, cross-reference the URL your implementation uses
   against the Haskell source in euler-api-txns:
   - Read the gateway's Endpoints.hs file (e.g., Gateway/PhonePe/Endpoints.hs)
   - Read the gateway's Flow.hs file for the actual URL used by this specific flow
   - Verify the base URL domain is correct (e.g., PhonePe wallet flows use Mercury/
     pg-sandbox, NOT Hermes/PG)
   - Verify the path is correct (e.g., /v3/merchant/otp/send, /v3/wallet/balance)
   - If the URL in config/development.toml needs to be changed for testing, note
     what you changed — you will need to revert it before pushing.
   - Do this for EVERY flow in the pre-call chain AND the target flow.

4. **Execute the pre-call dependency chain via grpcurl.**
   If the analysis report lists pre-call dependencies:
   - Execute each pre-call flow via grpcurl in dependency order
   - Capture output IDs/tokens from each response (e.g., customer_id from
     CreateCustomer, token_id from SetupMandate)
   - Feed those outputs as inputs to the next call in the chain
   - If a pre-call fails: this is a FAIL — report back to orchestrator with
     the specific error so Subagent 3 can fix the pre-call implementation
   - Do NOT skip pre-calls. Do NOT use dummy/placeholder values.
   - Do NOT use hardcoded test IDs unless they are known-valid connector sandbox
     test fixtures documented in the connector's API docs.

5. **Test the target flow** using real outputs from the pre-call chain.

6. **Evaluate the response — STRICT:**

   **PASS:** HTTP 200 with correct business data. This is the ONLY acceptable
   outcome. The task succeeds.

   **CAPABILITY_MISMATCH:**
   The connector returns "feature not enabled", "recurring not supported",
   "not available in test mode", or similar errors indicating the connector's
   test environment CANNOT support this flow. This is NOT a code bug — it is
   an environment limitation.
   → Return CAPABILITY_MISMATCH to orchestrator with the specific error.
   The orchestrator will pick the next connector and re-run Subagents 3→4.

   **FAIL:**
   Any other non-200: wrong URL, bad auth, parse error, missing field, wrong
   status mapping, gRPC error, "Api Mapping Not Found", request body mismatch,
   credential errors, etc.
   → Return FAIL to orchestrator with:
     - The exact error message
     - The full grpcurl command and response
     - Your diagnosis of the root cause
     - Suggested fix (which file, which function, what to change)
   The orchestrator will re-spawn Subagent 3 to fix, then re-run Subagent 4.

   **A non-200 where pre-calls were skipped or dummy values were used is
   ALWAYS a FAIL. "I used a dummy token" is never a valid result.**

7. **Track any test-only config changes.**
   If you modified config/development.toml (e.g., changed a base_url or
   secondary_base_url to point to a different environment for testing), record:
   - Which file was changed
   - What the original value was
   - What you changed it to
   - WHY it needed to be changed for testing
   These changes MUST be reverted before pushing (handled by Subagent 5).

8. **Stop the server** after testing:
   ```bash
   kill $(pgrep -f grpc-server) 2>/dev/null
   ```

## Return Format

Return a test report with:

### TEST REPORT

**Flow:** <name>
**Connector:** <name>
**Result:** PASS / FAIL / CAPABILITY_MISMATCH
**Attempt:** #N

**Pre-call chain executed:**
| Step | Flow | grpcurl command | Response status | Output captured |
|------|------|-----------------|-----------------|-----------------|
| 1 | <flow> | <cmd> | 200 | customer_id=xxx |
| 2 | <target> | <cmd> | 200 | — |
(or "No pre-calls needed")

**Target flow grpcurl command (with credentials):**
```bash
<full command as executed>
```

**Target flow response:**
```json
<full response JSON>
```

**HTTP status code from connector:** <code>
**Verification notes:** <what was checked>

**If FAIL — root cause diagnosis:**
- Root cause: <what went wrong>
- Suggested fix: <what to change, in which file>
- Files likely affected: <list>

**If CAPABILITY_MISMATCH:**
- Connector: <name>
- Error: <exact error message from connector>
- Reason: <why this connector can't test this flow in its test environment>
- Next connector to try: <from analysis report, or "none remaining">

**Test-only config changes made (to be reverted before push):**
| File | Original value | Test value | Reason |
|---|---|---|---|
| <file> | <original> | <test value> | <reason> |
(or "None" if no config changes were needed)

**Issues:** [list or "None"]
```

### Handoff → Subagent 5 (only on PASS)

Pass the test report as context to Subagent 5. Do NOT hand off on FAIL or
CAPABILITY_MISMATCH — those go back to the orchestrator for retry.

---

## Subagent 5: Push & PR

**Type:** `general`

**Purpose:** Revert any test-only config changes, push the branch, and raise a PR
with a proper title, description, and test evidence. The code is already committed
in separate commits by Subagent 2 (core flow infra) and Subagent 3 (connector
integration).

### Prompt Template

```
You are finalizing and raising a PR for a newly implemented flow in connector-service.

## Connector Report
{{PASTE CONNECTOR REPORT FROM SUBAGENT 3}}

## Test Report
{{PASTE TEST REPORT FROM SUBAGENT 4}}

## Repos
- connector-service: /home/kanikachaudhary/Kanika/connector-service/

## Workflow Docs
- Testing strategy: /home/kanikachaudhary/Kanika/flow-migration-workflow/8_testing_and_operations/8.1_testing_strategy.md

## Tasks

1. **Revert any test-only config changes.**
   Check the test report for "Test-only config changes made". If any config files
   (development.toml, sandbox.toml, production.toml) were modified solely for
   testing (e.g., a base_url was pointed to a different environment), revert those
   changes now.
   ```bash
   git diff config/  # verify what changed
   # If any changes are test-only, revert and commit:
   git checkout config/development.toml
   git add -A
   git commit -m "chore: revert test-only config changes"
   ```

2. **Run pre-commit checks:**
   ```bash
   cargo clippy --all-targets --all-features
   cargo +nightly fmt --all
   ```
   If either produces changes, commit them:
   ```bash
   git add -A
   git commit -m "chore: apply fmt/clippy fixes"
   ```

3. **Verify the commit history** on this branch is clean:
   ```bash
   git log --oneline main..HEAD
   ```
   You should see separate commits for:
   - Core flow infrastructure (from Subagent 2)
   - Connector integration (from Subagent 3)
   - Any retry fixes (from Subagent 3 retries, if applicable)
   - Config revert / fmt fixes (from this subagent, if applicable)

4. **Push the branch:**
   ```bash
   git push -u origin <branch-name>
   ```

5. **Create the PR** using `gh pr create` with the following structure:

   **Title:** `feat(<connector>): add <FlowName> flow for <Connector>`

   **Body must include these sections:**

   ### Summary
   - Brief bullet points from the connector report: what was added, which
     layers were touched, any config changes, any pre-call flows implemented.

   ### Test Results

   ```markdown
   ## How did you test this?

   ### gRPC smoke test against <Connector> preprod — PASS

   **Pre-call chain:**
   <list each pre-call step, or "No pre-calls needed">

   **Request:**
   ```bash
   grpcurl -plaintext \
     -H 'x-connector-config: <REDACTED>' \
     -H 'x-merchant-id: <merchant_id>' \
     -H 'x-tenant-id: default' \
     -H 'x-request-id: test_001' \
     -H 'x-connector-request-reference-id: test_ref_001' \
     -d '<request body — replace only secrets (API keys, salt keys) with placeholders>' \
     localhost:8000 types.PaymentService/<FlowRpcName>
   ```

   **Response (HTTP 200 — SUCCESS):**
   ```json
   <FULL response JSON exactly as returned — do NOT redact any part of the response>
   ```
   ```

   ### Test-Only Config (if applicable)
   If any config was temporarily changed for testing but reverted before commit,
   mention it in the PR so reviewers know:
   ```markdown
   **Note:** During testing, `config/development.toml` had
   `phonepe.secondary_base_url` set to `https://mercury-t2.phonepe.com/` (Mercury
   preprod) to test the wallet endpoint. This has been reverted to the standard
   value. To reproduce the test locally, set this URL in your development config.
   ```

## SECURITY RULES (MANDATORY)

- The `x-connector-config` header contains API keys, salt keys, merchant secrets.
  ALWAYS replace the entire value with `<REDACTED>` in the PR request sample.
- In the request body (`-d` argument), replace only secrets (API keys, salt keys,
  auth tokens) with `<PLACEHOLDER>` descriptors (e.g., `<SALT_KEY>`, `<AUTH_TOKEN>`).
  Non-secret identifiers like merchant_id and phone numbers can stay as-is.
- **Leave the FULL response JSON exactly as returned.** Do NOT redact or placeholder
  any part of the response body — reviewers need the complete raw output
  (rawConnectorRequest, rawConnectorResponse, checksums, base64 payloads, field
  values, headers, everything) to verify correctness.
- Do NOT include any credentials in the commit message.

## Return Format

Return:

### PR REPORT

**PR URL:** <url>
**Branch:** <branch>
**Commits on branch:**
| SHA | Message | Subagent |
|-----|---------|----------|
| <sha> | feat(...): add flow type infrastructure | S2 (Core Flow) |
| <sha> | feat(...): add connector integration | S3 (Connector) |
| <sha> | fix/chore (if any) | S3 retry / S5 |
**PR Title:** <title>
**Test status:** PASS
**Test-only config reverted:** YES (list what) / NO (nothing to revert)
**Pre-call flows included:** [list] or "None"
**Files changed:** <count>
**Lines added/removed:** +<added>/-<removed>
```

---

## Rules (Shared Across All Subagents)

These rules apply to every subagent. Include them in the prompt context when
spawning subagents that write code (Subagents 2, 3, and 5).

- **ONE BRANCH PER FLOW (with dependency exception).** Every flow gets its own
  fresh branch from `main`. Independent flows always get separate branches.
  Exception: pre-call dependency flows required for testing the target flow go
  on the SAME branch — they are prerequisites, not independent features.
  ```bash
  git checkout main && git pull origin main
  git checkout -b feat/<connector>-<flow-name>
  ```
- Do NOT modify any existing connector's working code — only add new code.
- Do NOT remove or modify existing euler_txn gateway adapters.
- If a type mapping is ambiguous, flag it and ask the user — do not guess.
- If the gateway requires encryption (AES, JWE, RSA, etc.), flag it and ask
  for guidance before implementing.
- **Follow connector-service naming conventions, NOT euler-api-txns conventions.**
  Before introducing ANY new field or type name:
  1. Grep existing `.proto` files in `crates/types-traits/grpc-api-types/proto/`
  2. Grep existing Rust domain types in `crates/types-traits/domain_types/src/`
  3. Use whatever name the codebase already uses for that concept
  4. Only invent a new name if the concept is genuinely new to connector-service
  Gateway-specific serde renames (matching the external API) are fine.
- **BEFORE EVERY COMMIT:**
  ```bash
  cargo clippy --all-targets --all-features   # zero errors/warnings
  cargo +nightly fmt --all                     # zero formatting changes
  ```
- Do NOT amend commits or force push unless the user explicitly asks.
- **Separate commits for core flow and connector code.** On the same branch,
  use distinct commits for different layers of work:
  - Subagent 2 commits: core flow infrastructure (flow marker, domain types,
    sub-trait, proto, conversions, gRPC handler, default impls)
  - Subagent 3 commits: connector-specific integration (transformers, URL,
    headers, auth, error handling, config)
  - Retry fix commits: targeted fixes from test failures
  This separation makes PRs easier to review and allows reverting connector
  code without touching flow infrastructure. Never squash these into one commit.
- **Response/request data type reuse:** Do NOT blindly create a new response/request
  data type for every flow. Reuse `PaymentsResponseData` (add a variant) when the flow
  is semantically a payment transaction. Create a standalone struct only when the flow's
  response fields have zero semantic overlap with payment transaction concepts (e.g.,
  wallet balance, OTP tokens, mandate lifecycle). The deciding factor is **semantic
  domain**, not field count. See `4.5_new_flow_type.md` Layer 2 for the full guide.
- **Revert test-only config before commit.** If you changed `config/development.toml`
  (or other config files) to point to a different URL for testing, revert those changes
  before committing. Mention in the PR what was used for testing so reviewers can
  reproduce.
- **Pre-call dependencies go on the same branch.** If the target flow requires
  pre-call flows (e.g., CreateCustomer before MandateStatusCheck), implement
  those dependencies on the same branch. They are prerequisites, not independent
  features.
- **Full implementation authority.** You have full permission to modify any layer
  of code required: core flow logic, connector integrations, pre-call
  implementations, proto definitions, config. Do not stop at partial
  implementation. Do not rely on manual intervention. Automatically implement
  all missing dependencies.
- **No false positives — ZERO TOLERANCE for dummy values.** A "bad request" or
  4xx error must NOT be accepted if the required pre-calls (token creation, entity
  setup, auth steps) were skipped or stubbed. This is the single most important
  rule in this workflow. Violations include:
  - Using a hardcoded/dummy/placeholder ID, token, or reference instead of
    implementing the pre-call that produces it
  - Seeing an empty stub (`impl ... {}`) for a pre-call flow and proceeding
    to test with dummy data instead of implementing the real integration
  - Accepting a 4xx response as "the pre-call isn't available" when the real
    problem is that YOU didn't implement it
  - Declaring a test "passed" or "acceptable" when pre-call outputs were faked
  If you catch yourself about to type a dummy value for a field that should
  come from a pre-call: STOP. Go back and implement the pre-call. There are
  NO shortcuts and NO exceptions. Every ID, token, and reference used in
  testing must come from a real grpcurl call to a real implemented flow.
- **Connector fallback on capability mismatch.** If the chosen connector returns
  "feature not enabled" or similar capability errors in its test environment,
  do not stop. The orchestrator will auto-switch to another connector that
  implements this flow (per the analysis report). The Connector Integration
  subagent will implement the new connector, Testing will re-run the full
  pre-call chain, and this cycle continues until a working connector is found
  or all options are exhausted.
- **Iterate until HTTP 200 — no partial results accepted.** The task is NOT
  complete until the connector returns a successful business response (HTTP 200
  with correct data). On any failure:
  1. Diagnose the root cause (wrong URL, bad auth, missing field, parse error)
  2. Fix the code (via Connector Integration subagent)
  3. Rebuild (clippy + fmt + build)
  4. Retest (via Testing subagent)
  5. Repeat steps 1-4 until 200
  If the connector returns a capability error, switch to the next connector
  and repeat the full cycle. Do NOT stop, do NOT accept partial results, do NOT
  declare the task complete with a non-200 response. The only exit conditions are:
  (a) HTTP 200 received — proceed to Push & PR
  (b) ALL connectors from the analysis report have been tried and ALL returned
      environment-level capability limitations (not code bugs) — escalate to
      user with exhaustive evidence from every connector attempted

---

## Variants

### Dry Run (Analysis Only)

Only run Subagent 1. Skip Subagents 2-5. Return the analysis report to the user.

```
Analyze the <FlowName> flow for migration. Do NOT write any code.

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```

### Gateway-Centric (Migrate Multiple Flows for One Gateway)

When the user specifies a gateway and a list of flows:

```
Migrate flows Authorize, PSync, Capture for Razorpay to connector-service.

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```

The orchestrator runs Subagent 1 once (analyzing all flows for that gateway),
then runs the **full Subagent 2→3→4→5 pipeline once per flow**, each on its own
fresh branch from `main`. Each flow gets its own branch and PR.

### Batch Planning (Multiple Gateways)

```
Plan migrations for HDFC, Paytm, Billdesk. Analysis only, no code.

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```

Runs Subagent 1 for each gateway. Returns a prioritized migration plan.

---

## Quick Start Examples

**Migrate one flow (most common):**
```
Migrate the VerifyOtpForWallet flow to connector-service.

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```

**Analyze first, then decide:**
```
Analyze the CheckBalanceForWallet flow for migration. Do NOT write any code.

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```

**Multiple flows for one gateway:**
```
Migrate flows VerifyOtpForWallet, CheckBalanceForWallet, DebitWallet for PhonePe to connector-service.

Workflow: /home/kanikachaudhary/Kanika/flow-migration-workflow/
euler-api-txns: /home/kanikachaudhary/Kanika/euler-api-txns/
connector-service: /home/kanikachaudhary/Kanika/connector-service/
```
