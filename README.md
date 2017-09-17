# c-lightning: A specification compliant Lightning Network implementation in C -> ported to work with CHIPS

c-lightning is a [standard compliant](https://github.com/lightningnetwork/lightning-rfc) implementation of the Lightning Network protocol.
The Lightning Network is a scalability solution for Bitcoin, enabling secure and instant transfer of funds between any two party for any amount. 

For more information about the Lightning Network please refer to http://lightning.network.

## Project Status

This implementation is still very much work in progress, and, although it can be used for testing, __it should not be used for real funds__.
We do our best to identify and fix problems, and implement missing feature.

Any help testing the implementation, reporting bugs, or helping with outstanding issues is very welcome.
Don't hesitate to reach out to us on IRC at [#lightning-dev @ freenode.net](http://webchat.freenode.net/?channels=%23lightning-dev), [#c-lightning @ freenode.net](http://webchat.freenode.net/?channels=%23c-lightning), or on the mailing list [lightning-dev@lists.linuxfoundation.org](https://lists.linuxfoundation.org/mailman/listinfo/lightning-dev).

## Getting Started

c-lightning currently only works on Linux (and possibly Mac OS with some tweaking), and requires a locally running `chipsd` that is fully caught up with the network you're testing on.

### Installation

Please refer to the [installation documentation](doc/INSTALL.md) for detailed instructions.
For the impatient here's the gist of it for Ubuntu and Debian:

```
sudo apt-get install -y autoconf git build-essential libtool libgmp-dev libsqlite3-dev python python3
git clone https://github.com/jl777/lightning
cd lightning
make
```

### Starting `lightningd`

In order to start `lightningd` you will need to have a local `chipsd` node running:

```
chipsd -daemon
```


Once `chipsd` has synchronized with the network, you can start `lightningd` with the following command:

```
lightningd/lightningd --network=testnet --log-level=debug
```

### Opening a channel on the Bitcoin testnet

First you need to transfer some funds to `lightningd` so that it can open a channel:

```
# Returns an address <address>
cli/lightning-cli newaddr 

# Returns a transaction id <txid>
chips-cli sendtoaddress <address> <amount>

# Retrieves the raw transaction <rawtx>
chips-cli getrawtransaction <txid>

# Notifies `lightningd` that there are now funds available:
cli/lightning-cli addfunds <rawtx>
```

Eventually `lightningd` will include its own wallet making this transfer easier, but for now this is how it gets its funds.
If you don't have any testcoins you can get a few from a faucet such as [TPs' testnet faucet](http://tpfaucet.appspot.com/) or [Kiwi's testnet faucet](https://testnet.manu.backend.hamburg/faucet).

Once `lightningd` has funds, we can connect to a node and open a channel.
Let's assume the remote node is accepting connections at `<ip>:<port>` and has the node ID `<node_id>`:

```
cli/lightning-cli connect <ip> <port> <node_id>
cli/lightning-cli fundchannel <node_id> <amount>
```

This opens a connection and, on top of that connection, then opens a channel.
You can check the status of the channel using `cli/lightning-cli getpeers`.
The funding transaction needs to confirm in order for the channel to be usable, so wait a few minutes, and once that is complete it `getpeers` should say that the status is in _Normal operation_. 

### Receiving and receiving payments

Payments in Lightning are invoice based.
The recipient creates an invoice with the expected `<amount>` in millisatoshi and a `<label>`:

```
cli/lightning-cli invoice <amount> <label>
```

This returns a random value called `rhash` that is part of the invoice.
The recipient needs to communicate its ID `<recipient_id>`, `<rhash>` and the desired `<amount>` to the sender.

The sender needs to compute a route to the recipient, and use that route to actually send the payment:

```
route=$(cli/lightning-cli getroute <recipient_id> <amount> 1 | jq --raw-output .route -)
cli/lightning-cli sendpay $route <rhash>
```

Notice that in the first step we stored the route in a variable and reused it in the second step.
`lightning-cli` should return a preimage that serves as a receipt, confirming that the payment was successful.

This low-level interface is still experimental and will eventually be complemented with a higher level interface that is easier to use.

## Further information

JSON-RPC interface is documented in the following manual pages:

* [invoice](doc/lightning-invoice.7.txt)
* [listinvoice](doc/lightning-listinvoice.7.txt)
* [waitinvoice](doc/lightning-waitinvoice.7.txt)
* [delinvoice](doc/lightning-delinvoice.7.txt)
* [getroute](doc/lightning-getroute.7.txt)
* [sendpay](doc/lightning-sendpay.7.txt)

For simple access to the JSON-RPC interface you can use the `cli/lightning-cli` tool, or the [python API client](contrib/pylightning).
