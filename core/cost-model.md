# Cost Model

## Cost Model Constants

- **TXN_SIGNATURE_COST**: 720  
  Cost for each signature included in the transaction

- **ED25519_SIGNATURE_COST**: 2400  
  Cost for each precompile signature

- **SECP256K1_SIGNATURE_COST**: 6690  
  Cost for each precompile signature

- **WRITABLE_ACCT_COST**: 300  
  Cost for each writable account

- **INSTR_DATA_BYTES_PER_CU**: 4
  Cost per byte of instruction data

- **BUILTIN_INSTR_COST**: 3000  
  Cost of built-in instructions

- **NONBUILTIN_INSTR_COST**: 200000  
  Cost of non-builtin instructions

- **MAX_COMPUTE_UNIT_LIMIT**: 1400000  
  Maximum compute unit limit

- **HEAP_PAGE_COST**: 8  
  Cost per heap page

- **HEAP_PAGE_SIZE**: 32768  
  Size of one heap page (32KB)

- **MAX_LOADED_ACCT_DATA_SZ**: 64 * 1024 * 1024
  Default size (64MB)

- **SIMPLE_VOTE_COST**: 3428  
  Fixed cost for simple vote transactions

Before the release of feature gate
`reserve_minimal_cus_for_builtin_instructions`
(`C9oAhLxDBm3ssWtJx1yBGzPY55r2rArHmN1pbQn6HogH`) the following fixed
costs were assigned to builtin instructions. After the feature gate is
released, validators will assign `BUILTIN_INSTR_COST` to builtins and
`NONBUILTIN_INSTR_COST` to non-builtins (non-builtings includes
instructions that used to be builtins but were migrated to BPF).

- **PRE_FEATURE_GATE_STAKE_PROGRAM_DEFAULT_COST**: 750
- **PRE_FEATURE_GATE_CONFIG_PROGRAM_DEFAULT_COST**: 450
- **PRE_FEATURE_GATE_ALT_PROGRAM_DEFAULT_COST**: 750
- **PRE_FEATURE_GATE_SYSTEM_PROGRAM_DEFAULT_COST**: 150
- **PRE_FEATURE_GATE_COMPUTE_BUDGET_PROGRAM_DEFAULT_COST**: 150
- **PRE_FEATURE_GATE_BPF_LOADER_UPGRADEABLE_PROGRAM_DEFAULT_COST**: 2370
- **PRE_FEATURE_GATE_BPF_LOADER_DEPRECATED_PROGRAM_DEFAULT_COST**: 1140
- **PRE_FEATURE_GATE_BPF_LOADER_PROGRAM_DEFAULT_COST**: 570
- **PRE_FEATURE_GATE_LOADER_V4_PROGRAM_DEFAULT_COST**: 2000

Before feature gate `ed25519_precompile_verify_strict`
(`ed9tNscbWLYBooxWA7FE2B5KHWs8A6sxfY8EzezEcoo`) the fixed cost for an
ED25519 signature was different.
- **PRE_FEATURE_GATE_ED25519_SIGNATURE_COST**: 2280

Before feature gate `enable_secp256r1_precompile`
(`sr11RdZWgbHTHxSroPALe6zgaT5A1K9LcE4nfsZS4gi`) there was a fixed charge
for SECP256R1 signatures included in the trasaction.
- **PRE_FEATURE_GATE_SECP256R1_SIGNATURE_COST**: 4800


## Messages

### SignatureCost

`SignatureCost` is the cost due to signature verification via builtin verifiers. This cost can be computed deterministically from the transaction payload.

**`SignatureCost` is computed as:**
```
(TXN_SIGNATURE_COST * txn_sig_count) +
(ED25519_SIGNATURE_COST * ed25519_sig_count) +
(SECP256K1_SIGNATURE_COST * secp256k1_sig_count)
```

- `txn_sig_count`: the native signatures included directly int the transaction
- `ed25519_sig_count`: the number of signatures used by ed25519 precompiles
- `secp256k1_sig_count`: the number of signatures used by secp256k1 precompiles


### WriteCost

`WriteCost` is the cost due to having to acquire write-locks on accounts. This cost can be computed deterministically from the transaction payload.

**`WriteCost` is computed as:**
```
WRITABLE_ACCT_COST * num_writable_accts
```

- `num_writable_accts`: The number of accounts in a transaction that are flagged as writable in the transaction payload.

### InstructionDataCost

`InstructionDataCost` is the cost incurred by the instruction data in the transaction. This cost can be computed deterministically from the transaction payload.

