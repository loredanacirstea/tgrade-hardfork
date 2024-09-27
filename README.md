# Tgrade Hard Fork


https://github.com/loredanacirstea/tgrade/tree/tgrade-hardfork

New Genesis:
(link todo)

Genesis checksum: `TODO`

Note: after the chain is live, we will need to analyze the data, try some transactions, make sure everything behaves as expected for at least 1 day, before we can consider the chain alive again.

## Process

* close your node
* archive your old Tgrade state if you need
* reset data & private validator state with `tgrade tendermint unsafe-reset-all`
* replace your existing `./config/genesis.json` with the new genesis
* build your `tgrade` binary from `tgrade-hardfork` and replace it
```
git clone git@github.com:loredanacirstea/tgrade.git
git checkout tgrade-hardfork
make install
```
* only have active validators in your `persistent_peers=""` from `./config/config.toml` 
* start node

## Reproduce hard-fork genesis:

Migration code is in https://github.com/loredanacirstea/tgrade/blob/tgrade-hardfork/cmd/tgrade/hardfork.go

* export your current Tgrade genesis

```sh
tgrade export --home=./testnet/node0/tgrade > ./tgrade_state_export.json
```

* migrade genesis changing the validator set

```sh
tgrade migrate-genesis-with-validatorset [genesis_file] [output_file] [hardfork_index] [validator_addresses_file]

# example
tgrade migrate-genesis-with-validatorset ./tgrade_state_export.json ./tgrade_state_export_migrated.json 2 ./tgrade_validators.json
```

### Utils

* if you are looking at the genesis, for decoding KVmodel key-value pairs from contract storage: https://observablehq.com/d/9dfb012116b2391b


## Other testing

You can also test on a local chain that you start with 2+ validators. Then stop the chain, export the genesis, migrate it to only remain with 1+ validator, then see if the chain restarts.

```sh
tgrade testnet --chain-id=testing --output-dir=$(pwd)/testnet --v=2 --keyring-backend=test --commit-timeout=1500ms --minimum-gas-prices=""

tgrade export --home=./testnet/node0/tgrade > ./testnet/tgrade_state_export.json

# copy node0 -> node00 & reset state

tgrade tendermint unsafe-reset-all --home=./testnet/node00/tgrade

# create your ./testnet/validators.json with an JSON array of whitelisted bech32 validator addresses, one of which should be your `node0` address e.g.
# ["tgrade18q2le253fl6d9jjuq0qp7w95pfqwfks9acycs2"]

tgrade migrate-genesis-with-validatorset ./testnet/tgrade_state_export.json ./testnet/tgrade_state_export_migrated.json 2 ./testnet/validators.json && rm ./testnet/node00/tgrade/config/genesis.json && cp ./testnet/tgrade_state_export_migrated.json ./testnet/node00/tgrade/config/genesis.json

tgrade start --home=./testnet/node00/tgrade
```
