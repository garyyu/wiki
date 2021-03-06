Running grin can be as simple as
```
target/debug/grin
```

or

```
cargo run
```

but to easily run grin with any of its commands and switches, we recommend you set up your PATH to include the target/debug folder where your grin binary is placed when you [build grin](https://github.com/mimblewimble/docs/wiki/Building) with `cargo build`.


## Configuration

Configure grin using `grin.toml` and on the command line.

At startup, grin looks for a configuration file called 'grin.toml' in the following places in the following order, using the first one it finds:

1. The current working directory
2. The directory in which the grin executable is located
3. {USER_HOME}/.grin

Command line switches can be used to override grin.toml settings.

If no configuration file is found, command line switches must be given to grin in order to start it. If a configuration file is found but no command line switches are provided, grin starts in server mode using the values found in the configuration file.

At present, the relevant modes of operation are 'server' and 'wallet'. When running in server mode, any command line switches provided will override the values found in the configuration file. Running in wallet mode does not currently use any values from the configuration file other than logging output parameters.

## Participate on Testnet2

Start by exploring the command line with:

```
grin help
grin wallet help
```

By default, the Grin wallet will only listen locally, to receive grins from another user, you will need the edit `grin.toml` and change the wallet listening address as below:

```
[wallet]
# Host IP for wallet listener, change to "0.0.0.0" to receive grins
api_listen_interface = "0.0.0.0"
```

As a user, you can try to:

- receive and send transactions (ask on Gitter to get some grins)
- view your wallets "outputs"
- sync the chain
  - fast sync
  - archive sync
- mine, using any of these PoW implementations:
  - cpu_mean_compat mining plugin (typical performance: 1000 graphs/s)
  - cpu_mean mining plugin (typical performance: 2000 graphs/s)
  - GPU miner

Ask about errors you see, and try re-telling the answers in your own words.
That brings attention to them, so we can make them more understandable for
users, and improve the page about
[troubleshooting](https://github.com/mimblewimble/docs/wiki/Troubleshooting).

### Sending/receiving grins
1. Verify that port 13415 is open. You may need to set up port forwarding on your network router. 
2. When transacting, both the sender and recipient should be running the latest version of the wallet code to ensure compatibility.

## Mining
1. Make sure you're synched up to the tip of the chain (Testnet1 - later on this will be more automatic...) by running `grin server run` and wait 1-2 hours or so until you reach the tip of chain and see the debug message "Disabling sync"
2. Look in the `target/debug/plugins` you find any mining plugings that have been built. Current fastest plugin is called lean or mean or so, and runs best on modern (2015+) Intel CPUs. Update grin.toml to use your preferred mining plugin, maybe by commenting _compat plugin and uncommenting the other faster plugin.

Be warned - you'll be mining test coins (no value) and there are bugs all over the place. Help us squish them! First see  [troubleshooting Testnet1](https://github.com/mimblewimble/docs/wiki/Testnet1-troubleshooting) for the easy ones, and then come over to the [Grin Lobby](https://gitter.im/grin_community/Lobby) and talk about your bug collection!

## Known problems
When you run into problems, please see [troubleshooting](https://github.com/mimblewimble/docs/wiki/Troubleshooting)
and ask on [gitter chat](https://gitter.im/grin_community/Lobby).

Especially if you're not sure you've found a new bug,
[please ask on the chat](https://gitter.im/grin_community/Lobby)
or on the [maillist](https://launchpad.net/~mimblewimble)

And before you file a new bug, please take a quick look through
[known bugs](https://github.com/mimblewimble/grin/issues?utf8=%E2%9C%93&q=label%3Abug+)
and other existing issues.

## Example with 3 local nodes

The following outlines a more advanced example simulating a multi-server network with transactions being posted.

For the sake of example, we're going to run three nodes with varying setups. Create two more directories beside your node1 directory, called node2 and node3. If you want to clear data from your previous run (or anytime you want to reset the blockchain and all peer data) just delete the wallet.dat file in the node1 directory and run rm -rf .grin to remove grin's database.

### Node 1: Genesis and Miner

As before, node 1 will create the blockchain and begin mining. As we'll be running many servers from the same machine, we'll configure specific ports for other servers to explicitly connect to.

First, we run a wallet server in listen mode to receive rewards on port 15000 (we'll log in debug mode for more information about what's happening)

    node1$ grin wallet -p "password" -a "http://127.0.0.1:10001" listen -l 15000

Then we start node 1 mining with its P2P server bound to port 10000 and its api server at 10001. We also provide our wallet address where we'll receive mining rewards. In another terminal:

    node1$ grin server -m -p 10000 -a 10001 -w "http://127.0.0.1:15000" run

### Node 2: Regular Node (not mining)

We'll set up Node 2 as a simple validating node (i.e. it won't mine,) but we'll pass in the address of node 1 as a seed. Node 2 will join the network founded by node 1 and then sync its blockchain and peer data.

In a new terminal, tell node 2 to run a server using node 1's P2P address as a seed.  Node 2's P2P server will run on port 20000 and its API server will run on port 20001.

    node2$ grin server -s "127.0.0.1:10000" -p 20000 -a 20001 run

Node 2 will then sync and process and validate new blocks that node 1 may find.

### Node 3: Regular node running wallet listener

Similar to Node 2, we'll set up node 3 as a non-mining node seeded with node 2 (node 1 could also be used). However, we'll also run another wallet in listener mode on this node:

    node3$ grin server -s "127.0.0.1:20000" -p 30000 -a 30001 run

Node 3 is now running it's P2P service on port 30000 and its API server on 30001. You should be able to see it syncing its blockchain and peer data with nodes 1 and 2. Now start up a wallet listener.

    node3$ grin wallet -p "password" -a "http://127.0.0.1:30001" listen -l 35000

In contrast to other blockchains, a feature of a MimbleWimble is that a transaction cannot just be directly posted to the blockchain. It first needs to be sent from the sender to the receiver,
who will add a blinding factor before posting it to the blockchain. The above command tells the wallet server to listen for transactions on port 35000, and, after applying it's own blinding factor to the transaction, forward them on to the listening API server on node 1. (NB: we should theoretically be able to post transactions to node 3 or 2, but for some reason transactions posted to peers don't seem to propagate properly at present)

### Node 1 - Send money to node 3

With all of your servers happily running and your terminals scrolling away, let's spend some of the coins mined in node 1 by sending them to node 3's listening wallet.

In yet another terminal in node 1's directory, create a new partial transaction spending 20000 coins and send them on to node 3's wallet listener. We'll also specify that we'll
use node 2's API listener to validate our transaction inputs before sending:

    node1$ grin wallet -p "password" -a "http://127.0.0.1:20001" send -d "http://127.0.0.1:35000" 20000

The grin.toml configuration file has further information about the various options available.