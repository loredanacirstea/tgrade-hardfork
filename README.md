# Tgrade Hard Fork


https://github.com/loredanacirstea/tgrade/tree/tgrade-hardfork commit `a38e5bb4633167588c5358092840e17e1de72951`

New Genesis:
https://github.com/loredanacirstea/tgrade-hardfork/raw/refs/heads/main/tgrade_state_export_migrated.json.gz

* checksum new genesis: `0db60740680dcf67483a4005613b339fe899337aafde8fe07e30c048c733b4f0`

(Note: the exported genesis contains a timestamp `"genesis_time":"2022-06-27T12:00:01Z"` that you will have to replace it to get the above checksum) 

After the chain is live, we will need to analyze the data, try some transactions, make sure everything behaves as expected for at least 1 day, before we can consider the chain alive again.

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
tgrade export --home=./testnet/node0/tgrade > ./tgrade_state_export_initial.json
```

* replace your genesis timestamp with the one used to produce above checksum by using `--genesis-time-reset` and `--genesis-time` flags. 

```sh
tgrade migrate-genesis-with-validatorset ./tgrade_state_export_initial.json ./tgrade_state_export.json 2 ./tgrade_validators.json --genesis-time="2022-06-27T12:00:01Z" --genesis-time-reset  
```

* check checksum to be  `0a7b76253b6e0b537d2c030d2829feb85b97b5ab50443e561ce3be520513f147` (for the old Tgrade state at https://github.com/loredanacirstea/tgrade-hardfork/raw/refs/heads/main/tgrade_state_export.json.gz)

```
sha256sum ./tgrade_state_export.json
```

* migrate genesis changing the validator set

```sh
tgrade migrate-genesis-with-validatorset [genesis_file] [output_file] [hardfork_index] [validator_addresses_file]

# example
tgrade migrate-genesis-with-validatorset ./tgrade_state_export.json ./tgrade_state_export_migrated.json 2 ./tgrade_validators.json
```

* check checksum

```
sha256sum ./tgrade_state_export_migrated.json
```

### Utils

* if you are looking at the genesis, for decoding KVmodel key-value pairs from contract storage: https://observablehq.com/d/9dfb012116b2391b


## Other testing

You can also test on a local chain that you start with 2+ validators. Then stop the chain, export the genesis, migrate it to only remain with 1+ validator, then see if the chain restarts.

Example for 4 nodes. Replace the addresses with your local setup.
```sh
tgrade testnet --chain-id=testing --output-dir=$(pwd)/testnet --v=4 --single-host --keyring-backend=test --commit-timeout=1500ms --minimum-gas-prices="" --starting-ip-address=127.0.0.1

tgrade start --home=./testnet/node0/tgrade
tgrade start --home=./testnet/node1/tgrade
tgrade start --home=./testnet/node2/tgrade
tgrade start --home=./testnet/node3/tgrade

# leave chain running for > 20 blocks, then stop the nodes and export the genesis:
tgrade export --home=./testnet/node0/tgrade > ./testnet/tgrade_state_export.json

# copy the nodes you want to remain with e.g.
# ./testnet/node1 -> ./testnet/node11
# ./testnet/node2 -> ./testnet/node22
# reset the state for node11, node22

tgrade tendermint unsafe-reset-all --home=./testnet/node11/tgrade
tgrade tendermint unsafe-reset-all --home=./testnet/node22/tgrade

# keep only your node1 & node2 persistent peers in `./config/config.toml`
# `persistent_peers=""`

# create your ./testnet/validators.json with whitelisted bech32 validator addresses from node1 and node2 address and oversight member addresses e.g.
# {"validators":["tgrade1expjfylgfkrz2uc99w0s5zmng8f7km537erm4s","tgrade1wq8ja2239jzq3zfj8snmrku0cwnalpw85muc8n"],"oversight":["tgrade1c8jdd3xfzaq03fm6awf7j45zcvh49g7clhtc5r"]}
# get oversight member addresses from:
tgrade keys list --keyring-backend=test --home=./testnet/node0/tgrade
# get validator addresses from
tgrade keys list --keyring-backend=test --home=./testnet/node11/tgrade
tgrade keys list --keyring-backend=test --home=./testnet/node22/tgrade

# migrate your exported genesis
tgrade migrate-genesis-with-validatorset ./testnet/tgrade_state_export.json ./testnet/tgrade_state_export_migrated.json 2 ./testnet/validators.json && cp ./testnet/tgrade_state_export_migrated.json ./testnet/node11/tgrade/config/genesis.json && cp ./testnet/tgrade_state_export_migrated.json ./testnet/node22/tgrade/config/genesis.json

tgrade start --home=./testnet/node11/tgrade
tgrade start --home=./testnet/node22/tgrade

```
