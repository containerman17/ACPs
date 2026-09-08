| ACP | 302 |
| :--- | :--- |
| **Title** | C-Chain Imports and Exports from EVM Wallets |
| **Author(s)** | Ilya Solohin ([@containerman17](https://github.com/containerman17)) |
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

C-Chain imports and exports are atomic transactions, not EVM transactions, so a generic EVM wallet cannot make them. This ACP adds a precompile. An EVM transaction that calls it is the import or the export. The block verifier checks imported UTXOs in shared memory before the block is accepted, and execution credits them from data recorded in the block. `ImportTx` and `ExportTx` stay for a transition period and are removed by a later upgrade.

## Motivation

The C-Chain is not fully EVM compatible: moving AVAX to or from the P-Chain needs an Avalanche-specific wallet. Generic wallets and smart accounts cannot stake, delegate, or fund validators. The atomic transaction format is also the only reason the C-Chain carries a second transaction type, signature scheme, mempool path, and gas model.

### Background

A UTXO exported to the C-Chain names 20 owner bytes. Today those are usually a key's P-style address, and the `ImportTx` signature picks the EVM destination. With this ACP, exports name the receiving EVM address.

Under [ACP-194](../194-continuous-execution/README.md), blocks are verified before acceptance and executed after, and the verifier does not run transactions. Anything execution needs from outside the EVM must be checked at verification from data the verifier can read directly, and recorded in the block so replay never reads shared memory.

## Specification

### Precompile

Address `0x0200000000000000000000000000000000000007`:

```solidity
interface ICrossChainTransfer {
    struct UTXOID { bytes32 txID; uint32 outputIndex; }

    /// msg.sender imports its own UTXOs and credits `to`. Direct calls only.
    function importUTXOs(UTXOID[] calldata utxos, address to) external;

    /// Anyone imports UTXOs whose owners allowed it. Each owner is credited.
    /// Direct calls only.
    function remoteImportUTXOs(UTXOID[] calldata utxos) external;

    /// msg.sender allows or forbids remoteImportUTXOs on its UTXOs.
    /// Any call depth. Default is forbidden.
    function allowRemoteImport(bool allowed) external;

    /// Moves msg.value, in whole nAVAX, to shared memory as one UTXO owned by
    /// `to` on `destinationChainID`. Any call depth.
    function exportAVAX(bytes32 destinationChainID, address to) external payable;

    event Imported(address indexed recipient, bytes32 txID, uint32 outputIndex, uint64 amountNAVAX);
    event RemoteImportAllowed(address indexed owner, bool allowed);
    event Exported(address indexed from, bytes32 indexed destinationChainID, address to, uint64 amountNAVAX);
}
```

One nAVAX is `1e9` wei. Only AVAX is accepted. The node looks a UTXO ID up in the P-Chain store, then the X-Chain store.

### Import

An import transaction has `to` equal to the precompile and calldata calling `importUTXOs` or `remoteImportUTXOs`. Internal calls to either revert. The UTXO list is therefore in the calldata, where the verifier can read it without executing.

Before a block is accepted, for every UTXO named by an import transaction the verifier MUST check:

1. It exists in shared memory for the P-Chain or X-Chain of this network and is a `secp256k1fx.TransferOutput` of AVAX.
2. It has threshold one and one owner address.
3. Its locktime is not after the block timestamp.
4. No other import transaction in this block and no processing ancestor block names it.

The builder records owner, amount, and source chain of each UTXO in the block's extra data, where atomic transactions live today. A block that fails a check is not accepted. A node that lacks the source chain data retries verification when peers vote for the block, as with atomic imports today. Bootstrapping nodes skip the checks, as today.

At execution, `importUTXOs` MUST revert unless `msg.sender` owns every UTXO, and then credits `to`. `remoteImportUTXOs` MUST revert unless every owner has called `allowRemoteImport(true)`, and then credits each owner. Owner and amount come from the block's extra data. The credit is the amount times `1e9` wei, with one `Imported` event per UTXO. After the block executes, the node marks credited UTXOs consumed in shared memory. A UTXO named by a reverted call stays unconsumed and, by rule 4, becomes importable again when that block settles.

UTXOs with more than one owner or a threshold above one fail rule 2 and stay in shared memory. During the transition they import through `ImportTx`. After activation, the P-Chain MUST reject an export to the C-Chain whose output is not a single-owner, threshold-one transfer output.

### Export

`exportAVAX` MUST revert if `msg.value` is zero, not a whole number of nAVAX, or above `uint64` nAVAX, or if `destinationChainID` is not the P-Chain or X-Chain. The value stays at the precompile address, which has no code path to release it. After the block executes, the node writes one UTXO to shared memory for the destination chain: owner `to`, threshold one, the amount in nAVAX, no locktime. The UTXO ID is the transaction hash and the log index of the `Exported` event. A forced transfer to the precompile address does not create an export.

### Gas

| Operation | Gas |
| :- | :- |
| `importUTXOs` or `remoteImportUTXOs` base | 20,000 |
| Each imported UTXO | 5,000 |
| `allowRemoteImport` | 20,000 |
| `exportAVAX` | 40,000 |

Import gas covers one shared memory read and one consumption write per UTXO. Export gas covers the UTXO write and the shared memory index update. An import cannot pay for its own gas: under ACP-194 the sender covers gas at admission and the credit lands at execution.

### Transition

The precompile activates in the next C-Chain upgrade. `ImportTx` and `ExportTx` keep their rules during a transition period, so UTXOs exported with P-style owners before wallets update stay importable. A later upgrade removes both from the mempool and from blocks. That upgrade MAY return any UTXO still in shared memory to the P-Chain side of its owner, since the owner bytes are a P-Chain address for the same key.

Shared memory, the UTXO format, and the P-Chain and X-Chain transaction formats do not change, apart from the single-owner export rule above. Replay and state sync need nothing new: imports replay from the block, consumed UTXOs are protected by the existing shared memory removal markers, and the precompile's only storage is the remote import flags.

## Backwards Compatibility

Wallets exporting to the C-Chain must name the receiving EVM address as owner. MetaMask and similar wallets import with one direct call. A Safe or other contract wallet calls `allowRemoteImport(true)` once from a contract transaction, after which any account can import for it with `remoteImportUTXOs`. Nodes that do not upgrade fail to verify blocks with imports. This is a required upgrade.

## Reference Implementation

The [`containerman17/cchain-evm-wallet`](https://github.com/ava-labs/avalanchego/tree/containerman17/cchain-evm-wallet) branch of AvalancheGo implements this in the SAE C-Chain: the precompile in `vms/saevm/cchain/crosschain`, the verifier filter and extra-data records in `vms/saevm/cchain/hooks.go`, unit tests covering export, owner and remote import, refused imports, and replay on live and bootstrapping nodes, and an e2e test that round-trips AVAX between the C-Chain and P-Chain with ordinary EVM transactions.

## Security Considerations

The verifier is the trust boundary. A UTXO that passes verification is credited at execution without a second look, so the four checks must be complete. Verification reads shared memory, which the P-Chain and X-Chain write. A lagging node cannot verify a block with imports until it catches up, and if many validators lag, block acceptance slows. This is the dependency atomic imports have today. Execution never reads shared memory, so nodes with different shared memory contents compute the same state root.

Only the owner chooses the recipient, and the caller pays all gas. A third party cannot redirect funds, cannot take a fee from them, and can import only for owners that opted in. This removes the fee griefing possible with `ImportTx`, where the transaction builder set the fee. An export naming owner bytes no key controls burns the funds, as today.

## Open Questions

1. Confirm the precompile address and gas values against the implementation.
2. Set the length of the transition period.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
