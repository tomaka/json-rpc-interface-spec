# chainHead_unstable_follow

**Parameters**:

- `runtimeUpdates`: A boolean indicating whether the events should report changes to the runtime.

**Return value**: String containing an opaque value representing the subscription.

This functions lets the JSON-RPC client track the state of the head of the chain: the finalized, non-finalized, and best blocks.

## Usage

This function works as follows:

- When called, returns an opaque `followSubscription` that can used to match events and in various other `chainHead`-prefixed functions.

- Later, generates an `initialized` notification (see below) containing the hash of the current finalized block, and, if `runtimeUpdates` is `true`, the runtime specification of the runtime of the current finalized block.

- Afterwards, generates one `newBlock` notification (see below) for each non-finalized block currently in the node's memory (including all forks), then a `bestBlockChanged` notification. The notifications must be sent in an ordered way such that the parent of each block either can be found in an earlier notification or is the current finalized block.

- When a new block arrives, generates a `newBlock` notification. If the new block is also the new best block of the node, also generates a `bestBlockChanged` notification.

- When the node finalizes a block, generates a `finalized` notification indicating which blocks have been finalized and which blocks have been pruned.

- If the node is overloaded and cannot avoid a gap in the notifications, or in case of a warp syncing, or if the maximum number of pinned blocks is reached (see below), generates a `stop` notification indicating that the subscription is now dead and must be re-created. No more notifications will be sent out on this subscription.

**Note**: This list of notifications makes it very easy for a JSON-RPC client to follow just the best block updates (listening to just `bestBlockChanged` events) or follow just the finalized block updates (listening to just `initialized` and `finalized` events). It is however not possible to easily figure out whether the runtime has been modified when these updates happen. This is not problematic, as anyone using the JSON-RPC interface naively propably doesn't need to account for runtime changes anyway.

## The `runtimeUpdates` parameter

If the `runtimeUpdates` parameter is `true`, then blocks shouldn't (and can't) be reported to JSON-RPC clients before the JSON-RPC server has finished obtaining the runtime specification of the blocks that it reports. This includes the finalized block reported in the `initialized` event.

If `runtimeUpdates` is `false`, then the `initialized` event must be sent back quickly after the function returns. If `runtimeUpdates` is `true`, then the JSON-RPC server can take as much time as it wants to send back the `initialized` event.

For this reason, blocks might be reported more quickly when `runtimeUpdates` is `false`.

**Note**: It is unlikely that high-level UIs built on top of a JSON-RPC client can do anything before the JSON-RPC server has access to the runtime. Consequently, they should consider the time before the `initialized` event is generated as a loading time. During this loading time, the JSON-RPC server might be performing downloads and CPU-intensive operations. This loading time can be considered as a replacement for the `isSyncing` field of the legacy `system_health` JSON-RPC call.

If a JSON-RPC client wants to be sure to receive an `initialized` event quickly but is also interested in the runtime, it is encouraged to create two subscriptions: one with `runtimeUpdates: true` and one with `runtimeUpdates: false`.

## Notifications format

This function will later generate one or more notifications in the following format:

```json
{
    "jsonrpc": "2.0",
    "method": "chainHead_unstable_followEvent",
    "params": {
        "subscription": "...",
        "result": ...
    }
}
```

Where `subscription` is the value returned by this function, and `result` can be one of:

### initialized

```json
{
    "event": "initialized",
    "finalizedBlockHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "finalizedBlockRuntime": ...
}
```

The `initialized` event is always the first event to be sent back, and is only ever sent back once per subscription.

`finalizedBlockRuntime` is present if and only if `runtimeUpdates`, the parameter to this function, is `true`.

The format of `finalizedBlockRuntime` is described later down this page.

### newBlock

```json
{
    "event": "newBlock",
    "blockHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "parentBlockHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "newRuntime": ...
}
```

The `newBlock` indicates a new non-finalized block.

