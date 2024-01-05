# How to join Bluzelle Mainnet

## Inatall the Curiumd binary.
### Build from the source code.
#### Build Tools
Install make and gcc
```
sudo apt-get update

sudo apt-get install -y make gcc lz4
```
#### Install Go
Our current binary is built by Go `1.18.10`. We're going to download the tarball, extract it to `/usr/local`, and export `GOROOT` to our `$PATH`
```
wget https://go.dev/dl/go1.18.10.linux-amd64.tar.gz
sudo tar -C /usr/local -xvf go1.18.10.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

#### Install the binaries
Next, let's install the latest version of Bluzelle. Make sure you `git checkout` the correct released version.

```
git clone -b <latest-release-tag> https://github.com/bluzelle/bluzelle-public.git
cd bluzelle-public/curium && make install

```

Verify that everything installed successfully by running:
```
curiumd version --long
```

### Download binary from github releases.

```
wget <binary_url>
```

## General Configuration
Make sure to walk through the basic setup and configuration. Operators will need to initialize `curiumd`, download the genesis file for `bluzelle-9`.
### initialize chain
Choose a custom moniker for the node and initialize. By default, the `init` command creates the ~/.curium directory with subfolders `config` and `data`. In the `/config` directory, the most important files for configuration are `app.toml` and `config.toml`.

```
curiumd init <custom-moniker>
```
> Note: Monikers can contain only ASCII characters. Using Unicode characters is not supported and renders the node unreachable.

The `moniker` can be edited in the `~/.curium/config/config.toml` file:
```
# A custom human readable name for this node
moniker = "<custom_moniker>"
```

### Genesis File
Once the node is initialized, download the genesis file and move to the `/config` directory of the Curium home directory.
```
wget https://raw.githubusercontent.com/bluzelle/mainnet/main/genesis.json
mv genesis.json ~/.curium/config/genesis.json
```

### Seeds & Peers
Upon startup the node will need to connect to peers. If there are specific nodes a node operator is interested in setting as seeds or as persistent peers, this can be configured in `~/.ccurium/config/config.toml`

```
# Comma separated list of seed nodes to connect to
seeds = "<seed node id 1>@<seed node address 1>:26656,<seed node id 2>@<seed node address 2>:26656"

# Comma separated list of nodes to keep persistent connections to
persistent_peers = "<node id 1>@<node address 1>:26656,<node id 2>@<node address 2>:26656"
```

### Address Book
Node operators can optionally download the addressbook from [addrbook.json](https://temp)

```
wget https://raw.githubusercontent.com/bluzelle/mainnet/main/addrbook.json
mv addrbook.json ~/curium/config
```

### Pruning of State
> Note: This is an optional configuration.

There are four strategies for pruning state. These strategies apply only to state and do not apply to block storage. A node operator may want to consider custom pruning if node storage is a concern or there is an interest in running an archive node.
 To set pruning, adjust the `pruning` parameter in the `~/.curium/config/app.toml` file.
 The following pruning state settings are available:

 1. `everything`: Prune all saved states other than the current state.
 2. `nothing`: Save all states and delete nothing.
 3. `default`: Save the last 100 states and the state of every 10,100th block.
 4. `custom`: Specify pruning settings with the `pruning-keep-recent`, `pruning-keep-every`, and `pruning-interval` parameters.

 By default, every node is in default mode which is the recommended setting for most environments. If a node operator wants to change their node's pruning strategy then this must be done before the node is initialized.

 In `~/.curium/config/app.toml`
 ```
 # default: the last 100 states are kept in addition to every 500th state; pruning at 10 block intervals
# nothing: all historic states will be saved, nothing will be deleted (i.e. archiving node)
# everything: all saved states will be deleted, storing only the current state; pruning at 10 block intervals
# custom: allow pruning options to be manually specified through 'pruning-keep-recent', 'pruning-keep-every', and 'pruning-interval'
pruning = "custom"

