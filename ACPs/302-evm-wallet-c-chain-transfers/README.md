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
    struct UTXOID {
        bytes32 txID;
        uint32 outputIndex;
    }

    /// Credits each UTXO to its owner and consumes it. Anyone can call this.
    /// The caller pays gas. The owner receives the full amount.
    function importUTXOs(UTXOID[] calldata utxos) external;

    /// Moves msg.value, in whole nAVAX, to shared memory as one UTXO owned by
    /// `to` on `destinationChainID`.
    function exportAVAX(bytes32 destinationChainID, address to) external payable;

    event Imported(address indexed owner, bytes32 txID, uint32 outputIndex, uint64 amountNAVAX);
    event Exported(address indexed from, bytes32 indexed destinationChainID, address to, uint64 amountNAVAX);
}
```

Amounts in shared memory are nAVAX. One nAVAX equals `1e9` wei. The `to` argument holds the 20 owner bytes of a P-Chain or X-Chain address. Both functions accept only the AVAX asset. Import takes no source chain argument. The node looks a UTXO ID up in the P-Chain store and then the X-Chain store.

### Import

An import transaction is just gas exchanged for funds that the source chain already decided to send. The call names UTXO IDs and nothing else. Owner, amount, and locktime come from the UTXO in shared memory. The caller does not need to own anything. The funds go to the UTXO owner, so a third party can import for a contract wallet or for another user and only spends its own gas.

Under SAE, nodes vote on a block before executing it. The node must know which UTXOs a block imports without running its transactions. A transaction declares them in one of two places that the node can read directly:

1. **Direct call.** `to` is the precompile and the calldata is an `importUTXOs` call. The UTXO list in the calldata is the declaration.
2. **Access list.** An [EIP-2930](https://eips.ethereum.org/EIPS/eip-2930) access list entry for the precompile address. Each storage key is `keccak256(txID || outputIndex)`. This lets a contract call `importUTXOs` from any depth, as long as the outer transaction declared the UTXOs.

The block's imported UTXO set is the union of every transaction's declarations. Before a block is accepted, the verifier MUST check for each declared UTXO:

1. It exists in shared memory for the P-Chain or X-Chain of this network and holds a `secp256k1fx.TransferOutput` of the AVAX asset.
2. It has threshold one and one owner address.
3. Its locktime is not after the block timestamp.
4. No other declaration in this block and no processing ancestor block declares it.

A block that fails these checks is not accepted. If the node lacks the source chain data, consensus retries verification when peers vote for the block, so the node catches up when the data arrives. This is the existing behavior for atomic imports. Bootstrapping nodes skip these checks, as they do today, because the network already accepted the block.

At execution, `importUTXOs` MUST revert if any UTXO in the call was not declared by the enclosing transaction. Otherwise it credits each UTXO's owner with its amount times `1e9` wei and emits `Imported`. Owner and amount come from the verifier's result for the block, not from a live shared memory read, so execution is deterministic. After the block executes, the node marks the credited UTXOs consumed in shared memory. A declared UTXO that no call credits stays unconsumed.

### Export

`exportAVAX` MUST revert if `msg.value` is zero, is not a whole number of nAVAX, or exceeds `uint64` nAVAX. It MUST revert if `destinationChainID` is not the P-Chain or X-Chain of this network. It MAY be called from any depth, because the export needs no verification input.

The value stays at the precompile address. After the block executes, the node writes one UTXO to shared memory for the destination chain: owner `to`, threshold one, the amount in nAVAX, no locktime. The UTXO ID is derived from the transaction hash and the log index of the `Exported` event, so historical execution reproduces it.

A forced transfer to the precompile address does not create an export. Only a successful `exportAVAX` call does.

### Gas

| Operation | Gas |
| :- | :- |
| `importUTXOs` base | 20,000 |
| Each imported UTXO | 5,000 |
| Each declared UTXO in an access list | 1,900, the EIP-2930 storage key cost |
| `exportAVAX` | 40,000 |

Import gas covers one shared memory read and one consumption write for each UTXO. The direct-call declaration costs calldata gas only. Export gas covers the UTXO write to shared memory and the shared memory index update. These values are proposed and MUST be reviewed against the reference implementation.

### Removal of atomic transactions

After activation, the C-Chain MUST reject `ImportTx` and `ExportTx` in the mempool and in blocks. Atomic transactions in blocks accepted before activation keep their effect. The `avax.issueTx`, `avax.getAtomicTx`, and `avax.getUTXOs` APIs become read-only or are removed at the implementers' discretion.

Shared memory, the UTXO format, and the P-Chain and X-Chain import and export transactions do not change. A P-Chain `ExportTx` to the C-Chain still names the 20 address bytes of the receiving EVM account as the UTXO owner.

### Replay and state sync

Accepted blocks can be re-executed during bootstrap before the source chain data arrives. The existing shared memory removal markers keep a consumed UTXO from being recreated by a subsequently processed export. Imports replay from the block content alone, because every transaction declares its UTXOs and the verifier fixed their owners and amounts before acceptance.

The precompile has no storage. State sync needs nothing beyond ordinary EVM state.

### Activation

These rules activate in the next C-Chain network upgrade. Before activation, calls to the precompile address behave as calls to an empty account, and atomic transactions keep their current rules.

## Backwards Compatibility

This proposal removes `ImportTx` and `ExportTx` from the C-Chain. Wallets and tools that build them must move to the precompile. Any UTXO exported to the C-Chain before activation stays importable through `importUTXOs`, because shared memory does not change.

MetaMask and the Safe app cannot set an access list today. They import through a direct call, which also covers funds owned by a contract wallet: any account imports them and the contract receives the full amount. A contract that must import inside its own transaction needs a sender that can set an access list. That path is specified now so wallets can adopt it without another upgrade.

X-Chain to C-Chain transfers are supported through the same functions. The X-Chain carries little traffic, so this is for completeness.

Nodes that do not upgrade fail to verify blocks that contain imports and fail to parse the new consensus rules. This is a required upgrade.

## Reference Implementation

The precompile is not implemented yet. An earlier prototype on the [`containerman17/cchain-evm-wallet`](https://github.com/ava-labs/avalanchego/tree/containerman17/cchain-evm-wallet) branch implements the export hook and the settled-state import check with a Solidity helper and an atomic import transaction. The verifier, the shared memory handling, and the export hook carry over. The atomic transaction and the helper contract are replaced by the precompile.

## Security Considerations

The verifier is the trust boundary. A UTXO that passes verification is credited at execution without a second look. The checks above must therefore be complete: existence, asset, threshold, locktime, and no double declaration within processing blocks.

Anyone can import anyone's UTXOs. This is safe because the credit always goes to the UTXO owner and the caller pays all gas. There is no fee taken from the imported amount, so a third party cannot grief the owner with a high fee, unlike the atomic `ImportTx` where the transaction builder set the fee.

Verification reads shared memory, which is written by the P-Chain and X-Chain. A node whose source chain lags cannot verify a block with imports until it catches up. Consensus retries the block, so this costs latency on that node, not safety. If many validators lag at once, C-Chain block acceptance slows until they catch up. This is the same dependency atomic imports have today.

Execution never reads shared memory. Two nodes with different shared memory contents at execution time still compute the same state root, because the block content fixed every amount.

Export funds stay at the precompile address, which has no code path to release them. A balance shortfall cannot occur, because the value arrives with the call.

Removing atomic transactions removes an entire transaction format from the mempool and block verifier. That reduces attack surface. Any tool that still submits them fails at the RPC.

## Open Questions

1. Confirm the precompile address and the gas values against the implementation.
2. Decide whether `avax.*` APIs are removed or kept read-only.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