`parentBlockHash` is guaranteed to be equal either to the current finalized block hash, or to a block reported in an earlier `newBlock` event.

`newRuntime` must not be present if `runtimeUpdates`, the parameter to this function, is `false`. `newRuntime` must be `null` if the runtime hasn't changed compared to its parent.

If present and non-null, the format of `newRuntime` is the same as the `finalizedBlockRuntime` field in the `initialized` event and is explained later down this page.

### bestBlockChanged

```json
{
    "event": "bestBlockChanged",
    "bestBlockHash": "0x0000000000000000000000000000000000000000000000000000000000000000"
}
```

The `bestBlockChanged` event indicates that the given block is now considered to be the best block.

`bestBlockHash` is guaranteed to be equal either to the current finalized block hash, or to a block reported in an earlier `newBlock` event.

### finalized

```json
{
    "event": "finalized",
    "finalizedBlocksHashes": [
        "0x0000000000000000000000000000000000000000000000000000000000000000",
        "0x0000000000000000000000000000000000000000000000000000000000000000"
    ],
    "prunedBlocksHashes": [
        "0x0000000000000000000000000000000000000000000000000000000000000000",
        "0x0000000000000000000000000000000000000000000000000000000000000000"
    ]
}
```

The `finalized` event indicates that some of the blocks that have earlier been reported with `newBlock` events are now either finalized or no longer relevant.

`finalizedBlocksHashes` contains a list of blocks that are now part of the finalized block, ordered by increasing block number. The last element in this list is the current finalized block.

`prunedBlocksHashes` contains, in no particular order, a list of blocks that are not descendants of the latest finalized block. These blocks will never be finalized and can be discarded.

All items in `finalizedBlocksHashes` and `prunedBlocksHashes` are guaranteed to have been reported through earlier `newBlock` events.

The current best block, in other words the last block reported through a `bestBlockChanged` event, is guaranteed to either be the last item in `finalizedBlocksHashes`, or to not be present in either `finalizedBlocksHashes` or `prunedBlocksHashes`.

### stop

```json
{
    "event": "stop"
}
```

The `stop` event indicates that the JSON-RPC server was unable to provide a consistent list of the blocks at the head of the chain. This can happen because too many blocks have been pinned, because doing so would use an unreasonable amount of memory, or because a consensus mechanism creates a gap in the chain.

**Note**: In particular, warp syncing algorithms create a "jump" in the chain from a block to a much later block. Any subscription that is active when the warp syncing happens will receive a `stop` event.

No more event will be generated with this `subscription`.

Calling `chainHead_unstable_unfollow` on a subscription that has produced a `stop` event is optional.

## Pinning

The current finalized block reported in the `initialized` event, and each subsequent block reported with a `newBlock` event, is automatically considered by the JSON-RPC server as *pinned*. A block is guaranteed to not leave the node's memory for as long as it is pinned, making it possible to call functions such as `chainHead_unstable_header` on it. Blocks must be unpinned by the JSON-RPC client by calling `chainHead_unstable_unpin`.

When a block is unpinned, on-going calls to `chainHead_unstable_body`, `chainHead_unstable_call` and `chainHead_unstable_storage` against this block will still finish normally.

A block is pinned only in the context of a specific subscription. If multiple `chainHead_unstable_follow` subscriptions exist, then each `(subscription, block)` tuple must be unpinned individually. Blocks stay pinned even if they have been pruned from the blockchain, and must always be unpinned by the JSON-RPC client.

The JSON-RPC server is strongly encouraged to enforce a limit to the maximum number of pinned blocks. If this limit is reached, it should then stop the subscription by emitting a `stop` event. This specification does not mention any specific limit, but it should be large enough for clients to be able to pin all existing non-finalized blocks and a few finalized blocks.

**Note**: A JSON-RPC client should call `chainHead_unstable_unpin` only after it is sure to no longer be interested in a certain block. This typically happens after the block has been finalized or pruned. There is no requirement to call `chainHead_unstable_unpin` as quickly as possible.