**InstructionDataCost is computed as:**  
```
instr_data_sz / INSTR_DATA_BYTES_PER_CU
```

- `instr_data_sz`: Sum of instruction data sizes for all instructions included in the transaction.

### EstimatedExecutionCost

`EstimatedExecutionCost` is the estimated cost for instruction execution. This estimate is provided by a `ComputeBudgetInstruction::SetComputeUnitLimit(u32)` instruction. If this instruction is not present, then default cost values are used. If it can be determined that a transaction will fail before it is executed, then its associated total_requested_execution_cost is zero (for fee-only transactions). This cost can be computed deterministically from the transaction payload.

**EstimatedExecutionCost is determined as:**  

```
compute_instr_is_provided
    ? MIN(instruction_requested_cost, MAX_COMPUTE_UNIT_LIMIT)
    : (BUILTIN_INSTR_COST * num_builtin_instrs) +
      (NONBUILTIN_INSTR_COST * num_nonbuiltin_instrs)
```

- `compute_instr_is_provided`: True iff the transaction includes a `ComputeBudgetInstruction::SetComputeUnitLimit(u32)` instruction

- `instruction_requested_cost`: The requested compute units provided by `ComputeBudgetInstruction::SetComputeUnitLimit(u32)`. This values is silently capped to MAX_COMPUTE_UNIT_LIMIT.

- `num_builtin_instrs`: The number of builtin instructions. The list of builtin instructions is currently based on the the Agave implementation of the protocal, found [here](https://github.com/anza-xyz/agave/blob/master/builtins-default-costs/src/lib.rs).

- `num_nonbuiltin_instrs`: The number of instructions in the transaction that are not builtin instructions.

### ActualExecutionCost

`ActualExecutionCost` is the runtime cost of executing the instructions in a transaction. This cost is known post-execution, and cannot be known in advance.

### EstimatedLoadedAccountsDataCost

`EstimatedLoadedAccountsDataCost` is the estimated cost due to data loaded from the accounts listed in an instruction. This estimate may be computed from the provided data size in the `ComputeBudgetInstruction::SetLoadedAccountsDataSizeLimit(u32)` instruction. This cost can be computed deterministically given the raw transaction bytes.

**EstimatedLoadedAccountsDataCost is calculated as:**  
```
( ( compute_instr_is_provided
      ? MIN(instruction_requested_sz, MAX_LOADED_ACCT_DATA_SZ)
      : MAX_LOADED_ACCT_DATA_SZ
    ) + ( HEAP_PAGE_SIZE - 1 ) / HEAP_PAGE_SIZE ) * HEAP_PAGE_COST
```

- `compute_instr_is_provided`: True iff the transaction includes a `ComputeBudgetInstruction::SetLoadedAccountsDataSizeLimit(u32)` instruction

- `instruction_requested_sz`: The requested loaded accounts data size provided by `ComputeBudgetInstruction::SetLoadedAccountsDataSizeLimit(u32)`.

### ActualLoadedAccountsDataCost
`ActualLoadedAccountsDataCost` is the actual cost due to loaded account data.
This cost is after the transaction loading stage, and can be computed in the same way as `EstimatedLoadedAccountsDataCost`, but using the actual `instruction_requested_sz`.


## Transaction Cost

### EstimatedTransactionCost

`EstimatedTransactionCost` represents the total estimated transaction cost. It is composed by adding the following costs:

- SignatureCost
- WriteCost
- InstructionDataCost
- EstimatedExecutionCost
- EstimatedLoadedAccountsDataCost

This estimate is used heuristically for creating blocks as a leader, and is typically greater than the actual transaction cost, but there are no guarantees provided by this cost and it does not matter for ensuring consensus. This cost can be computed deterministically from the transaction payload.

### ActualTransactionCost
`ActualTransactionCost` represents the final cost of a transaction after execution. It is composed by adding the following costs:

- SignatureCost
- WriteCost
- InstructionDataCost
- ActualExecutionCost
- ActualLoadedAccountsDataCost

This is the value used to compute the total block cost. All nodes in a cluster must agree on this value. This cost is a function of the state of the blockchain, and cannot be known in advance.

### SimpleVoteTransactionCost
SimpleVoteTransactionCost represents a fixed cost for a simple vote transaction. A simple vote transaction has a fixed cost, `SIMPLE_VOTE_COST`, regardless of execution or payload content. In order for a transaction to qualify as a simple vote, it should satisfy the following conditions:

- It has either 1 or 2 signatures
- It is a Legacy transaction
- It has exactly 1 instruction
- ...which is a Vote instruction
