# Developing Secret Contracts

Secret Contacts are based on CosmWasm v0.10.

For more information check out the CosmWasm docs [HERE](https://docs.cosmwasm.com).

Before proceeding, we recommend learning about differences between SecretWasm and CosWasm [HERE](#differences-from-cosmwasm).

<details>
  <summary>Topics covered on this page</summary>

- [Developing Secret Contracts](#developing-secret-contracts)
  - [IDEs](#ides)
  - [Personal Secret Network for Secret Contract development](#personal-secret-network-for-secret-contract-development)
  - [Init](#init)
  - [Handle](#handle)
  - [Query](#query)
  - [Inputs](#inputs)
  - [APIs](#apis)
  - [State](#state)
  - [Some libraries/crates considerations](#some-libraries-crates-considerations)
  - [Randomness](#randomness)
    - [Roll your own](#roll-your-own)
      - [Poker deck shuffling example](#poker-deck-shuffling-example)
    - [Use an external oracle](#use-an-external-oracle)
  - [Outputs](#outputs)
  - [External query](#external-query)
  - [Compiling](#compiling)
    - [With docker](#with-docker)
    - [Without docker](#without-docker)
  - [Storing and deploying](#storing-and-deploying)
  - [Verifying your contract code on explorers](#verifying-your-contract-code-on-explorers)
  - [Testing](#testing)
  - [Debugging](#debugging)
  - [Building secret apps with SecretJS](#building-secret-apps-with-secretjs)
    - [Wallet integration](#wallet-integration)
  - [Differences from CosmWasm](#differences-from-cosmwasm)

</details>

## IDEs

Secret Contracts are written in the [Rust](https://www.rust-lang.org/) programming language and compiled to [WASM](https://webassembly.org/) binaries.

IDEs working well for developing Secret Contracts are:

- [CLion](https://www.jetbrains.com/clion/)
- [VSCode](https://code.visualstudio.com/) with the [rust-analyzer](https://rust-analyzer.github.io/) extension

## Personal Secret Network for Secret Contract Development

The developer blockchain is configured to run inside a docker container. Install [Docker](https://docs.docker.com/get-docker/) for your environment (Mac, Windows, Linux). 

**NOTE**: M1 Macs require additional effort to develop Secret Contracts for. Linux is the recommended platform.

Open a terminal window and change to your project directory.

Then start the developer SecretNetwork blockchain, labeled _secretdev_:

```
$ docker run -it --rm \
 -p 26657:26657 -p 26656:26656 -p 1317:1317 \
 --name secretdev enigmampc/secret-network-bootstrap-sw:latest
```

**NOTE**: The _secretdev_ docker container can be stopped with Ctrl+C

At this point you're running a local SecretNetwork full-node. 
    
Now let's connect to the container so we can view and manage the secret keys. 
    
Open a new terminal window and connect to the docker container:

```
docker exec -it secretdev /bin/bash
```

The local blockchain has a couple of keys setup for you (similar to accounts if you're familiar with Truffle Ganache). The keys are stored in the `test` keyring backend, which makes it easier for local development and testing.

```
secretcli keys list --keyring-backend test
```

![](../images/images/secretcli_keys_list.png)

`exit` when you are done.

## Init

`init` is the constructor of your contract. This function is called only once in the lifetime of the contract.

Example invocation from `secretcli`:

```bash
secretcli tx compute instantiate "$CODE_ID" "$INPUT_MSG" --label "$UNIQUE_LABEL" --from "$MY_KEY"
```

Example invocation from `SecretJS`:

```js
// TODO
```

## Handle

`handle` is the implementation of execute transactions.

Example invocation from `secretcli`:

```bash
secretcli tx compute execute "$CONTRACT_ADDRESS" "$INPUT_ARGS" --from "$MY_KEY" # Option A
secretcli tx compute execute --label "$LABEL" "$INPUT_ARGS" --from "$MY_KEY"    # Option B
```

Example invocation from `SecretJS`:

```js
// TODO
```

## Query

`query` is the implementation of read-only queries. Queries run over the current blockchain state, don't incur fees, and don't have access to `msg.sender`. **Queries are still metered by a gas limit as set on the executing node.**

Example invocation from `secretcli`:

```bash
secretcli q compute query "$CONTRACT_ADDRESS" "$INPUT_ARGS"
```

Example invocation from `SecretJS`:

```js
// TODO
```

## Inputs

## APIs

## State

## Debug printing

Under normal circumstances, contracts **cannot** be debugged while running 
on-chain, inside the enclave.
In most cases where you need to track down the source of an issue, this is fine:
- If a function is behaving unexpectedly, a unit test can be written to isolate
  and fix the issue
- If a `Result::Err` is being thrown, it usually has a clear and
  distinct error message which can be tracked to its source
- If your contract returns a result, but it's wrong and you
  can't figure out why, you can add logs to the response that show the values
  of variables

There is one error case that halts the contract, reverts the state,
and returns no feedback for the cause of the failure: **PANICS**

Starting from version 1.1.0 of SecretNetwork, we provide another useful tool to
the contract developing toolkit: debug prints.

`cosmwasm_std::debug_print` is both a function taking a `String` or `&str`,
as well as a macro that works just like `println` in normal rust. By default,
using it does nothing at all, and it's compiled away. To activate the debug
printing feature, contracts must enable the `debug-print` feature in
`cosmwasm_std`. This can be done by adding this line under the `[features]`
section in `Cargo.toml`:
```
debug-print = ["cosmwasm_std/debug-print"]
```
and then compiling the contract with `--features debug-print`.

Once compiled with this flag, and executed on the local dev image (using
`enigmampc/secret-network-sw-dev`), the messages passed to `debug_print` will
be shown as logs in the dev image's logging output.

Now you can print information from within the contract to the console running
the dev image, and these logs will be visible there; even if the contract
crashes. This is generally more convenient than the three techniques described
above. You can also safely leave the calls to `debug_print` in your source code,
and they will only have an effect when used during development!

Naturally, we don't want node runners to see private debug information in
production, so the interfaces used by this API are only available in the
local dev image. 
    
**Attempting to store a module compiled with this flag to testnet or mainnet will fail on validation.**

## Some libraries/crates considerations

- `bincode2` instead of `bincode` for serializing data.
- `serde_json_wasm` instead of `serde_json` for serializing data.
- `bech32` instead of `deps.api.canonical_address` and `deps.api.human_address`, as they only support `secret` prefix (E.g. not `secretvaloper` for staking use-cases).

## Randomness

### Roll your own

#### Poker deck shuffling example

1. When joining a room, [each player sends a secret number](https://github.com/scrtlabs/SecretHoldEm/blob/4f67c469bb4a0f53522c7ad069e54ae5c1effb6b/contract/src/contract.rs#L172).
2. Once the room is full, all secrets are combined with sha256 to [create a random seed](https://github.com/scrtlabs/SecretHoldEm/blob/4f67c469bb4a0f53522c7ad069e54ae5c1effb6b/contract/src/contract.rs#L349-L355).
3. With that seed, the [deck is shuffled](https://github.com/scrtlabs/SecretHoldEm/blob/4f67c469bb4a0f53522c7ad069e54ae5c1effb6b/contract/src/contract.rs#L356-L357).
4. Each round a [game counter is incremented](https://github.com/scrtlabs/SecretHoldEm/blob/4f67c469bb4a0f53522c7ad069e54ae5c1effb6b/contract/src/contract.rs#L602-L614), and along with the players' secrets is used to create a new seed for re-shuffling the deck.
5. On the frontend side, [SecretJS is used to generate a secure random number](https://github.com/scrtlabs/SecretHoldEm/blob/4f67c469bb4a0f53522c7ad069e54ae5c1effb6b/gui/src/App.js#L334-L354) and sends it as a secret when a player joins the table. A random number is not really necessary, and every secret number would work just as well.
6. As long as at least one player is not colluding with the rest, and by properties of sha256, the seeds for shuffling the deck are known only to the contract and to no one else. If all players are colluding, they might as well all play with open hands. :joy:

### Use an external oracle

No implementation exists yet, but it's not that hard to implement.

For example:

1. Have a `handle` function `input_entropy` for users to send entropy in
2. `input_entropy` will have a storage key named `seed`
3. On each input to `input_entropy`, `seed = hash(seed + input)`
4. Have another `handle` function `get_random_number` for users to get a random number
5. `get_random_number` must also add to the entropy pool, otherwise consecutive `get_random_number` calls will output the same random number. For example `seed = hash(seed + msg.sender + block.height + ...)`
6. `get_random_number` will just return the `hash(seed)` or some other non-reversible derivative of it, and update the `seed` with the new entropy like described in the previous point
7. Have `get_random_number` also callback to the caller contract with the random number
8. You can even have a cron job to send data from [random.org](https://www.random.org/) to `input_entropy`

This example has a much worse UX than rolling your own randomness, but at least contracts won't have to rely on users to send entropy and also won't take the risk of messing up the implementation.

## Outputs

## External query

## Compiling

### With docker

```console
$ docker run --rm -it -v /absolute/path/to/contract/project:/contract enigmampc/secret-contract-optimizer
```

Where `/absolute/path/to/contract/project` is pointing to the directory that contains your Secret Contract's `Cargo.toml`.

This will output an optimized build file `/absolute/path/to/contract/project/contract.wasm.gz`.

### Without docker

```console
$ rustup target add wasm32-unknown-unknown
$ sudo apt install binaryen
```

```console
$ RUSTFLAGS='-C link-arg=-s' cargo build --release --target wasm32-unknown-unknown --locked
$ wasm-opt -Oz ./target/wasm32-unknown-unknown/release/*.wasm -o ./contract.wasm
$ cat ./contract.wasm | gzip -9 > ./contract.wasm.gz
```

Breakdown:

1. Build a release mode WASM file, strip symbols
2. Further optimize with [wasm-opt](https://github.com/WebAssembly/binaryen)
3. Gzip

## Storing and deploying

## Verifying your contract code on explorers

## Testing

## Debugging

## Building secret apps with SecretJS

A Secret App, or a SApp, is a DApp with computational and data privacy.
A Secret App is usually comprised of the following components:

- A Secret Contract deployed on the Secret Network
- A frontend app built with a JavaScript framework (E.g. ReactJS, VueJS, AngularJS, etc.)
- The frontend app connects to the Secret Network using SecretJS
- SecretJS interacts with a REST API exposed by nodes in the Secret Network, the REST API/HTTPS server is commonly referred to as LCD Server (Light Client Daemon)
- By connecting SecretJS with a wallet, the wallet handles the interactions with the LCD server

### Wallet integration

Still not implemented in wallets. Can implement a local wallet, but this will probably won't be needed anymore after 2020.

## Differences from CosmWasm

Secret Contacts are based on CosmWasm v0.10, but in order to preserve privacy, they diverge in functionality in some cases.

- `code_hash` in callbacks
- Can access the current contract's `code_hash` via `env.contract_code_hash`
- Contract labels are unique, thus mandatory on callback to `init`
- `migrate` and `admin` for contracts is not allowed
- Iterator (`db_scan`, `db_next`) on contract state keys is not allowed
- `cosmwasm_std` changes...
- SGX memory limits (30 MiB total, 12 MiB per contract vs 32 MiB per contract)
- We charge gas for memory allocations, vanilla CosmWasm don't
- `SecretJS` has a new function - `GenerateNewSeed` to help apps to get a secure 32 byte random number
