# Security Audit Report

## Scope
- `crates/evm/src/evm/system_contracts/src/Bridge.sol`
- `crates/evm/src/evm/system_contracts/src/BitcoinLightClient.sol`
- `crates/citrea-stf`
- `crates/sequencer`
- `crates/l2-block-rule-enforcer`

*Note: The Clementine Bridge CLI (client-side tool) was not found in this repository and was therefore not audited.*

## Findings

### 1. Loose Timestamp Verification in L2 Block Rule Enforcer (Low)
**Location:** `crates/l2-block-rule-enforcer/src/hooks.rs`
**Description:** The `apply_timestamp_rule` function previously checked `current_timestamp < last_timestamp` to return an error. This implicitly allowed `current_timestamp == last_timestamp`.
**Impact:** Multiple blocks could have the exact same timestamp, potentially causing ambiguity in time-based logic or indexing.
**Fix:** Updated the check to `current_timestamp <= last_timestamp` to enforce strictly increasing timestamps.

### 2. User-Initiated Burning of cBTC (Informational)
**Location:** `Bridge.sol`, `safeWithdraw` function.
**Description:** The `safeWithdraw` function allows a user to burn cBTC by providing a valid signature for a payout transaction. The verification checks that the signature matches the public key in the spent output. If a user provides a payout transaction spending a UTXO *not* controlled by the bridge (but for which they have the key), they can successfully call `safeWithdraw` and burn their cBTC.
**Impact:** The user loses cBTC without the bridge actually facilitating a withdrawal (since the bridge doesn't own the spent UTXO). The bridge funds are safe. This is a user-error/griefing vector against oneself.

### 3. Implicit Deposit Amount Verification (Informational)
**Location:** `Bridge.sol`, `deposit` function.
**Description:** The `deposit` function mints a fixed `depositAmount` of cBTC. It verifies the Bitcoin transaction using `verifySigInTx`. It does not explicitly check the amount in the Bitcoin transaction input because that data is not directly available in the witness. However, it relies on the BIP-341 Schnorr signature verification where the sighash includes `shaAmounts`. The contract enforces that the `shaAmounts` used in verification corresponds to `depositAmount`. Thus, a signature is only valid if the input amount matches `depositAmount`.
**Status:** This design is correct but relies on the subtle behavior of Taproot signatures.

### 4. Dependency on Trusted System Signer for Light Client Updates (Informational)
**Location:** `citrea-stf` / `BitcoinLightClient.sol`
**Description:** The `BitcoinLightClient` contract is updated via transactions from `SYSTEM_SIGNER`. The security of this relies on the STF (`crates/evm/src/evm/executor.rs`) intercepting these transactions and verifying the `ShortHeaderProof` via `shp_provider`.
**Status:** Verified that `verify_system_tx` in `executor.rs` correctly delegates to `shp_provider`, and `shp_provider` (in zk mode) validates the proof against the block info.

## Conclusion
The core components audited appear robust. The timestamp rule was tightened to be strictly increasing. No critical vulnerabilities leading to direct theft or invalid state transitions were found in the scope.
