# SME Collateral Commitment Metadata

`record_sme_collateral_commitment(asset, amount)` in [`escrow/src/lib.rs`](../escrow/src/lib.rs) is a metadata-only Soroban escrow entrypoint. It allows the configured SME address to report collateral metadata for off-chain review, but it does **not** move, reserve, escrow, freeze, or verify any asset on-chain.

## Limitations & Contrast with Custody Flows

> [!WARNING]
> This entrypoint is **metadata-only**. It writes metadata to contract instance storage and emits a Soroban event. It does **not** act as an enforced lien or asset custody mechanism.

To prevent integration risks, integrators must understand how this metadata-only flow contrasts with on-chain asset custody:
- **No Token Transfers:** Calling this function does not transfer any tokens from the SME to the escrow contract, nor does it interact with any token contracts.
- **No Reserve Balances:** It does not freeze or lock any on-chain balances.
- **No Custody Verification:** The escrow contract does not verify that the SME actually owns, holds, or has custody of the referenced asset.
- **No Enforcement or Blocking:** Recording a collateral commitment does not block, gate, or restrict any other contract flows. Specifically, it has no effect on settlement ([`LiquifactEscrow::settle`]), SME withdrawal ([`LiquifactEscrow::withdraw`]), investor claims ([`LiquifactEscrow::claim_investor_payout`]), compliance holds, or any other state transition.

Future versions of the platform that enforce asset movement or custody must introduce distinct API endpoints. Historical records of this self-reported metadata are not proof of custody and must never be treated as proof of locked assets.

## On-chain Behavior

### 1. Authorization
Only the configured SME address (`InvoiceEscrow::sme_address`) is authorized to call this entrypoint. The contract enforces this by calling `sme_address.require_auth()` via the internal helper `load_escrow_require_sme`.

### 2. Validation Rules
The contract validates inputs and state before recording:
- **Positive Amount:** The `amount` parameter must be strictly positive (`amount > 0`). If it is zero or negative, the contract panics with [`EscrowError::CollateralAmountNotPositive`].
- **Non-empty Asset Symbol:** The `asset` parameter must be a non-empty Symbol (`asset != Symbol::new(&env, "")`). If an empty symbol is passed, the contract panics with [`EscrowError::CollateralAssetEmpty`].
- **Monotonic Timestamp on Replacement:** When replacing an existing commitment, the current ledger timestamp from `Env::ledger().timestamp()` must not be earlier than the previously recorded timestamp (`now >= prior_commitment.recorded_at`). This acts as a defense-in-depth against stale out-of-order writes. If the timestamp goes backwards, the contract panics with [`EscrowError::CollateralTimestampBackwards`].

### 3. Storage
The contract writes the metadata record under [`DataKey::SmeCollateralPledge`] in the instance storage. This completely replaces any previously recorded commitment.

The recorded data is represented by the [`SmeCollateralCommitment`] struct:
- `asset`: `Symbol` – the off-chain asset symbol.
- `amount`: `i128` – the reported amount.
- `recorded_at`: `u64` – the Soroban ledger timestamp when the commitment was written.

To retrieve the current record, external callers can use [`LiquifactEscrow::get_sme_collateral_commitment`].

### 4. Event Emission
The contract emits a [`CollateralRecordedEvt`] event upon successful execution, which contains:
- `name`: `Symbol` – hardcoded to `coll_rec` (`symbol_short!("coll_rec")`).
- `invoice_id`: `Symbol` – the invoice ID associated with the current escrow.
- `amount`: `i128` – the newly recorded collateral amount.
- `prior_amount`: `i128` – the previously recorded collateral amount (or `0` if no prior commitment existed). This provides clear replacement semantics for off-chain indexers.

## Test Coverage

The scenarios below are covered by the focused collateral suite in
[`escrow/src/tests/coverage.rs`](../escrow/src/tests/coverage.rs):

| Test | Scenario |
|------|----------|
| `test_collateral_first_record_returns_correct_fields_and_prior_amount_is_zero` | First record returns the correct asset/amount/timestamp; `get_sme_collateral_commitment` reflects it. |
| `test_collateral_first_record_event_prior_amount_is_zero` | `CollateralRecordedEvt` emitted by the first record has `prior_amount = 0`. |
| `test_collateral_replacement_overwrites_stored_value_and_emits_prior_amount` | Replacement overwrites storage; event carries the previous record's amount as `prior_amount`. |
| `test_collateral_backwards_timestamp_rejected` | Replacing with a ledger timestamp earlier than `recorded_at` is rejected with `CollateralTimestampBackwards`; original record is preserved. |
| `test_collateral_same_timestamp_replacement_is_allowed` | Equal timestamps (`now >= prior.recorded_at`) are accepted (monotonic, not strictly increasing). |
| `test_collateral_zero_amount_rejected` | Zero amount is rejected with `CollateralAmountNotPositive`. |
| `test_collateral_negative_amount_rejected` | Negative amount is rejected with `CollateralAmountNotPositive`. |
| `test_collateral_empty_asset_rejected` | Empty asset symbol is rejected with `CollateralAssetEmpty`. |
| `test_collateral_non_sme_caller_rejected` | A caller that is not the SME address is rejected (auth failure). |
| `test_collateral_record_does_not_change_token_balances` | No token balances change on the escrow contract, SME, or admin after recording. |

Additional collateral scenarios (happy-path and validation) are also exercised in:
- [`escrow/src/tests/admin.rs`](../escrow/src/tests/admin.rs) — collateral record in admin-flow baselines.
- [`escrow/src/tests/integration.rs`](../escrow/src/tests/integration.rs) — `test_collateral_record_event_payload_is_metadata_only` and `test_collateral_replacement_event_contains_prior_amount` for full event-payload verification.

## Off-chain Risk-Team Handling

Risk teams and off-chain services must treat the recorded data as self-reported metadata and verify its validity independently.

Recommended verification procedures:
1. **Verify Signer Context:** Confirm the transaction was signed by the correct SME address linked to the invoice.
2. **Resolve Asset Symbol:** Ensure the reported `asset` symbol maps to the correct physical asset or token contract.
3. **Verify Custody Separately:** Confirm custody accounts, statements, and security perfection outside the blockchain.
4. **Reconcile Independently:** Implement any asset-control or settlement actions in separate off-chain systems or dedicated contracts, completely detached from this metadata escrow record.
5. **Clear Labeling:** Label all indexed database fields as `reported_collateral_metadata` rather than implying locked balances or enforceable claims.
