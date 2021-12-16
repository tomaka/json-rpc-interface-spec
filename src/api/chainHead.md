# Introduction

Functions with the `chainHead` prefix allow tracking the head of the chain (in other words, the latest new and finalized blocks) and their storage.

The most important function in this category is `chainHead_unstable_follow`. It is the first function that is the user is expected to call, before (almost) any other. `chainHead_unstable_follow` returns the current list of blocks that are near the head of the chain, and generates notifications about new blocks. The `chainHead_unstable_body`, `chainHead_unstable_call`, `chainHead_unstable_header` and `chainHead_unstable_storage` functions can be used to obtain more details about the blocks that have been reported by `chainHead_unstable_follow`.

These functions are the functions most of the JSON-RPC clients will most commonly use. A JSON-RPC server implementation is encouraged to prioritize serving these functions over other functions, and to put pinned blocks in a quickly-accessible cache.

## Usage

_This section contains a small beginner guide destined for JSON-RPC client users._

This beginner guide shows how to use the `chainHead` functions in order to know the value of a certain storage item.

1. Call `chainHead_unstable_follow` with `runtimeUpdates: true` to obtain a `followSubscriptionId`. This `followSubscriptionId` will need to be passed when calling most of the other `chainHead`-prefixed functions. If at any point in the future the JSON-RPC server sends back a `{"event": "stop"}` notification, jump back to step 1.

2. When the JSON-RPC server sends back a `{"event": "initialized"}` notification with `subscriptionId` equal to your `followSubscriptionId`, store the value of `finalizedBlockHash` in that notification.

3. Call `chainHead_unstable_call` with `hash` equal to the `finalizedBlockHash` you've just retrieved, `function` equal to `Metadata_metadata`, and an empty `callParameters`. **TODO** must check metadata api version

4. If the JSON-RPC server sends back a `{"event": "inaccessible"}` notification, jump back to step 3. If the JSON-RPC server sends back a `{"event": "error"}` notification, enter panic mode as the client software is incompatible with the current state of the blockchain. If the JSON-RPC server sends back a `{"event": "disjoint"}` notification, jump back to step 1. If the JSON-RPC server instead sends back a `{"event": "done"}` notification, save the return value.

5. The return value you've just saved is called the metadata, prefixed with its SCALE-compact-encoded length. You must decode and parse this metadata. How to do this is out of scope of this small guide. The metadata contains information about the layout of the storage of the chain. Inspect it to determine how to find the storage item you're looking for.

6. In order to obtain a value in the storage, call `chainHead_unstable_storage` with `hash` equal to `finalizedBlockHash`, `key` the desired key, and `type` equal to `value`. If the JSON-RPC server sends back a `{"event": "disjoint"}` notification, jump back to step 1. If the JSON-RPC server instead sends back a `{"event": "inaccessible"}` notification, the value you're looking for is unfortunately inaccessible. If the JSON-RPC server instead sends back a `{"event": "done"}` notification, you can find the desired value inside.

7. You are strongly encouraged to maintain [a `Set`](https://developer.mozilla.org/fr/docs/Web/JavaScript/Reference/Global_Objects/Set) of the blocks where the runtime changes. Whenever a `{"event": "newBlock"}` notification is received with `subscriptionId` equal to your `followSubcriptionId`, and `newRuntime` is non-null, store the provided `blockHash` in this set.

8. Whenever a `{"event": "finalized"}` notification is received with `subscriptionId` equal to your `followSubcriptionId`, call `chainHead_unstable_unpin` once with your current `finalizedBlockHash` and once for each value in `finalizedBlocksHashes` except for the last one. The last value in `finalizedBlocksHashes` becomes your new `finalizedBlockHash`. If one or more entries of `finalizedBlockHashes` is found in your `Set` (see step 6), remove them from the set and jump to step 3 as the metadata has likely been modified. Otherwise, jump to step 6.

Note that these steps are a bit complicated. Any serious user of the JSON-RPC interface is expected to implement high-level wrappers around the various JSON-RPC functions.

For example, if multiple storage values are desired, only step 6 should be repeated once per storage item. All other steps are application-wide.