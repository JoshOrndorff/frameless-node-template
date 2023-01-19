# Substrate Frameless Node Template

Welcome to the FRAME-LESS Runtime!

A stripped down version of the [node
template](https://github.com/substrate-developer-hub/substrate-node-template), ready for hackin'.

- [Substrate Frameless Node Template](#substrate-frameless-node-template)
	- [Instructions](#instructions)
		- [Ungraded](#ungraded)
			- [1: Get Those Roots In Order!](#1-get-those-roots-in-order)
			- [2. Basic write](#2-basic-write)
			- [2. Make it Upgradable!](#2-make-it-upgradable)
			- [3. Opaque Transactions](#3-opaque-transactions)
			- [4. Opaque Decoding](#4-opaque-decoding)
			- [5. Timestamp](#5-timestamp)
			- [6. Optional: Get Block Author](#6-optional-get-block-author)
		- [Graded](#graded)
			- [Primary Task: The Runtime](#primary-task-the-runtime)
			- [Option 1: UTXO-based Cryptocurrency](#option-1-utxo-based-cryptocurrency)
			- [Option 2: Account-based Cryptocurrency](#option-2-account-based-cryptocurrency)
			- [Secondary Task: User Interface](#secondary-task-user-interface)
			- [Tertiary Task: Consensus](#tertiary-task-consensus)
	- [How To Build and Run](#how-to-build-and-run)
		- [Running](#running)
		- [Multiple-nodes](#multiple-nodes)
	- [RPC Cheatsheet](#rpc-cheatsheet)



## Instructions

This assignment has multiple parts. The ungraded part will be in the class and the TAs will help you
with it.

### Ungraded

#### 1: Get Those Roots In Order!

The template already does most of the work to make sure your heads contain the right state root and
extrinsic root. As your first step, make sure this is the case! Both of these should be set in
`on_finalize`, and double-checked in `execute_block`.

The easy way to check that your extrinsic root check is working is running your node without
`--release`. This will enable a `debug_assert!` on the client side that will ensure your authored
block is setting the right extrinsic root.

> Also, until you fix this, a second node, [as explained below](#multiple-nodes) will not be able to sync your node.

The second node should fail to sync until you fix this. Why is that? because the given
implementation only checks the extrinsic root at the end of `execute_block`.

Recall that the flow of a block-author node is:

```sh
initialize_block(empty_header: Header);
apply_extrinsic(ext: _);
// any changes made in this are not persisted in state.
finalize_block() -> Header;
```

and the block importer calls:

```sh
execute_block(actual_block: Block);
```

#### 2. Basic write

Alter your `BasicExtrinsic` to accept a very basic call type where you write a given value into the
storage under a hardcoded key.

Something along the lines of

```rust
pub enum Call {
	Set(u32),
}
```

#### 2. Make it Upgradable!

Next, make your chain upgradable. This should in itself be really easy, it is almost like the
previous set.

Whilst doing the upgrade, make sure to:

1. break your transaction format. For example, change `Set(u32)` to `Set(u128)`
2. bump your spec-version so that you won't get native execution issues.

Now, try and submit a new transaction...

#### 3. Opaque Transactions

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

#### 4. Opaque Decoding

Well, still no. The reason for that is that now your client will think of an extrinsic as `Vec<u8>`,
and the Runtime thinks of it as `BasicExtrinsic`. How are you going to link the two?

> Basically, ask yourself: how can you encode the `BasicExtrinsic`, such that it is decode-able as
> both `BasicExtrinsic`` and `Vec<u8>`?

In other words, imagine you pass in some bytes to your `curl/wscat` command. The same bytes should
be decode-able as both a `Vec<u8>` and `BasicExtrinsic`.

This requires you to write a custom `Encode/Decode` implementation for your `BasicExtrinsic`. The
best way to hint you at the solution is to guide you toward the type that is used in most real
substrate-based chains: `UncheckedExtrinsic`:
https://paritytech.github.io/substrate/master/sp_runtime/generic/struct.UncheckedExtrinsic.html. See
why and how the `Encode/Decode` implementation for this type is different.

#### 5. Timestamp

Next, write an inherent for your runtime that puts the timestamp into the block. For this, you need
a new call type like

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

> Look into `pallet-timestamp` for inspiration.

#### 6. Optional: Get Block Author

Inside of the block header, there is a field called `Digest`. This contains some information that is
passed from the client, to the runtime, as part of the header. Part of this information is about
which authoring "engine" is being used, and which validator is authoring blocks. Extract this
information, and stored it onchain, such that one can query a storage item from your chain state and
know who authored a given block.

> Look into `pallet-authorship` for inspiration.


### Graded

This assignment covers the material from Module 3: Blockchain Fundamentals and Module 4: Substrate.
In it, you will build a cryptocurrency Substrate runtime, a simple user interface, and gently touch
Substrate's consensus layer.

Your project should include a README (roughly 200 - 2000 words) explaining how to use the project
and what parts of code you want the grader to look at.

#### Primary Task: The Runtime

Choose one of these two options to complete. You do not need to do both.

#### Option 1: UTXO-based Cryptocurrency

- Do not use frame (Obviously because we haven't talked about it yet)
- No account model
- Runtime logic is similar to Bitcoin, Litecoin, Monero (not the privacy part)
- Users are able to spend and receive UTXOs
- Block reward is paid as a new UTXO
- Fees are implied by the difference between the consumed and produced UTXOs.

Your final runtime MUST follow the following specification:



#### Option 2: Account-based Cryptocurrency

- Do not use frame (Obviously because we haven't talked about it yet)
- Must use account model
- Runtime logic is similar to Ethereum's Eth token (not including the EVM)
- Users are able to send and receive tokens
- Block reward is paid to the author's account
- Fees are specified explicitly by the user

Your final runtime MUST expose the following functions in its runtime-api:

```rust
sp_core::crypto::sr

/// Your transaction should something that, when decoded, is compatible with the following:
struct Transaction {
	/// The payload that this signature signs must be
	signature: Signature,
	/// The call
	call: Call,
}

/// Similarly, your `Call` enum must be compatible with this:
```rust
enum Call {
	Transfer()
}



trait Api {
	fn balance_of(who: [u8; 32]) -> Option<u128>;
	#[cfg(feature = "std")]
	fn test_only_transfer(from: [u8; 32], to: [u8; 32], amount: u128);
}


```



#### Secondary Task: User Interface

Build a user-interface for the blockchain you built above. Your user interface should allow users to
read all relevant chain state and submit signed transactions. To achieve this you will need to make
use of Substrate's RPC methods.

This user interface does not need to be a beautiful webapp, although that is certainly welcome. A
simple CLI based UI is enough.

#### Tertiary Task: Consensus

Make your blockchain use a hybrid Proof of Work / Proof of Authority consensus scheme. Use Proof of
Work for block authoring. You may look at the code for the academy-pow chain that we ran in module
three for inspiration. Make it use Proof of Authority Grandpa for deterministic finality. You may
look at the Substrate node template for inspiration.


## How To Build and Run

`cargo b` and `cargo r` (short for `run` and `build`) are your default ways to build and run the
project. In almost all cases, you should run your node with `--release` to get a reasonable
performance. If you only build, the binary can be found in `./target/{debug|release}/node-template`.

Once the project has been built, the following command can be used to explore all parameters and
subcommands:

```sh
./target/release/node-template -h
```

### Running

This command will start the single-node development chain with non-persistent state:

```bash
./target/release/node-template --dev
```

`--dev` will imply multiple things:

1. Set `--alice` as your local node authority, which means your node will author blocks (Alice is
   the block author of the `dev` chain by default).
2. Set `--tmp`, which means every time your chain would start from scratch.

If you don't include `--tmp`, your chain will start accumulating its database. For this exercise,
you probably want to stick to `--tmp`. In case you don't you can purge the development chain's state:

```bash
./target/release/node-template purge-chain --dev
```

Start the development chain with detailed logging:

In order to have all of the runtime logs enabled, run the project with `RUST_LOG=frameless=trace`.

```bash
RUST_LOG=frameless=trace ./target/release/node-template --dev
```

Or

```bash
./target/release/node-template --dev -lframeless=trace
```

### Multiple-nodes

In order to run multiple nodes, you can try either of:

```
# runs a chain with --chain=dev --alice --tmp
$ ./target/release/node-template --dev
# runs a chain with --chain=dev --bob --tmp
$ ./target/release/node-template --dev --bob
```

## RPC Cheatsheet

Consult the `JSON-RPC` lecture (including the speaker notes!), or:

```
# read a storage key/
wscat -c 127.0.0.1:9944 -x '{"jsonrpc":"2.0", "id":1, "method":"state_getStorage", "params": ["0x123"]}'

# submit an extrinsic
wscat -c 127.0.0.1:9944 -x '{"jsonrpc":"2.0", "id":1, "method":"author_submitExtrinsic", "params": ["0x123"]}'

# You can easily achieve the same with `curl` as well:

curl http://34.79.74.54:9934 -H "Content-Type:application/json;charset=utf-8" -d   '{"jsonrpc":"2.0","id":1,"method":"system_chain"}'
```

The easiest way to get the encoded keys is to write a unit test in your runtime that encodes a key/extrinsic.
