# RGB Lightning Sample

RGB-enabled LN node based on [ldk-sample].

The node enables the possibility to create payment channels containing assets
issued using the RGB protocol, as well as routing RGB asset denominated
payments across multiple channels, given that they all possess the necessary
liquidity. In this way, RGB assets can be transferred with the same user
experience and security assumptions of regular Bitcoin Lightning Network
payments. This is achieved by adding to each lightning commitment transaction a
dedicated extra output containing the anchor to the RGB state transition.

More context on how RGB works on the Lightning Network can be found [here](https://docs.rgb.info/lightning-network-compatibility).

The RGB functionality for now can be tested only in a regtest environment,
but an advanced user may be able to apply changes in order to use it also on
other networks. Please be careful, this software is early alpha, we do not take
any responsability for loss of funds or any other issue you may encounter.

Also note that the following RGB projects (included in this project as git
sumbodules) have been modified in order to make the creation of static
consignments (without entropy) possible. Here links to compare the applied
changes:
- [bp-core](https://github.com/RGB-Tools/bp-core/compare/v0.9...static)
- [client_side_validation](https://github.com/RGB-Tools/client_side_validation/compare/v0.9...static)
- [rgb-core](https://github.com/RGB-Tools/rgb-core/compare/v0.9.0...static)
- [rgb-node](https://github.com/RGB-Tools/rgb-node/compare/v0.9.1...static)
- [rust-rgb20](https://github.com/RGB-Tools/rust-rgb20/compare/v0.9.0...static)

But most importantly [rust-lightning] has been changed in order to support
RGB channels,
[here](https://github.com/RGB-Tools/rust-lightning/compare/v0.0.113...rgb)
a compare with `v0.0.113`, the version we applied the changes to.

## Installation

Clone the project:
```sh
git clone https://github.com/RGB-Tools/ldk-sample
```

Initialize the git submodules:
```sh
git submodule update --init
```

Build the modified RGB node docker image and the ldk-sample crate:
```sh
docker-compose build
cargo build
```

## Usage (in regtest environment)

A regtest environment has been added for easier testing.

Instructions and commands are meant to be run from the project's root
directory.

The included `Dockerfile` builds an RGB node image that is required to add RGB
functionality to [ldk-sample]. Each ldk node requires its dedicated RGB node.

The `docker-compose.yml` file manages:
- a regtest bitcoind node
- an electrs instance
- an [RGB proxy server] instance
- 3 RGB nodes

Run this command in order to start with a clean regtest environment:
```sh
tests/test.sh --start
```

The command will create the directories needed by the services, start the
docker services and mine some blocks. Command will always start clean, taking
down previous running services if any.

Once services are running, ldk nodes can be started.
Each ldk node needs to be started in a separate shell with `cargo run`,
specifying:
- bitcoind user, password, host and post
- ldk data directory
- rgb-node port
- ldk peer listening port
- network

Here's an example of how to start three nodes, each one with its own rgb-node:
```sh
# 1st shell
cargo run user:password@localhost:18443 dataldk0/ 63963 9735 regtest

# 2nd shell
cargo run user:password@localhost:18443 dataldk1/ 63964 9736 regtest

# 3rd shell
cargo run user:password@localhost:18443 dataldk2/ 63965 9737 regtest
```

Once ldk nodes are running, they can be operated via their CLI.
See the [on-chain] and [off-chain] sections below and the CLI `help` command for
information on the available commands.

To stop running nodes, exit their CLI with the `quit` command (or `^D`).

To stop running services and to cleanup data directories, run:
```sh
tests/test.sh --stop
```

If needed, more nodes can be added. To do so:
- add data directories for the additional rgb (`datargb<n>`) and ldk
  (`dataldk<n>`) nodes
- add an entry for each additional rgb-node in `docker-compose.yml`, with
  different exposed port and data directory
- run additional `cargo run`s for ldk nodes, specifying the correct rgb node
  and peer listening ports

## On-chain operations

On-chain RGB operations are available as CLI commands. The following sections
briefly explain how to use each one of them.

### Issuing an asset
To issue a new asset, call the `issueasset` command followed by:
- total supply
- ticker
- name
- precision

Example:
```
issueasset 1000 USDT Tether 0
```

### Receiving assets
To receive assets, call the `receiveasset`.
Provide the sender with the returned blinded UTXO.

Example:
```
receiveasset
```

### Sending assets
To send assets to another node with an on-chain transaction, call the
`sendasset` command followed by:
- the asset's contract ID
- the amount to be sent
- the recipient's blinded UTXO

Example:
```
sendasset rgb1lfxs4dmqs7a90vrz0yaje60fakuvu9u9esx882shy437yxazmysqamnv2r 400 txob1y3w8h9n4v4tkn37uj55dvqyuhvftrr2cxecp4pzkhjxjc4zcfxtsmdt2vf
```

### Refreshing a transfer
Transfers complete automatically on the sender side after the `sendasset`
command. To complete a transfer on the receiver side, once the send operation
is complete, call the `refresh` command.

Example:
```
refresh
```

### Showing an asset's balance
To show an asset's balance, call the `assetbalance` command followed by the
asset's contract ID for which the balance should be displayed.

Example:
```
assetbalance rgb1lfxs4dmqs7a90vrz0yaje60fakuvu9u9esx882shy437yxazmysqamnv2r
```

### Mining blocks
A command to mine new blocks is provided for convenience. To mine new blocks,
call the `mine` command followed by the desired number of blocks.

Example:
```
mine 6
```

## Off-chain operations

Off-chain RGB operations are available as modified CLI commands. The following
sections briefly explain which commands support RGB functionality and how to use
them.

### Opening channels
To open a new channel, call the `openchannel` command followed by:
- the peer's pubkey, host and port
- the bitcoin amount to allocate to the channel, in satoshis
- the bitcoin amount to push, in millisatoshis
- the RGB asset's contract ID
- the RGB amount to allocate to the channel
- the `--public` optional flag, to announce the channel

Example:
```
openchannel 03ddf2eedb06d5bbd128ccd4f558cb4a7428bfbe359259c718db7d2a8eead169fb@127.0.0.1:9736 999666 546000 rgb1lfxs4dmqs7a90vrz0yaje60fakuvu9u9esx882shy437yxazmysqamnv2r
zmysqamnv2r 100 --public
```

### Listing channels
To list the available channels, call the `listchannels` command. The output
contains RGB information about the channel:
- `rgb_contract_id`: the asset's contract ID
- `rgb_local_amount`: the amount allocated to the local peer
- `rgb_remote_amount`: the amount allocated to the remote peer

Example:
```
listchannels
```

### Sending assets
To send RGB assets over the LN network, call the `keysend` command followed by:
- the receiving peer's pubkey
- the bitcoin amount in satoshis
- the RGB asset's contract ID
- the RGB amount

Example:
```
keysend 03ddf2eedb06d5bbd128ccd4f558cb4a7428bfbe359259c718db7d2a8eead169fb 2000000 rgb1lfxs4dmqs7a90vrz0yaje60fakuvu9u9esx882shy437yxazmysqamnv2r 10
```

At the moment, only the `keysend` command has been modified to support RGB
functionality. The invoice-based `sendpayment` will be added in the future.

### Closing channels
To close a channel, call the `closechannel` (for a cooperative close) or the `forceclosechannel` (for a unilateral close) command followed by:
- the channel ID
- the peer's pubkey

Example (cooperative):
```
closechannel 83034b8a3302bb9cc63d75ffd49b03e224cb28d4911702827a8dd2553d0f5229 03ddf2eedb06d5bbd128ccd4f558cb4a7428bfbe359259c718db7d2a8eead169fb
```

Example (unilateral):
```
forceclosechannel 83034b8a3302bb9cc63d75ffd49b03e224cb28d4911702827a8dd2553d0f5229 03ddf2eedb06d5bbd128ccd4f558cb4a7428bfbe359259c718db7d2a8eead169fb
```

## Scripted tests

A few scenarios can be tested using a scripted sequence.

The entrypoint for scripted tests is the shell command `tests/test.sh`,
which can be called from the project's root directory.

To view the available tests, call it with the `-l` option.
Example:
```sh
tests/test.sh -l
```

To start a test, call it with:
- the `-t` option followed by the test name
- the `--start` option to automatically create directories and start the services
- the `--stop` option to automatically stop the services and cleanup
Example:
```sh
tests/test.sh -t multihop --start --stop
```

To get a help message, call it with the `-h` option.
Example:
```sh
tests/test.sh -h
```

## License

Licensed under either:

 * Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT License ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

[RGB proxy server]: https://github.com/grunch/rgb-proxy-server
[ldk-sample]: https://github.com/lightningdevkit/ldk-sample
[off-chain]: #off-chain-operations
[on-chain]: #on-chain-operations
[rust-lightning]: https://github.com/lightningdevkit/rust-lightning
