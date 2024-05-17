# Bitcoin Monitor

This repository explains how to deploy a Bitcoin supernode, which connects to many other Bitcoin nodes for measuring block heights

## Build modified Bitcoin Binary

We modify exisitng `bitcoind` from [Bitcoin Core](https://github.com/bitcoin/bitcoin) for easier deployment.

Follow [Building section of Bitcoin doc](https://github.com/bitcoin/bitcoin/blob/master/doc/README.md#building) after modifying following codes

### Increase maximum outbound peers

We will add peers to the supernode manually via Bitcoin RPC `ADDNODE` command, so upper limit of manually added peers should be increased.

This can be set via `MAX_ADDNODE_CONNECTIONS` in `src/net.h`

```
/** Maximum number of addnode outgoing nodes */
static const int MAX_ADDNODE_CONNECTIONS = 8; // change this to large number
```

### Prevent a node from sending new block information to other blocks

By default, a `bitcoind` node does not update a peer's block height if the node send new block information to the peer.

We have discussed this issue (some supernodes have missed this, reporting wrong result) in the section 3.2 of [Baek, Seungjin, et al. "Short Paper: On the Claims of Weak Block Synchronization in Bitcoin." International Conference on Financial Cryptography and Data Security. 2022.](https://dl.acm.org/doi/abs/10.1007/978-3-031-18283-9_33)

Since the new block information can be sent via `CMPCTBLOCK`, `INV`, `HEADERS` messages, dropping the 3 messages types from outgoing messages is enough.

This can be done via checking `msg.command` in `CConnman::PushMessage` function in `src/net.cpp`, by injecting following lines:

```
void CConnman::PushMessage(CNode* pnode, CSerializedNetMsg&& msg)
{
    // drop 3 message types to prevent sending new block information to peers
    if(msg.command == NetMsgType::CMPCTBLOCK || msg.command == NetMsgType::INV || msg.command == NetMsgType::HEADERS) {
        return;
    }

...
```

## Running  a supernode

After executing `bitcoind`, you can regularly add new peer addresses via `ADDNODE` command using `bitcoin-cli`, where peer information can be found in [Snapshot API of bitnodes.io](https://bitnodes.io/api/) or other Bitcoin monitoring services.