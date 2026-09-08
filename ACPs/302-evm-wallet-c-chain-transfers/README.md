| ACP | 302 |
| :--- | :--- |
| **Title** | C-Chain Imports and Exports from EVM Wallets |
| **Author(s)** | Ilya Solohin ([@containerman17](https://github.com/containerman17)) |
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

A generic EVM wallet can send AVAX and call contracts on the C-Chain. It cannot sign the C-Chain's atomic `ImportTx` and `ExportTx`, because those are not EVM transactions.

This ACP replaces the C-Chain atomic transactions with a precompile. An EVM transaction that calls the precompile is the import or the export. Nothing else is signed or submitted. The block verifier checks that the imported UTXOs exist before the block is accepted. Execution credits the caller and consumes the UTXOs.

After activation, the C-Chain rejects `ImportTx` and `ExportTx`. The P-Chain and X-Chain sides do not change.

## Motivation

The C-Chain is an EVM chain, but two of its operations are not EVM transactions. Export and import require an Avalanche-specific wallet. A user of a generic EVM wallet or a smart account cannot do them, so the C-Chain is not fully EVM compatible.

The missing operations are:

- Export AVAX from a C-Chain account to a P-Chain or X-Chain address.
- Import AVAX that a P-Chain or X-Chain transaction exported to the C-Chain account.

Staking, delegation, and validator funding are uses of these transfers. This ACP changes the C-Chain half of each transfer. P-Chain imports and exports through EVM wallet interfaces are a subsequent step.

The atomic transaction format is the only reason the C-Chain needs a second transaction type, a second signature scheme, a second mempool path, and a second gas model. One precompile removes all of them.

### Background

The P-Chain, X-Chain, and C-Chain exchange AVAX through shared memory. An export creates a UTXO there. An import consumes it.

Under [ACP-194](../194-continuous-execution/README.md), consensus accepts C-Chain blocks before execution. A block is verified before acceptance and executed after. Anything execution reads must be fixed at verification, or nodes can compute different state roots.

Exports already reach shared memory through an execution hook. Imports are the harder direction, because the C-Chain must know that the UTXOs exist before it accepts a block that spends them.

## Specification

### Precompile

A native precompile at `0x0200000000000000000000000000000000000007` with this interface:

```solidity
interface ICrossChainTransfer {
    struct UTXO {
        bytes32 txID;
        uint32 outputIndex;
        uint64 amount;
    }

    /// Credits msg.sender with the sum of `utxos` and consumes them.
    /// `sourceChainID` is the P-Chain or X-Chain blockchain ID.
    function importUTXOs(bytes32 sourceChainID, UTXO[] calldata utxos) external returns (uint256 amountWei);

    /// Moves msg.value, in whole nAVAX, to shared memory as one UTXO owned by
    /// `to` on `destinationChainID`.
    function exportAVAX(bytes32 destinationChainID, address to) external payable;

    event Imported(address indexed to, bytes32 indexed sourceChainID, uint256 amountWei);
    event Exported(address indexed from, bytes32 indexed destinationChainID, address to, uint64 amountNAVAX);
}
```

Amounts in shared memory are nAVAX. One nAVAX equals `1e9` wei. The `to` argument holds the 20 owner bytes of a P-Chain or X-Chain address. Both functions accept only the AVAX asset.

### Import

An import is a transaction whose `to` is the precompile address and whose calldata calls `importUTXOs`. Internal calls to `importUTXOs` revert. Verification must see every import in the block, and it can only see direct calls.

Before a block is accepted, for each import transaction in the block the verifier MUST check:

1. `sourceChainID` is the P-Chain or X-Chain of this network.
2. The list is not empty, is sorted by `(txID, outputIndex)`, and has no duplicates.
3. Each UTXO exists in shared memory for that source chain and holds a `secp256k1fx.TransferOutput` of the AVAX asset.
4. Each UTXO has threshold one, one owner address, and that address equals the transaction sender.
5. Each UTXO locktime is not after the block timestamp.
6. Each `amount` equals the UTXO amount.
7. No UTXO is consumed by an earlier transaction in this block or by any processing ancestor block.
8. The transaction gas limit is not less than the import gas cost below.

A block that fails these checks is not accepted. If the node lacks the source chain data, consensus retries verification when peers vote for the block, so the node catches up when the data arrives. This is the existing behavior for atomic imports.

Execution credits the sender with the sum of `amount` times `1e9` wei, marks the UTXOs consumed in shared memory, and emits `Imported`. Execution MUST NOT read shared memory for anything the verifier did not already check. The call reverts only if the transaction runs out of gas. Revert leaves the UTXOs unconsumed.

### Export

`exportAVAX` MUST revert if `msg.value` is zero, is not a whole number of nAVAX, or exceeds `uint64` nAVAX. It MUST revert if `destinationChainID` is not the P-Chain or X-Chain of this network. It MAY be called from any depth, because the export needs no verification input.

The value stays at the precompile address. After the block executes, the node writes one UTXO to shared memory for the destination chain: owner `to`, threshold one, the amount in nAVAX, no locktime. The UTXO ID is derived from the transaction hash and the log index of the `Exported` event, so historical execution reproduces it.

A forced transfer to the precompile address does not create an export. Only a successful `exportAVAX` call does.

### Gas

| Operation | Gas |
| :- | :- |
| `importUTXOs` base | 20,000 |
| Each imported UTXO | 5,000 |
| `exportAVAX` | 40,000 |

Import gas covers one shared memory read and one consumption write for each UTXO. Export gas covers the UTXO write to shared memory and the shared memory index update. These values are proposed and MUST be reviewed against the reference implementation.

### Removal of atomic transactions

After activation, the C-Chain MUST reject `ImportTx` and `ExportTx` in the mempool and in blocks. Atomic transactions in blocks accepted before activation keep their effect. The `avax.issueTx`, `avax.getAtomicTx`, and `avax.getUTXOs` APIs become read-only or are removed at the implementers' discretion.

Shared memory, the UTXO format, and the P-Chain and X-Chain import and export transactions do not change. A P-Chain `ExportTx` to the C-Chain still names the 20 address bytes of the receiving EVM account as the UTXO owner.

### Replay and state sync

Accepted blocks can be re-executed during bootstrap before the source chain data arrives. The existing shared memory removal markers keep a consumed UTXO from being recreated by a subsequently processed export. Imports replay from the block content alone, because the verifier fixed every amount before acceptance.

The precompile has no storage. State sync needs nothing beyond ordinary EVM state.

### Activation

These rules activate in the next C-Chain network upgrade. Before activation, calls to the precompile address behave as calls to an empty account, and atomic transactions keep their current rules.

## Backwards Compatibility

This proposal removes `ImportTx` and `ExportTx` from the C-Chain. Wallets and tools that build them must move to the precompile. Any UTXO exported to the C-Chain before activation stays importable through `importUTXOs`, because shared memory does not change.

Smart contract wallets cannot import through an internal call in this version. The owner bytes in the UTXO must match an account that sends the transaction directly.

X-Chain to C-Chain transfers are supported through the same functions. The X-Chain carries little traffic, so this is for completeness.

Nodes that do not upgrade fail to verify blocks that contain imports and fail to parse the new consensus rules. This is a required upgrade.

## Reference Implementation

The precompile is not implemented yet. An earlier prototype on the [`containerman17/cchain-evm-wallet`](https://github.com/ava-labs/avalanchego/tree/containerman17/cchain-evm-wallet) branch implements the export hook and the settled-state import check with a Solidity helper and an atomic import transaction. The verifier, the shared memory handling, and the export hook carry over. The atomic transaction and the helper contract are replaced by the precompile.

## Security Considerations

The verifier is the trust boundary. A UTXO that passes verification is credited at execution without a second look. The checks above must therefore be complete: existence, asset, owner, threshold, locktime, amount, and no double consumption within processing blocks.

Verification reads shared memory, which is written by the P-Chain and X-Chain. A node whose source chain lags cannot verify a block with imports until it catches up. Consensus retries the block, so this costs latency on that node, not safety. If many validators lag at once, C-Chain block acceptance slows until they catch up. This is the same dependency atomic imports have today.

Execution never reads shared memory. Two nodes with different shared memory contents at execution time still compute the same state root, because the block content fixed every amount.

Export funds stay at the precompile address, which has no code path to release them. A balance shortfall cannot occur, because the value arrives with the call.

Removing atomic transactions removes an entire transaction format from the mempool and block verifier. That reduces attack surface. Any tool that still submits them fails at the RPC.

## Open Questions

1. Confirm the precompile address and the gas values against the implementation.
2. Decide whether `avax.*` APIs are removed or kept read-only.
3. Decide whether smart contract wallets get an import path in a later revision, for example a declared import list checked by the verifier.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