## Multiple subscriptions

If a JSON-RPC client maintains mutiple `chainHead_unstable_follow` subscriptions at the same time, it has no guarantee that the blocks reported by the various subscriptions are the same. While the finalized blocks reported should eventually be the same, it is possible that in the short term some subscriptions lag behind others.

**Note**: For example, imagine there exists two active `chainHead_unstable_follow` subscriptions named A and B. Block N is announced on the peer-to-peer network and is announced to A. But then a sibling of block N gets finalized, leading to block N being pruned. Block N might never be announced to B.

## About the runtime

The format of the `finalizedBlockRuntime` and `newRuntime` fields can be one of:

#### valid

```json
{
    "type": "valid",
    "spec": {
        "specName": ...,
        "implName": ...,
        "authoringVersion": ...,
        "specVersion": ...,
        "implVersion": ...,
        "transactionVersion": ...,
        "apis": [...],
    }
}
```

In normal situations, the `type` is `valid`.

The fields of `spec` are:

- `specName`: Opaque string indicating the name of the chain.

- `implName`: Opaque string indicating the name of the implementation of the chain.

- `authoringVersion`: Opaque integer. Used by the internals of validator nodes that use a native executor in order to be sure that their native executor matches the WebAssembly code.

- `specVersion`: Opaque integer. The JSON-RPC client can assume that the call to `Metadata_metadata` will always produce the same output as long as the `specVersion` is the same.

- `implVersion`: Opaque integer. Whenever the runtime code changes in a backwards-compatible way, the `implVersion` is modified while the `specVersion` is left untouched.

- `transactionVersion`: Opaque integer. Necessary when building the bytes of a transaction. Transactions that have been generated with a different `transactionVersion` are incompatible.

- `apis`: Object containing a list of "entry point APIs" supported by the runtime. Each key is an opaque string indicating the API, and each value is an integer version number. Before making a runtime call (using `chainHead_unstable_call`), you should make sure that this list contains the entry point API corresponding to the call and with a known version number.

**Note**: In Substrate, the key contains the hexadecimal-encoded 8-bytes blake2 hash of the name of the API. For example, the `TaggedTransactionQueue` API is `0xd2bc9897eed08f15`.

**Note**: The format of `apis` is not the same as in the legacy JSON-RPC API.

#### Example value

```json
{
    "specName": "westend",
    "implName": "parity-westend",
    "authoringVersion": 2,
    "specVersion": 9122,
    "implVersion": 0,
    "transactionVersion": 7,
    "apis": {
        "0xdf6acb689907609b": 3
        "0x37e397fc7c91f5e4": 1,
        "0x40fe3ad401f8959a": 5,
        "0xd2bc9897eed08f15": 3,
        "0xf78b278be53f454c": 2,
        "0xaf2c0297a23e6d3d": 1,
        "0x49eaaf1b548a0cb0": 1,
        "0x91d5df18b0d2cf58": 1,
        "0xed99c5acb25eedf5": 3,
        "0xcbca25e39f142387": 2,
        "0x687ad44ad37f03c2": 1,
        "0xab3c0572291feb8b": 1,
        "0xbc9d89904f5b923f": 1,
        "0x37c8bb1350a9a2a8": 1
    }
}
```

### invalid

```json
{
    "type": "invalid",
    "error": "..."
}
```

The runtime is of type `invalid` if the JSON-RPC server considers the runtime as invalid, for example because the WebAssembly runtime code doesn't match its expectations.

`error` is a human-readable string indicating why the node considers it as invalid. This string isn't meant to be shown to end users, but is for developers to understand the problem.

**Note**: The typical situation where a node could consider the runtime as invalid is a light client after a warp syncing. The light client knows that it's its fault for considering the runtime as invalid, but it has no better way to handle this situation than to return an error through the JSON-RPC interface for the error to get shown to the user.
