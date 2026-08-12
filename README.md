# EVM zkVM Opcode Testing Harness

A hand-crafted suite of **261 edge-case state test fixtures** for validating an EVM implementation running inside a zkVM (built and used against ZKsync's **Airbender** zkVM) against Ethereum's canonical execution semantics on the Cancun fork.

Standard EVM test suites (e.g. the official [execution-spec-tests](https://github.com/ethereum/execution-spec-tests)) cover the happy path well, but zkVM-hosted EVM implementations tend to diverge from a reference client in the *boundaries*: integer wraparound, gas refund/rollback ordering, transient storage lifecycle, warm/cold access-list interactions, and precompile edge cases. This harness is a targeted stress test for exactly those boundaries.

## What it does

Every test in this repo is a self-contained script that:

1. Writes an [Ethereum state-test fixture](https://ethereum.github.io/execution-spec-tests/) (EEST-style JSON — `env` / `pre` / `transaction` / `post`) describing a specific opcode or protocol edge case, into `ethereum-fixtures/stable/state_tests/cancun/eip_edge_cases/`.
2. Runs a Rust-based test runner (`cargo run --bin evm-tester`) against the fixture, executing it on the target EVM implementation and diffing the resulting state (storage, balances, logs) against the expected post-state.

This mirrors the workflow real EVM/zkVM teams use to fuzz for consensus bugs: encode a precise pre-state and calldata/bytecode, predict the exact expected post-state by hand from the Yellow Paper / EIP spec, then check the implementation actually produces it.

```jsonc
// example: ARITHMETIC/ADD/test_add_overflow_wrap.json
{
  "test_add_overflow_wrap[fork_Cancun-state_test]": {
    "_info": { "comment": "ADD max_uint256 + 1 should wrap to 0" },
    "pre": {
      "0x1000...0100": { "code": "0x60017f...ffff01600055" } // PUSH1 1, PUSH32 MAX_UINT256, ADD, SSTORE
    },
    "post": {
      "Cancun": [{ "state": { "0x1000...0100": { "storage": { "0x00": "0x00" } } } }]
    }
  }
}
```

## Coverage

| Category | Tests | What it targets |
|---|---|---|
| `ARITHMETIC` | 34 | `ADD`, `MUL`, `SUB`, `DIV`, `SDIV`, `MOD`, `SMOD`, `ADDMOD`, `MULMOD`, `EXP`, `SIGNEXTEND` — overflow/underflow wrap, div-by-zero, sign edge cases |
| `BITWISE` | 18 | `AND`/`OR`/`XOR` family via `BYTE`, `SHL`, `SHR`, `SAR`, comparisons (`LT`, `GT`, `SLT`, `SGT`, `EQ`, `ISZERO`), `NOT` |
| `CALL` | 55 | `CALL`, `CALLCODE`, `DELEGATECALL`, `STATICCALL`, `CREATE`, `CREATE2`, `SELFDESTRUCT` — value transfer, gas stipends, address collisions |
| `CONTROLFLOW` | 8 | `JUMP`, `JUMPI`, `JUMPDEST`, `PC`, `STOP`, `RETURN`, `REVERT`, `INVALID` |
| `ENVIRONMENT` | 48 | `ADDRESS`, `BALANCE`, `BLOCKHASH`, `CALLDATA*`, `CODE*`, `EXTCODE*`, `LOG0-4`, `ORIGIN`, `NONCE`, `RETURNDATASIZE` |
| `MEMORY` | 23 | `MLOAD`, `MSTORE`, `MSTORE8`, `MSIZE`, `MCOPY`, `SHA3`, memory-expansion gas/word rounding |
| `STORAGE` | 19 | `SLOAD`, `SSTORE`, `TLOAD`, `TSTORE` — dirty-slot gas accounting, transient storage lifecycle |
| `GAS` | 13 | EIP-150 63/64 rule, EIP-2929 warm/cold access costs, EIP-2930 access lists, call value stipend |
| `Transaction-Level` | 7 | Intrinsic gas boundaries, EIP-3529 refund caps, sender nonce/fee behavior on revert |
| `Rollback Ordering` | 7 | State rollback ordering across nested calls/reverts (warmness, refunds, `TSTORE`, return data) |
| `Revert Interaction` | 4 | Interaction between reverts and warm/cold access-list state |
| `Precompile Deviations` | 5 | `ECRECOVER`, `SHA256`, `RIPEMD160` known-vector and failure-mode behavior |
| `VM` | 13 | Account touch/existence semantics, refund counters across nested reverts |
| `Ethereum_Canonical_Semantics` | 3 | Cross-cutting spec conformance (e.g. calls during constructor, calls to nonexistent accounts) |
| `specialcase` | 4 | Bootloader gas refund staging, validation-failure vs. execution-revert gas handling |

**~60 distinct opcodes** exercised across 15 categories, all targeting the Cancun fork (including Cancun-era EIPs: EIP-1153 transient storage, EIP-2929/2930 access lists, EIP-3529 refund reduction, EIP-5656 `MCOPY`).

## Repo structure

```
<OPCODE_FAMILY>/<OPCODE>/test_<scenario>.json   # per-opcode edge cases, e.g. ARITHMETIC/ADD/
<CROSS_CUTTING_CATEGORY>/test_<scenario>.json   # protocol-level behavior, e.g. GAS/, Rollback Ordering/
```

## Running the tests

Each fixture is paired with the exact runner command used to execute it, e.g.:

```bash
cargo run --bin evm-tester -- --update_indexes
cargo run --bin evm-tester -- --path "eip_edge_cases" --name "test_tstore_operand_order[fork_Cancun-state_test]"
```

The fixture is unpacked into the local `ethereum-fixtures` checkout and run through the target EVM binary; the runner reports pass/fail against the expected post-state encoded in the fixture.

## Why this exists

zkVM-hosted EVMs re-implement the entire instruction set from scratch to make it provable, which means every opcode's edge-case behavior — not just its "normal" behavior — has to independently match Ethereum consensus rules, or the zkVM will produce valid proofs for an invalid state transition. This harness was built to methodically walk the opcode set and protocol-level rules (gas accounting, rollback ordering, transient storage, access lists) and pin down exact expected behavior per the Yellow Paper and relevant EIPs, so deviations get caught before they reach production.