# These are applied if and only if the pruning strategy is custom.
pruning-keep-recent = "10"
pruning-keep-every = "1000"
pruning-interval = "10"
 ```

 ## Sync Options
 There are three main ways to sync a node on the Bluzelle; Blocksync, StateSync and QuickSync.
 Among these three ways, the fastest one is QuickSync. QuickSync uses the compressed data snapshot and you can simply download that file and extract into the data folder. The next one is StateSync. The default one is BlockSync and it started from the Genesis file. This BlockSync could be used for setting up a archival node. 

 ### Blocksync
 Blocksync is faster than traditional consensus and syncs the chain from genesis by downloading blocks and verifying against the merkle tree of validators. For more information see [CometBFT's Fastsync Docs](https://docs.cometbft.com/v0.34/core/fast-sync)

 When syncing via Blocksync, node operators will either need to manually upgrade the chain or set up [Cosmovisor](https://hub.cosmos.network/main/hub-tutorials/join-mainnet.html#cosmovisor) to upgrade automatically.

 #### Getting Started Blocksync.
 This sync should start from the genesis file. So plz install corresponding binary version and start the node. Current corresponding version tag is `v9.0`.
 ```
 curiumd start
 ```
 The node will begin rebuilding state until it hits the first upgrade height at block 3,333,333. If Cosmovisor is set up then there's nothing else to do besides wait, otherwise the node operator will need to perform the manual upgrade.

 ### StateSync
 State Sync is an efficient and fast way to bootstrap a new node, and it works by replaying larger chunks of application state directly rather than replaying individual blocks or consensus rounds. For more information, see [CometBFT's State Sync Docs](https://docs.cometbft.com/v0.34/core/state-sync)

 To enable state sync, visit an explorer to get a recent block height and corresponding hash. A node operator can choose any height/hash in the current bonding period, but as the recommended snapshot period is `1000` blocks, it is advised to choose something close to `current height - 1000`. Current height can be get on the [Bluzelle Ping.pub Explorer](https://ping.explorer.net.bluzelle.com/Bluzelle/block)

 With the block height and hash selected, update the configuration in `~/.curium/config/config.toml` to set `enable = true`, and populate the `trust_height` and `trust_hash`. Node operators can configure the rpc servers to a preferred provider, but there must be at least two entries. It is important that these are two rpc servers the node operator trusts to verify component parts of the chain state. While not recommended, uniqueness is not currently enforced, so it is possible to duplicate the same server in the list and still sync successfully.

 ```
 #######################################################
###         State Sync Configuration Options        ###
#######################################################
[statesync]
# State sync rapidly bootstraps a new node by discovering, fetching, and restoring a state machine
# snapshot from peers instead of fetching and replaying historical blocks. Requires some peers in
# the network to take and serve state machine snapshots. State sync is not attempted if the node
# has any local state (LastBlockHeight > 0). The node will have a truncated block history,
# starting from the height of the snapshot.
enable = true

# RPC servers (comma-separated) for light client verification of the synced state machine and
# retrieval of state data for node bootstrapping. Also needs a trusted height and corresponding
# header hash obtained from a trusted source, and a period during which validators can be trusted.
#
# For Cosmos SDK-based chains, trust_period should usually be about 2/3 of the unbonding time (~2
# weeks) during which they can be financially punished (slashed) for misbehavior.
rpc_servers = "https://a.client.sentry.bluzelle.com:26657,https://b.client.sentry.bluzelle.com:26657"
trust_height = <trust_height>
trust_hash = "trust_hash"
trust_period = "168h0m0s"
 ```

 Start Cuium to begin state sync.  It may take take some time for the node to acquire a snapshot, but the command and output should look similar to the following:

 ```
 $ curiumd start --x-crisis-skip-assert-invariants

...

> INF Discovered new snapshot format=1 hash="0x000..." height=3663000 module=statesync

...

> INF Fetching snapshot chunk chunk=4 format=1 height=3663000  module=statesync total=45
> INF Applied snapshot chunk to ABCI app chunk=0 format=1 height=3663000  module=statesync total=45
 ```

 Once state sync successfully completes, the node will begin to process blocks normally. If state sync fails and the node operator encounters the following error: `State sync failed err="state sync aborted"`, either try restarting `curiumd` or running `curiumd tendermint unsafe-reset-all` (make sure to backup any configuration and history before doing this).

 ### Quick Sync
 You can get snapshot from [snapshot](https://genznodes.dev/resources/snapshot/bluzelle)

 You can replace the data folder into the node home directory.
 ```
 curl -o - -L https://snapshot.genznodes.dev/bluzelle/bluzelle-3693443.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.curium
 ```
Run your node again.
```
curiumd start
```
