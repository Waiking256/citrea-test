# Loose Timestamp Verification allows Non-Increasing L2 Block Timestamps

#### Summary
I analyzed the `l2-block-rule-enforcer` module which is responsible for enforcing protocol rules on L2 blocks produced by the sequencer.
This module *should* ensure that every new L2 block has a strictly greater timestamp than the previous one (`current > last`), maintaining a logical progression of time.
However, I observed that the code actually checks `current < last` (failing only if the new timestamp is strictly *less* than the old one).
As a result, the sequencer can propose multiple blocks with the *exact same timestamp*, breaking the assumption of strictly monotonic time and potentially causing ordering ambiguities in time-sensitive logic or indexing services.

#### Affected Contract
- File: `crates/l2-block-rule-enforcer/src/hooks.rs`
- Function: `apply_timestamp_rule`

#### Vulnerability Details
The `L2BlockRuleEnforcer` intends to enforce temporal ordering. However, the comparison logic implementation is insufficient.
Code claims to enforce "timestamp must be greater", but the check is `if current_timestamp < *last_timestamp`.
This effectively allows `current_timestamp == *last_timestamp`.

#### Root Cause
The check misses the equality case.

```rust
// File: crates/l2-block-rule-enforcer/src/hooks.rs:104

fn apply_timestamp_rule(
    &self,
    l2_block: &HookL2BlockInfo,
    last_timestamp: &mut u64,
) -> Result<(), L2BlockHookError> {
    let current_timestamp = l2_block.timestamp();

    // BUG: This allows current_timestamp == *last_timestamp
    if current_timestamp < *last_timestamp {
        return Err(L2BlockHookError::TimestampShouldBeGreater);
    }

    *last_timestamp = current_timestamp;

    Ok(())
}
```

#### Execution Flow

1. **Sequencer** produces Block N with timestamp `T=1000`.
2. **Enforcer** updates `last_timestamp` to `1000`.
3. **Sequencer** produces Block N+1 with timestamp `T=1000`.
4. **Enforcer** checks `if 1000 < 1000`. This is `false`.
5. The block is **accepted**.
6. **Result**: Two distinct blocks exist with the same timestamp, violating strict ordering.

#### Impact
This issue allows the sequencer to stall the "logical clock" of the L2 while continuing to produce blocks.
*   **Time-based logic**: Smart contracts relying on `block.timestamp` changing between blocks for randomness or timing checks (e.g., "cannot interact twice in the same block time") may be bypassed if they assume different blocks imply different times.
*   **Indexing**: External indexers relying on unique (block_height, timestamp) tuples might break or overwrite data.
*   **Protocol Soundness**: Violates the implied guarantee of the rollup's time progression.

#### Standalone Validation

### Validation Test File

```rust
// File: crates/l2-block-rule-enforcer/src/tests/bug_repro.rs
use sov_mock_da::MockDaSpec;
use crate::tests::genesis_tests::{get_l2_block_rule_enforcer, TEST_CONFIG};
use crate::tests::{sc_info_helper, setup_evm};

#[test]
fn test_repro_loose_timestamp() {
    let (l2_block_rule_enforcer, mut working_set) =
        get_l2_block_rule_enforcer::<MockDaSpec>(&TEST_CONFIG);

    setup_evm(&mut working_set);

    let mut l2_block_info = sc_info_helper();
    let timestamp = 1000;
    l2_block_info.set_time_stamp(timestamp);

    println!("=== Step 1: Submitting Block 1 with timestamp {} ===", timestamp);
    let res1 = l2_block_rule_enforcer.end_l2_block_hook(&l2_block_info, &mut working_set);
    assert!(res1.is_ok());

    println!("=== Step 2: Submitting Block 2 with timestamp {} (same as Block 1) ===", timestamp);
    let res2 = l2_block_rule_enforcer.end_l2_block_hook(&l2_block_info, &mut working_set);

    // In a correct implementation, this should fail.
    // In the vulnerable implementation, this will pass.
    if res2.is_ok() {
        println!("Result: Ok(()) - BUG CONFIRMED: Duplicate timestamp accepted");
    } else {
         println!("Result: {:?} - BUG NOT REPRODUCED", res2);
    }
    assert!(res2.is_ok());
}
```

### How to Run

```bash
cargo test -p l2-block-rule-enforcer --features native -- test_repro_loose_timestamp --nocapture
```

### Captured Test Output

```text
$ cargo test -p l2-block-rule-enforcer --features native -- test_repro_loose_timestamp --nocapture

running 1 test
=== Step 1: Submitting Block 1 with timestamp 1000 ===
=== Step 2: Submitting Block 2 with timestamp 1000 (same as Block 1) ===
Result: Ok(()) - BUG CONFIRMED: Duplicate timestamp accepted
test tests::bug_repro::test_repro_loose_timestamp ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 6 filtered out; finished in 0.13s
```

### Log-to-Impact Walkthrough
1.  `Submitting Block 2 with timestamp 1000` shows we are attempting to enforce the rule against the previous timestamp (also 1000).
2.  `Result: Ok(())` proves the function returned successfully instead of returning `L2BlockHookError::TimestampShouldBeGreater`.
3.  `BUG CONFIRMED` confirms the assertion `res2.is_ok()` passed, meaning the system accepted the invalid state transition.

#### Recommended Fix
Update the comparison to use less-than-or-equal (`<=`).

```diff
- if current_timestamp < *last_timestamp {
+ if current_timestamp <= *last_timestamp {
      return Err(L2BlockHookError::TimestampShouldBeGreater);
  }
```
