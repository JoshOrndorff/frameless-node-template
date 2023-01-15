# Substrate Frameless Node Template

Welcome to the FRAME-LESS Runtime!

A stripped down version of the [node template](https://github.com/substrate-developer-hub/substrate-node-template), ready for hackin'.

## Instructions

Here are the list of things you have to do:

### 1: Get Those Roots In Order!

The template already does most of the work to make sure your heads contain the right state root and
extrinsic root. As your first step, make sure this is the case! Both of these should be set in
`on_finalize`, and double-checked in `execute_block`.

The easy way to check that your extrinsic root check is working is running your node without
`--release`. This will enable a `debug_assert!` on the client side that will ensure your authored
block is setting the right extrinsic root. If you want a bit more, try syncing your blockchain with
a new node, like:

```
TODO.
```

The second node should fail to sync until you fix this. Why is that? because the given
implementation only checks the extrinsic root at the end of `execute_block`.

Recall that the flow of a block-author node is:

```sh
initialize_block(empty_header: Header);
apply_extrinsic(ext: _);
finalize_block() -> Header;
```

and the block importer calls:

```sh
initialize_block(actual_header: Header);
execute_block();
```

### 2. Basic write

Alter your `BasicExtrinsic` to accept a very basic call type where you write a given value into the
storage under a hardcoded key.

Something along the lines of

```rust
pub enum Call {
	Set(u32),
}
```

### 2. Make it Upgradable!

Next, make your chain upgradable. This should in itself be really easy, it is almost like the
previous set.

Whilst doing the upgrade, make sure to:

1. break your transaction format. For example, change `Set(u32)` to `Set(u128)`
2. bump your spec-version so that you won't get native execution issues.

Now, try and submit a new transaction...

### 3. Opaque Transactions

Yes, it failed because of:

```rust
pub mod opaque {
	type OpaqueExtrinsic = BasicExtrinsic;
	pub type Block = generic::Block<Header, OpaqueExtrinsic>;
}
```

This module is used in your client to understand what an "opaque" (untyped) extrinsic is, and you
have hardcoded it to `BasicExtrinsic``! Of course, because your client is not upgraded, it cannot
decode new transactions anymore (and if you send the old format, the runtime won't be able to decode
it).

The correct type to use is `sp_runtime::OpaqueExtrinsic`. Take a look and replace it.

```rust
type OpaqueExtrinsic = sp_runtime::OpaqueExtrinsic;
```

Replay the above scenario. Will it work?

### 4. Opaque Decoding

Well, still no. The reason for that is that now your client will think of an extrinsic as `Vec<u8>`,
and the Runtime thinks of it as `BasicExtrinsic`. How are you going to link the two? Which bytes

In other words, imagine you pass in some bytes to your `curl/wscat` command. The same bytes should
be decode-able as both a `Vec<u8>` and `BasicExtrinsic`.

This requires you to write a custom `Encode/Decode` implementation for your `BasicExtrinsic`. The
best way to hint you at the solution is to guide you toward the type that is used in most real
substrate-based chains: `UncheckedExtrinsic`:
https://paritytech.github.io/substrate/master/sp_runtime/generic/struct.UncheckedExtrinsic.html. See
why and how the `Encode/Decode` implementation for this type is different.

### 5. Timestamp

Next, write an inherent for your runtime that puts the timestamp into the block. For this, you need a new call type like

```rust
pub enum Call {
	Set(u32),
	SetTimestamp(u64),
	Upgrade(Vec<u8>),
}
```

The client will ask the runtime to create any given inherent at `fn inherent_extrinsics` and asks it
to do any kind of soft-verification at `fn check_inherent`. In both cases, the substrate client will
put its currently known timestamp at `sp_inherent::INHERENT_IDENTIFIER` key of `data`.

### 6. Optional: Get Block Author

TODO


TODO: update finalize_block signature in the other lecture.
TODO: idea: extract block_author from digest?.


## Build

The `cargo run` command will perform an initial build. Use the following command to build the node without launching it:

```sh
cargo b -r
```

### Embedded CLI Docs

Once the project has been built, the following command can be used to explore all parameters and subcommands:

```sh
./target/release/node-template -h
```

### Single-Node Development Chain

This command will start the single-node development chain with non-persistent state:

```bash
./target/release/node-template --dev
```

`--dev` will imply multiple things:

1. Set `--alice` as your local node authority, which means your node will author blocks (Alice is the block author of the `dev` chain by default).
2. Set `--tmp`, which means every time your chain would start from scratch.

If you don't include `--tmp`, your chain will start accumulating its database. Purge the development chain's state:

```bash
./target/release/node-template purge-chain --dev
```

Start the development chain with detailed logging:

```bash
RUST_BACKTRACE=1 ./target/release/node-template -ldebug --dev
```

In case of being customizing the location of the database:

```bash
// Create a folder to use as the db base path
$ mkdir my-chain-state

// Use of that folder to store the chain state
$ ./target/release/node-template --dev --base-path ./my-chain-state/
```
