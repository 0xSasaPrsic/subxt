Subxt is a library to **sub**mit e**xt**rinsics to a [substrate](https://github.com/paritytech/substrate) node via RPC.

The generated Subxt API exposes the ability to:
- [Submit extrinsics](https://docs.substrate.io/v3/concepts/extrinsics/) (Calls)
- [Query storage](https://docs.substrate.io/v3/runtime/storage/) (Storage)
- [Query constants](https://docs.substrate.io/how-to-guides/v3/basics/configurable-constants/) (Constants)
- [Subscribe to events](https://docs.substrate.io/v3/runtime/events-and-errors/) (Events)


### Generate the runtime API

Subxt generates a runtime API from downloaded static metadata. The metadata can be downloaded using the
[subxt-cli](https://crates.io/crates/subxt-cli) tool.

To generate the runtime API, use the `subxt` attribute which points at downloaded static metadata.

```rust
#[subxt::subxt(runtime_metadata_path = "metadata.scale")]
pub mod node_runtime { }
```

The `node_runtime` has the following hierarchy:

```rust
pub mod node_runtime {
    pub mod PalletName {
        pub mod calls { }
        pub mod storage { }
        pub mod constants { }
        pub mod events { }
    }
}
```

For more information regarding the `node_runtime` hierarchy, please visit the
[subxt-codegen](https://docs.rs/subxt-codegen/latest/subxt_codegen/) documentation.


### Initializing the API client

```rust
use subxt::{ClientBuilder, DefaultConfig, PolkadotExtrinsicParams};

let api = ClientBuilder::new()
    .set_url("wss://rpc.polkadot.io:443")
    .build()
    .await?
    .to_runtime_api::<node_runtime::RuntimeApi<DefaultConfig, PolkadotExtrinsicParams<DefaultConfig>>>();
```

The `RuntimeApi` type is generated by the `subxt` macro from the supplied metadata. This can be parameterized with user
supplied implementations for the `Config` and `Extra` types, if the default implementation differs from the target
chain.

To ensure that extrinsics are properly submitted, during the build phase of the Client the
runtime metadata of the node is downloaded. If the URL is not specified (`set_url`), the local host is used instead.


### Submit Extrinsics

Extrinsics are obtained using the API's `RuntimeApi::tx()` method, followed by `pallet_name()` and then the
`call_item_name()`.

Submit an extrinsic, returning success once the transaction is validated and accepted into the pool:

Please visit the [balance_transfer](../examples/examples/balance_transfer.rs) example for more details.


### Querying Storage

The runtime storage is queried via the generated `RuntimeApi::storage()` method, followed by the `pallet_name()` and
then the `storage_item_name()`.

Please visit the [fetch_staking_details](../examples/examples/fetch_staking_details.rs) example for more details.

### Query Constants

Constants are embedded into the node's metadata.

The subxt offers the ability to query constants from the runtime metadata (metadata downloaded when constructing
the client, *not* the one provided for API generation).

To query constants use the generated `RuntimeApi::constants()` method, followed by the `pallet_name()` and then the
`constant_item_name()`.

Please visit the [fetch_constants](../examples/examples/fetch_constants.rs) example for more details.

### Subscribe to Events

To subscribe to events, use the generated `RuntimeApi::events()` method which exposes:
- `subscribe()` - Subscribe to events emitted from blocks. These blocks haven't necessarily been finalised.
- `subscribe_finalized()` - Subscribe to events from finalized blocks.
- `at()` - Obtain events at a given block hash.


*Examples*
- [subscribe_all_events](../examples/examples/subscribe_all_events.rs): Subscribe to events emitted from blocks.
- [subscribe_one_event](../examples/examples/subscribe_one_event.rs): Subscribe and filter by one event.
- [subscribe_some_events](../examples/examples/subscribe_some_events.rs): Subscribe and filter event.

### Static Metadata Validation

There are two types of metadata that the subxt is aware of:
- static metadata: Metadata used for generating the API.
- runtime metadata: Metadata downloaded from the target node when a subxt client is created.

There are cases when the static metadata is different from the runtime metadata of a node.
Such is the case when the node performs a runtime update.

To ensure that subxt can properly communicate with the target node the static metadata is validated
against the runtime metadata of the node.

This validation is performed at the Call, Constant, and Storage levels, as well for the entire metadata.
The level of granularity ensures that the users can still submit a given call, even if another
call suffered changes.

Full metadata validation:

```rust
// To make sure that all of our statically generated pallets are compatible with the
// runtime node, we can run this check:
api.validate_metadata()?;
```

Call level validation:

```rust
let extrinsic = api
    .tx()
    .balances()
    // Constructing an extrinsic will fail if the metadata
    // is not in sync with the generated API.
    .transfer(dest, 123_456_789_012_345)?;
```

### Runtime Updates

There are cases when the node would perform a runtime update, and the runtime node's metadata would be
out of sync with the subxt's metadata.

The `UpdateClient` API keeps the `RuntimeVersion` and `Metadata` of the client synced with the target node.


Please visit the [subscribe_runtime_updates](../examples/examples/subscribe_runtime_updates.rs) example for more details.