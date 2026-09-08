| ACP | 302 |
| :--- | :--- |
| **Title** | C-Chain Imports and Exports from EVM Wallets |
| **Author(s)** | Ilya Solohin ([@containerman17](https://github.com/containerman17)) |
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

The C-Chain is an EVM chain, but its imports and exports are atomic transactions, not EVM transactions. A generic EVM wallet cannot sign them.

This ACP adds a precompile. An EVM transaction that calls it is the import or the export. The block verifier checks that imported UTXOs exist before the block is accepted, and execution credits their owners. The atomic `ImportTx` and `ExportTx` stay for a transition period and are removed by a later upgrade.

## Motivation

The C-Chain is an EVM chain, but two of its operations are not EVM transactions. Export and import require an Avalanche-specific wallet. A user of a generic EVM wallet or a smart account cannot do them, so the C-Chain is not fully EVM compatible.

The missing operations are:

- Export AVAX from a C-Chain account to a P-Chain or X-Chain address.
- Import AVAX that a P-Chain or X-Chain transaction exported to a C-Chain account.

Staking, delegation, and validator funding are uses of these transfers. This ACP changes the C-Chain half of each transfer. P-Chain imports and exports through EVM wallet interfaces are a subsequent step.

The atomic transaction format is the only reason the C-Chain has a second transaction type, a second signature scheme, a second mempool path, and a second gas model. One precompile makes all of them removable.

### Background

The P-Chain, X-Chain, and C-Chain exchange AVAX through shared memory. An export creates a UTXO there. An import consumes it. A UTXO exported to the C-Chain names 20 owner bytes. Today those bytes are usually the P-style address of a key, `ripemd160(sha256(pubkey))`, and the atomic `ImportTx` signature picks the EVM destination. With this ACP, exports name the EVM address of the receiving account as the owner.

Under [ACP-194](../194-continuous-execution/README.md), consensus accepts C-Chain blocks before execution. A block is verified before acceptance and executed after. The verifier does not run transactions. Anything execution depends on from outside the EVM must be checked at verification from data the verifier can read directly.

Exports already reach shared memory through an execution hook. Imports are the harder direction, because the C-Chain must know that the UTXOs exist before it accepts a block that spends them.

## Specification

### Precompile

A native precompile at `0x0200000000000000000000000000000000000007`:

```solidity
interface ICrossChainTransfer {
    struct UTXOID {
        bytes32 txID;
        uint32 outputIndex;
    }

    /// Imports msg.sender's own UTXOs and credits `to`. Direct calls only.
    /// Reverts if msg.sender does not own every UTXO.
    function importUTXOs(UTXOID[] calldata utxos, address to) external;

    /// Imports UTXOs on behalf of their owners. Direct calls only. Each UTXO
    /// is credited to its owner, who must have allowed remote imports. The
    /// caller pays gas and receives nothing.
    function importForOwners(UTXOID[] calldata utxos) external;

    /// msg.sender allows or forbids anyone to import its UTXOs with
    /// importForOwners. Callable from any depth. Default is forbidden.
    function setRemoteImport(bool allowed) external;

    /// Moves msg.value, in whole nAVAX, to shared memory as one UTXO owned by
    /// `to` on `destinationChainID`. Callable from any depth.
    function exportAVAX(bytes32 destinationChainID, address to) external payable;

    event Imported(address indexed recipient, bytes32 txID, uint32 outputIndex, uint64 amountNAVAX);
    event RemoteImportSet(address indexed owner, bool allowed);
    event Exported(address indexed from, bytes32 indexed destinationChainID, address to, uint64 amountNAVAX);
}
```

Amounts in shared memory are nAVAX. One nAVAX equals `1e9` wei. All functions accept only the AVAX asset. The node looks a UTXO ID up in the P-Chain store and then the X-Chain store, so import takes no source chain argument.

### Import

An import is gas exchanged for funds the source chain already decided to send. The call names UTXO IDs and nothing else. Owner, amount, and locktime come from the UTXO in shared memory.

There are two import calls. `importUTXOs` is the owner importing its own UTXOs. The owner picks the recipient, as the signer of an `ImportTx` does today. `importForOwners` is anyone importing for owners that opted in. The funds go to the owners. The caller only pays gas, so a third party can import for a contract wallet or for another user without touching the funds.

An import transaction has `to` equal to the precompile address and calldata that calls one of the two. Internal calls to either revert. This is what lets the verifier see every import without executing: the UTXO list is in the calldata of a transaction addressed to the precompile.

Before a block is accepted, the verifier MUST collect the UTXO IDs from every import transaction in the block and check for each one:

1. It exists in shared memory for the P-Chain or X-Chain of this network and holds a `secp256k1fx.TransferOutput` of the AVAX asset.
2. It has threshold one and one owner address.
3. Its locktime is not after the block timestamp.
4. No other import transaction in this block and no processing ancestor block names it. A UTXO named by a reverted import therefore becomes importable again once that block settles.

The builder writes the owner, amount, and source chain of each verified UTXO into the block's extra data, where atomic transactions live today. Execution and replay read them from there, never from shared memory.

A block that fails these checks is not accepted. If the node lacks the source chain data, consensus retries verification when peers vote for the block, so the node catches up when the data arrives. This is the existing behavior for atomic imports. Bootstrapping nodes skip these checks, as they do today, because the network already accepted the block.

At execution, `importUTXOs` MUST revert unless `msg.sender` owns every UTXO in the call, and then credits `to`. `importForOwners` MUST revert unless every owner has set `setRemoteImport(true)`, and then credits each owner. The credit is the UTXO amount times `1e9` wei, with one `Imported` event per UTXO. Owner and amount come from the block's extra data, not from a live shared memory read, so execution is deterministic. After the block executes, the node marks the credited UTXOs consumed in shared memory. A UTXO named by a reverted call stays unconsumed.

Only the owner can redirect funds. A third party cannot, and without the remote flag cannot import them at all. The UTXOs then wait in shared memory for the owner, as they do today.

UTXOs with a threshold above one or more than one owner address fail verification. The precompile has no signature path for them, so they stay in shared memory. During the transition they import through `ImportTx` as today. After activation, the P-Chain MUST reject an export to the C-Chain whose output is not a single-owner, threshold-one transfer output, so no new such UTXOs are created.

### Remote import flag

`setRemoteImport` writes one boolean in the precompile's storage under `msg.sender`. It needs no verification input, so it works from any call depth. A contract wallet sets it once and any account can then call `importForOwners` for the contract's UTXOs, which land on the contract.

### Export

`exportAVAX` MUST revert if `msg.value` is zero, is not a whole number of nAVAX, or exceeds `uint64` nAVAX. It MUST revert if `destinationChainID` is not the P-Chain or X-Chain of this network. It works from any call depth, because the export needs no verification input.

The value stays at the precompile address. After the block executes, the node writes one UTXO to shared memory for the destination chain: owner `to`, threshold one, the amount in nAVAX, no locktime. The UTXO ID is derived from the transaction hash and the log index of the `Exported` event, so historical execution reproduces it.

A forced transfer to the precompile address does not create an export. Only a successful `exportAVAX` call does.

### Gas

| Operation | Gas |
| :- | :- |
| `importUTXOs` or `importForOwners` base | 20,000 |
| Each imported UTXO | 5,000 |
| `setRemoteImport` | 20,000 |
| `exportAVAX` | 40,000 |

Import gas covers one shared memory read and one consumption write for each UTXO. `setRemoteImport` is one storage write. Export gas covers the UTXO write to shared memory and the shared memory index update. These values are proposed and MUST be reviewed against the reference implementation.

An import cannot pay for its own gas. Under ACP-194 the sender must cover gas from its balance at admission, and the credit lands at execution.

### Transition and removal of atomic transactions

The precompile activates in the next C-Chain network upgrade. `ImportTx` and `ExportTx` keep their current rules during a transition period, so UTXOs with P-style owners exported before wallets update stay importable the old way.

A later upgrade removes `ImportTx` and `ExportTx` from the mempool and from blocks. That upgrade MAY return any UTXO still in shared memory to the P-Chain side of its owner, since the owner bytes are a valid P-Chain address for the same key. No funds are lost by the removal.

Shared memory, the UTXO format, and the import and export transaction formats of the P-Chain and X-Chain do not change. The only new source-chain rule is the single-owner output requirement above for exports to the C-Chain.

### Replay and state sync

Accepted blocks can be re-executed during bootstrap before the source chain data arrives. The existing shared memory removal markers keep a consumed UTXO from being recreated by a subsequently processed export. Imports replay from the block content alone, because the block's extra data records every owner and amount.

The precompile's only storage is the remote import flags, ordinary EVM state. State sync needs nothing else.

## Backwards Compatibility

Wallets that export to the C-Chain must set the UTXO owner to the receiving EVM address. Exports that still use the P-style owner remain importable through `ImportTx` during the transition period.

MetaMask and similar wallets import with one direct call. A Safe or other contract wallet sets the remote flag once from inside a contract transaction, and any account can then import for it with `importForOwners`. No access list or other transaction field that wallets do not support is needed.

Nodes that do not upgrade fail to verify blocks that contain imports. This is a required upgrade.

## Reference Implementation

The [`containerman17/cchain-evm-wallet`](https://github.com/ava-labs/avalanchego/tree/containerman17/cchain-evm-wallet) branch of AvalancheGo implements this proposal in the SAE C-Chain:

- `vms/saevm/cchain/crosschain`: the precompile, its ABI, gas charges, and the per-block table of verified imports.
- `vms/saevm/cchain/hooks.go`: the transaction filter that checks import calldata against shared memory before a block is built or verified, the extra-data import records, the export debit, and the shared-memory writes after execution.
- `vms/saevm/cchain/tx/extdata.go`: the extra-data encoding with import records, versioned so pre-existing blocks parse unchanged.
- `vms/saevm/cchain/crosschain_test.go`: export, owner import, refused and allowed remote import, verification failure without the UTXO, and replay on live and bootstrapping nodes.
- `tests/e2e/c/evm_wallet_transfers.go`: a network test that exports from the C-Chain with the precompile, imports on the P-Chain, exports back to the EVM address, and imports with the precompile.

The atomic `ImportTx` and `ExportTx` still work on the branch. Their removal is a later upgrade.

## Security Considerations

The verifier is the trust boundary. A UTXO that passes verification is credited at execution without a second look. The checks above must therefore be complete: existence, asset, threshold, locktime, and no double consumption within processing blocks.

Verification reads shared memory, which is written by the P-Chain and X-Chain. A node whose source chain lags cannot verify a block with imports until it catches up. Consensus retries the block, so this costs latency on that node, not safety. If many validators lag at once, C-Chain block acceptance slows until they catch up. This is the same dependency atomic imports have today.

Execution never reads shared memory. Two nodes with different shared memory contents at execution time compute the same state root, because the block content fixed every owner and amount.

Only the owner chooses the recipient, and the caller pays all gas. A third party cannot redirect funds and cannot charge a fee from the imported amount. This removes the fee griefing possible with `ImportTx`, where the transaction builder set the fee. A third party can import only for owners that opted in, and only to those owners.

An export that names owner bytes no key controls burns the funds. This is true today. Wallets prevent it by filling the owner from the receiving account.

Export funds stay at the precompile address, which has no code path to release them. A balance shortfall cannot occur, because the value arrives with the call.

## Open Questions

1. Confirm the precompile address and the gas values against the implementation.
2. Set the length of the transition period before `ImportTx` and `ExportTx` are removed.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
