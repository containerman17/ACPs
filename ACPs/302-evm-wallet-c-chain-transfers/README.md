| ACP | 302 |
| :--- | :--- |
| **Title** | C-Chain Imports and Exports from EVM Wallets |
| **Author(s)** | Ilya Solohin ([@containerman17](https://github.com/containerman17)) |
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

A generic EVM wallet can send AVAX and call contracts on the C-Chain. It cannot sign the C-Chain's atomic `ImportTx` and `ExportTx` through the standard EVM transaction interface.

This ACP adds contract calls for those two operations. Each operation starts with an explicit EVM transaction authorized by the user. An export completes through C-Chain execution hooks. An import requires a subsequent atomic transaction with the same inputs, recipient, and fee that the owner authorized.

The helper stores import authorizations in ordinary EVM state. Anyone can submit the authorized atomic import without a second owner signature. The application usually submits it after the EVM receipt. Nodes have no obligation to keep a queue or complete pending requests.

This proposal is one step to compatibility with standard EVM wallet interfaces across Avalanche. It covers the C-Chain side of AVAX transfers between the C-Chain and P-Chain. Existing transfers with Avalanche signatures stay valid.

## Motivation

Avalanche must support standard EVM wallet interfaces across its user operations. This proposal addresses C-Chain imports and exports, which currently require support for a different transaction format.

The missing operations are:

- Export AVAX from a C-Chain account to a P-Chain address.
- Import AVAX that a P-Chain transaction exported to the C-Chain account.

Staking, delegation, and validator funding are uses of these transfers. This ACP changes the C-Chain half of each transfer. Support for P-Chain imports and exports through EVM wallet interfaces is a subsequent step. For this iteration, users continue to authorize P-Chain operations through a P-Chain wallet.

An exported UTXO alone does not authorize an import. The owner chooses the inputs and fee through an EVM contract call. The helper fixes the recipient to that owner. A submitter can send this same import, but cannot increase its fee or redirect its funds.

The EVM call records the owner's decision using the account's existing authorization mechanism. Binding the complete import prevents a submitter from changing the recipient, input set, or fee.

### Background

The P-Chain and C-Chain exchange AVAX through shared memory. An export creates an unspent transaction output, or UTXO, in that store. An import consumes the UTXO and credits funds on the destination chain.

C-Chain atomic transactions exist outside EVM execution. They have a different encoding, submission API, and credential format. A credential proves authority to spend an input.

Under [ACP-194](../194-continuous-execution/README.md), consensus accepts C-Chain blocks before execution. A subsequent block settles earlier execution results. Import authorization uses the post-execution state of the settled block selected for the containing block.

Exports use the Warp precompile from [ACP-30](../30-avalanche-warp-x-evm/README.md). Imports use contract storage, not Warp messages. The two operations do not require BLS signatures or delivery of Warp messages to a different chain.

## Specification

### Scope and units

The new path supports AVAX transfers between the C-Chain and P-Chain. Each import spends UTXOs with one owner and credits that owner's C-Chain account. X-Chain transfers, other assets, and other recipients are outside this proposal.

Shared memory stores amounts in nAVAX. One nAVAX equals `1e9` wei. Chain identifiers below are Avalanche blockchain IDs, different from EVM chain IDs.

The helper identifies the importing account through `msg.sender`. An EOA calls the helper directly. A contract wallet calls it through the wallet's own authorization rules. The imported funds belong to the account that calls the helper.

### Helper contract

The network specifies one `CChainHelper` contract with this interface. The full source is in [CChainHelper.sol](./CChainHelper.sol) beside this document and matches [the prototype at commit `16c540e1f2`](https://github.com/ava-labs/avalanchego/blob/16c540e1f2d9420e02f9fe4912fe8c4912ef6d37/tests/cchainhelper/CChainHelper.sol).

```solidity
interface ICChainHelper {
    struct UTXO {
        bytes32 txID;
        uint32 outputIndex;
        uint64 amount;
    }

    event ImportAuthorized(bytes32 indexed importHash, bytes unsignedTx);

    /// Exports msg.value, in whole nAVAX, to the P-Chain as a UTXO owned by `to`.
    function exportToP(address to) external payable returns (bytes32 messageID);

    /// Authorizes an import of `imported` to msg.sender with `fee` nAVAX burned.
    function importFromP(uint32 networkID, bytes32 avaxAssetID, UTXO[] calldata imported, uint64 fee)
        external
        returns (bytes32 importHash);

    /// Consensus reads this mapping directly at storage slot 0.
    function authorized(bytes32 importHash) external view returns (bool);
}
```

The helper has no administrator, upgrade function, or withdrawal function. Its address, deployed code, storage layout, and deployment transaction are protocol constants for each network.

The `to` argument contains the 20 owner bytes from a P-Chain address. Its Solidity type does not require that owner to control an EVM account. For imports, the P-Chain export must name the importing account's 20 EVM address bytes as the UTXO owner.

### Export authorization and execution

The helper MUST reject zero value and values that are not whole nAVAX. The amount in nAVAX MUST fit in a `uint64`.

The helper keeps the call's value. It calls `sendWarpMessage` at `0x0200000000000000000000000000000000000005` with this payload:

```text
P-Chain owner: 20 bytes
amount:         8 bytes, uint64, big-endian, nAVAX
```

The Warp message wraps this payload in an `AddressedCall`. Its `SourceAddress` identifies the helper.

After activation, the node processes export messages from successful calls in receipt order. It MUST check the Warp precompile address, event signature, helper source, and a payload length of 28 bytes.

For each export, the node MUST:

1. Decode the owner and amount.
2. Check that the helper's balance covers `amount * 1e9` wei.
3. Subtract that value from the helper's balance before committing EVM state.
4. Create the related UTXO in shared memory for the P-Chain.

The UTXO has these fields:

```text
TxID:        hash of the EVM transaction containing the export
OutputIndex: index of the export log within the block
AssetID:     AVAX asset ID
Output:      secp256k1fx.TransferOutput
Amount:      exported nAVAX
Locktime:    0
Threshold:   1
Addresses:   [P-Chain owner]
```

The node indexes the UTXO by its owner. An existing P-Chain `ImportTx` can consume it.

The helper must keep sufficient value to cover each export it emits. The node MUST NOT skip a required debit or create an unfunded UTXO. A balance shortfall is an execution error and can stop progress after consensus accepts the block.

Historical execution MUST reproduce the balance debit. Recovery MUST NOT apply the same shared-memory export two times. The helper's total balance need not be zero after execution. Forced transfers can add funds without authorizing exports.

### Import authorization

`importFromP` is nonpayable. It MUST reject an empty input list, duplicate inputs, and inputs outside ascending `(txID, outputIndex)` order. The helper MUST reject overflow when it adds amounts. The sum of input amounts MUST exceed `fee`. The caller supplies the network ID and AVAX asset ID. The helper does not check them. Wrong values produce an `ImportTx` that the atomic verifier rejects, so the caller only loses gas.

The helper constructs an unsigned C-Chain `ImportTx` using the existing atomic codec:

- The network ID and destination blockchain ID identify this C-Chain.
- The source blockchain ID is the P-Chain ID.
- Each input contains its UTXO reference, supplied amount, AVAX asset ID, and signature indices `[0]`.
- The transaction has one output to `msg.sender`, with amount `sum(inputs) - fee` and asset ID AVAX.

Let `U` be the complete canonical unsigned bytes, including the codec version and transaction type ID. The helper computes `H = keccak256(U)` and writes `authorized[H] = true`. It emits `ImportAuthorized(H, U)` and returns `H`.

The helper MUST store `authorized` as a `mapping(bytes32 => bool)` at storage slot zero. The authorization for `H` occupies this slot:

```text
K = keccak256(H || uint256(0))
```

Both arguments to the outer hash are 32 bytes. The complete 32-byte value at `K` MUST equal one for an import to pass authorization. This uses Solidity's [mapping layout](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html#mappings-and-dynamic-arrays).

The node reads this slot directly. It does not execute a contract call during atomic verification. The event supplies bytes for submission, but is not evidence of authorization.

The helper does not read shared memory. A successful call proves authorization, but does not prove that the referenced UTXOs exist or that the import can execute.

### Import credential and validity

The new credential has no fields:

```go
type ContractCredential struct{}
```

The proposed codec type ID is `10`, after `secp256k1fx.Credential`. Each credential encodes only its four-byte type ID, with no payload or payload-length field. The transaction keeps the existing four-byte credential count and one credential for each input.

An import using this path MUST use `ContractCredential` for each input. Mixed credential types are invalid. Existing imports that use only `secp256k1fx` credentials do not change.

During live block verification, the import MUST satisfy the existing structural, asset, fee, and double-spend rules. It MUST also satisfy all these conditions:

1. The new credential is active for the containing block.
2. The transaction imports AVAX from the P-Chain.
3. The helper's authorization slot for those unsigned bytes equals one in the containing block's settled state.
4. The transaction has one AVAX output only.
5. Each referenced UTXO exists in shared memory and contains a `secp256k1fx.TransferOutput`.
6. Each UTXO has threshold one and one address only, equal to the transaction's output address.
7. Each input amount equals its UTXO amount, and its signature indices are `[0]`.
8. Each UTXO's `Locktime` is no greater than the containing block's timestamp, in Unix seconds.

The verifier MUST calculate the approval slot and read its value one time for each transaction, not one time for each input. It MUST use the post-execution state root of the settled block selected by the containing block. A locally executed but unsettled approval MUST NOT authorize inclusion. The time-lock check MUST NOT use a node's wall clock.

Mempool admission can use the latest executed state and local time. Admission does not authorize inclusion in a block. Block building and block verification MUST apply the settled-state and block-time rules.

The atomic import consumes its inputs and credits its output. Consumption prevents a second import from spending the same UTXOs. The approval stays in contract storage after execution.

### Submission and execution

Anyone MAY submit the authorized import through the existing atomic API, mempool, and gossip path. The application usually submits it through `avax.issueTx` after the EVM receipt. This second protocol transaction requires no second owner signature or wallet confirmation.

Nodes MAY submit imports that they see in helper events. This is an optional convenience. Nodes have no obligation to keep pending requests, retry them, or reconstruct them after restart. Block validity MUST NOT depend on a local queue or knowledge of the authorization event.

A submitter can reconstruct the same bytes from the helper call or its event. No private key is needed for submission. A retry preserves the inputs, recipient, and fee. Pool rejection or eviction does not revoke the authorization.

The EVM receipt confirms authorization. It does not confirm that the import executed. Mempool admission also does not confirm that. Only execution of the atomic import completes it. An application that closes before submission must submit the import at a subsequent time or depend on a different submitter.

### Fees

An import has two costs:

- The EVM transaction pays gas for the helper call, including the storage write and event.
- The atomic transaction burns the authorized `fee` from the imported funds.

A directly calling EOA needs sufficient C-Chain AVAX to pay for the helper call. The pending UTXOs cannot pay that call's gas through this proposal.

For an import with `N` inputs and `L` unsigned bytes, this draft proposes:

```text
atomic gas = 10,000 + L + 1,000 * N + 4,700
gas fee cap = floor(fee * 1e9 / atomic gas)
```

The existing intrinsic, byte, and input charges stay. The additional 4,700 gas charges one cold account access and one cold storage read. The gas fee cap must not be less than the containing block's base fee. Existing atomic size limits and block budgets also apply.

The complete authorized fee burns when the atomic import executes. A submitter cannot increase it when the base fee increases. The owner can authorize a different import, with a different fee, through a new EVM transaction.

A new authorization does not revoke an earlier authorization. Either one can execute while its inputs stay unspent. After one consumes an input, any conflicting import becomes invalid. Retrying the same import requires no new EVM call and does not charge that call's gas again.

### Independent chain progress and replay

This path does not require P-Chain and C-Chain heights or block intervals to match. Settlement refers only to C-Chain execution.

The P-Chain export and C-Chain authorization can arrive in any order. An authorization with missing inputs does not prevent unrelated C-Chain transactions from executing. It stays an approval, not work that accepted-block execution must complete.

During live operation, nodes MUST check shared-memory inputs before accepting a block that contains the atomic import. A node without the required P-Chain data cannot verify that block at this time. If sufficient validators do not have that data, C-Chain progress can be delayed. This proposal keeps the existing cross-chain verification dependency.

Execution MUST derive the balance credit from the atomic transaction recorded in the accepted C-Chain block. It MUST NOT wait for P-Chain progress or choose the credit from current shared-memory contents.

Existing bootstrap rules permit replay of accepted imports before the related P-Chain exports. Shared memory records a removal marker for an input that is not present. The subsequent export clears that marker without recreating a spendable UTXO.

Historical execution MUST reproduce the recorded balance changes without reading the imported UTXOs. Recovery MUST preserve the existing rules for applying shared-memory changes one time.

### Activation and state recovery

These rules activate in the next C-Chain network upgrade. Parameters at activation:

| Parameter | Value |
| :- | :- |
| Credential type ID | `10` |
| Extra atomic gas for the approval read | `4,700` |
| Helper address | Set per network at deployment |

Before activation, nodes MUST reject the new credential and MUST NOT process helper messages as exports. The activation plan MUST prevent successful helper calls before activation.

The upgrade must set the helper address, code, storage layout, and deployment transaction for each network. The contract has no constructor arguments, so one deployment transaction gives the same address on each network. Nodes MUST NOT select the helper through operator configuration.

Authorizations are ordinary contract storage. Restart, replay, and state sync preserve them with EVM state. Verification does not require earlier helper receipts, a Warp message database, or a pending-request database.

A syncing implementation MUST make the required settled EVM state available before verifying new blocks. This proposal adds no authorization-history recovery mechanism. It does not complete or replace the C-Chain's general SAE state-sync implementation.

## Backwards Compatibility

The proposal changes C-Chain consensus rules and requires a coordinated upgrade. Existing `secp256k1fx` imports and exports keep their current rules. P-Chain consensus and transaction formats do not change.

Applications must supply the correct destination owner bytes. Generic EVM wallets need only contract-call support for the new C-Chain authorization steps. The application also needs the atomic submission API.

Atomic transaction decoders must support the new credential. Execution clients must apply the export debit to reproduce the C-Chain state. No new consensus queue or required submission service is added.

## Security Considerations

Contract storage gives authorization the same persistence and state-root authentication as other EVM state. An empty credential selects the new verification rule. It carries no proof because the verifier already has the settled state.

Permissionless submission keeps the existing atomic machinery for UTXO checks, conflict checks, fees, and consumption. The application can submit after the receipt without a second wallet confirmation. Mandatory node queues are not needed for block verification or authorization.

The helper does not read live P-Chain data during EVM execution. Such a read could make an accepted block's result depend on that node's P-Chain progress. A subsequent atomic import preserves the existing separation between verification and execution.

### Authorization and fees

The helper binds the import recipient to `msg.sender`. The verifier checks that each input belongs to that recipient. Approval of a different transaction, storage in a different contract, and an event without the related settled approval are not sufficient.

The credential MUST NOT bypass amount, ownership, or time-lock restrictions. A submitter can choose when to submit an authorized import, but cannot increase its fee. This proposal adds no cancellation or expiration. Authorization therefore permits subsequent submission while the inputs stay unspent.

### Helper integrity and export funds

The node trusts the helper's code and storage layout. An incorrect helper implementation can authorize an invalid owner or report an unfunded export. These constants require the same review as other consensus rules.

Successful exports debit the same amount that the user supplied. Reverted calls produce no export. Unexpected funds in the helper do not authorize more exports.

The helper has no receive function. That does not prevent forced transfers. [EIP-6780](https://eips.ethereum.org/EIPS/eip-6780) preserves the transfer of funds through `SELFDESTRUCT`. Such funds can stay in the helper.

### Storage and verification costs

Each different approval occupies an ordinary storage entry paid for by the caller's EVM gas. The helper does not clear approvals. Abandoned or invalid requests therefore keep their storage, as in other contracts.

This proposal requires no special retention policy. Gas prices the writes but does not impose a fixed bound on total storage. Clearing only successful approvals would not make such a bound.

Atomic credentials add four bytes for each input, without copying the unsigned transaction into each credential. The verifier hashes the transaction one time for authorization and reads one storage slot. Contract execution and its event consume EVM gas. No linear-cost claim applies to the Solidity loop, which copies its growing byte array on each iteration.

### Delayed or rejected imports

A helper call can succeed when an input is missing, spent, locked, or incorrectly described. The atomic verifier rejects an invalid import. The EVM call's gas is spent.

Missing P-Chain data, fees that are not sufficient, or mempool capacity can delay the import. No node must retry the request. Optional submission services can apply their own limits without changing authorization or block validity.

These rules do not wait for P-Chain data inside accepted-block execution. They do not guarantee uninterrupted C-Chain progress. Live verification depends on nodes processing the required P-Chain exports.

## Reference Implementation

The [prototype branch](https://github.com/ava-labs/avalanchego/tree/containerman17/cchain-evm-wallet) implements contract calls for both transfer directions.

The main files are:

- `tests/cchainhelper/CChainHelper.sol`
- `vms/saevm/cchain/tx/contract_credential.go`
- `vms/saevm/cchain/hooks.go`
- `vms/saevm/sae/block_builder.go`
- `vms/saevm/cchain/contract_import_test.go`
- `evmwallet_demo/`

The prototype stores approvals in contract storage, uses empty marker credentials, reads settled state, and checks time locks against the block timestamp. The demo application submits the atomic import after the EVM receipt.

Tests cover fee and input binding, ownership, time locks, settlement, restart, duplicate spending, and replay with delayed P-Chain data.

The prototype selects the helper through the node's `helper-address` configuration. Production nodes must not do this, see [Open Questions](#open-questions).

## Open Questions

1. Specify the helper's deployment method, address, and deployment transaction for each network, and the activation gate that rejects helper calls before the upgrade.
2. Complete C-Chain SAE state sync. The prototype skips it.
3. Allocate the credential type ID and review the proposed gas charges against the final implementation.
4. Specify how applications find an import's execution status from its atomic transaction ID.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
