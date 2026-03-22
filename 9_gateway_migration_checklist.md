# 9. Quick-Reference: Single Gateway Migration Checklist

> **Use this as a standalone checklist when migrating one gateway.**
>
> Copy this checklist, fill in the gateway name, and check off items as you go.
> For detailed instructions on any step, follow the linked documents.

---

## Gateway: `______________`

---

### 1. ANALYZE

> See [3.2_pre_migration_checklist.md](3_planning/3.2_pre_migration_checklist.md) for details

- [ ] Read euler_txn `Gateway/<Name>/Flow.hs` — list all exported functions
- [ ] Read euler_txn `Gateway/<Name>/Transforms.hs` — catalog request/response types
- [ ] Read euler_txn `Gateway/<Name>/Types.hs` — understand data structures
- [ ] Identify all `TxnFlowType` values this gateway handles
- [ ] Document feature toggles affecting this gateway
- [ ] Capture sample request/response pairs from production logs

---

### 2. IMPLEMENT (connector-service)

> See [4.1_create_connector.md](4_phase1_gateway_migration/4.1_create_connector.md) for details

- [ ] Create `connectors/<name>/mod.rs` with `ConnectorCommon`
- [ ] Create `connectors/<name>/transformers.rs` with request/response types
- [ ] Implement **Authorize** flow (`get_url`, `get_headers`, `get_request_body`, `handle_response_v2`)
- [ ] Implement **PSync** flow (status sync)
- [ ] Implement **Capture** flow (if applicable)
- [ ] Implement **Void** flow (if applicable)
- [ ] Implement **Refund** flow
- [ ] Implement **RSync** flow (refund status sync)
- [ ] Register in `connectors.rs`, `types.rs`, `connector_types.rs`, `payment.proto`
- [ ] Add base URLs to `config/*.toml`
- [ ] Write unit tests for all request/response transformations

---

### 3. BRIDGE (euler_txn)

> See [4.2_build_bridge.md](4_phase1_gateway_migration/4.2_build_bridge.md) for details

- [ ] Add gateway to `euler-x/src-generated/Gateway/ConnectorService/Flow.hs` dispatcher
- [ ] Create `euler-x/src-generated/Gateway/ConnectorService/<Name>.hs` bridge handler (if needed)
- [ ] Add feature flag via `isConnectorServiceEnabledForFunction` in `CommonGateway.hs`
- [ ] Implement response conversion back to euler_txn types

---

### 4. TEST

> See [8.1_testing_strategy.md](8_testing_and_operations/8.1_testing_strategy.md) and [8.2_validation_checklist.md](8_testing_and_operations/8.2_validation_checklist.md) for details

- [ ] Unit tests pass in connector-service
- [ ] Integration test with sandbox gateway API
- [ ] Shadow traffic comparison shows 0 discrepancies
- [ ] Latency within acceptable bounds

---

### 5. DEPLOY

> See [4.4_rollout.md](4_phase1_gateway_migration/4.4_rollout.md) for details

- [ ] Deploy connector-service with new connector
- [ ] Deploy euler_txn with bridge + feature flag (disabled)
- [ ] Enable for internal test merchants
- [ ] Enable shadow mode for production traffic
- [ ] Gradual rollout: 1% -> 10% -> 50% -> 100%

---

### 6. CLEANUP

- [ ] Monitor for 2 weeks at 100%
- [ ] Remove old gateway adapter code from euler_txn
- [ ] Update documentation

---

## Sign-Off

| Role | Name | Date | Approved? |
|------|------|------|-----------|
| Developer | | | |
| Reviewer | | | |
| On-call/SRE | | | |

---

## Reference Documents

| Document | Purpose |
|----------|---------|
| [1_orchestrator.md](1_orchestrator.md) | Master execution sequence |
| [3.1_flow_inventory.md](3_planning/3.1_flow_inventory.md) | Gateway tiers and priorities |
| [4.1_create_connector.md](4_phase1_gateway_migration/4.1_create_connector.md) | Step-by-step connector creation |
| [4.2_build_bridge.md](4_phase1_gateway_migration/4.2_build_bridge.md) | Bridge wiring in euler_txn |
| [4.3_per_flow_implementation.md](4_phase1_gateway_migration/4.3_per_flow_implementation.md) | Per-flow trait mapping |
| [7.1_type_mapping.md](7_reference/7.1_type_mapping.md) | Type mapping tables |
| [7.2_code_pattern_translation.md](7_reference/7.2_code_pattern_translation.md) | Haskell -> Rust patterns |
| [7.3_file_references.md](7_reference/7.3_file_references.md) | Key file paths |
| [8.3_rollback_plan.md](8_testing_and_operations/8.3_rollback_plan.md) | Rollback procedure |

---

*Last updated: 2026-03-20*
