# Transaction Lifecycle

We use `transaction` as a blanket term for anything that will change the current state of the state machine. Not all transactions transfer tokens, and each may be comprised of multiple messages requiring action from various modules.


## High Level Overview
1. **Creation:** Transactions are specified and created by a user of an application.
1. **Addition to Mempool:** Nodes validate transactions they receive, enabled by a check function implemented by the application, and broadcast them.
1. **Consensus:** The Proposer of the current round accumulates transactions into a block and validators in the network execute Tendermint BFT consensus to commit that block, thereby committing to the order of the transactions.
1. **State Changes:** The nodes running the application process the block’s transactions in order, deterministically committing to the new state of the application.

## Key Components
* The **ABCI** is an interface that allows for compatibility with nodes running Tendermint Core: nodes don’t need to know anything about the application but can easily query state, validate transactions, and execute transactions. Applications implement this interface.
* Applications built on the Cosmos SDK inherit from **BaseApp**, which implements ABCI for them. BaseApp includes a router to direct messages to their respective modules, Multistore to handle internal state, and handlers to implement the necessary logic for checking and executing transactions.
* **Internal state** is represented in a collection of KVStores each handled by a different module: e.g. if there is a currency, an auth module may keep track of keys and a bank module may keep track of account balances. There exist three distinct, simultaneous internal states during any given round that are synchronized after the completion of every Commit:
    * `CheckTxState` is maintained by the Mempool Connection and modified every time a transaction is validated.
    * `DeliverTxState` is maintained by the Consensus Connection and modified by transactions already committed in a block.
    * `QueryState` is maintained by the Query connection and used to query the last committed state.
## Transaction Lifecycle
### Creation
Transactions are specified and created by users of their applications (in this case built using the Cosmos SDK). A maximum amount of gas to be spent `GasWanted` is also provided. The originator of the transaction is a node that broadcasts the transaction, represented as a `[] bytes`, to its peers.
TODO: include code from nameservice or something
### Addition to Mempool
An ABCI procedure, `CheckTX`, interacts with the application to ensure the transaction is valid before the transaction is included in the Mempool. The default is for all nodes to broadcast via gossip all of the transactions that pass `CheckTx` and meet the  gas requirements.

`CheckTx` is handled by the **AnteHandler**, provided by BaseApp. Only the `checkTxState` is modified here and persist throughout this round’s series of `CheckTx`s; no actual state transitions happen yet. It also returns an estimated gas cost.

Tendermint enforces a maximum `GasWanted` per block; the application enforces that it is sufficient for `GasUsed`, i.e. enough funds were provided.

Nodes also keep a **Mempool cache**. It is not functionally necessary but used to prevent replay attacks; the node may keep 0 or any number of the most recent transactions.
### Consensus
Now we are entirely in the realm of Tendermint Core. The proposer designated for this round of consensus accumulates transactions it has seen and verified (assuming it is honest) in the block. The network goes through the pre-vote and pre-commit stages. Abstracting away some of the details, with 2/3 approval (by voting power) from the validators, the block is committed.
TODO: include code
### State Changes
The **Commit** Phase of consensus actually includes many steps, including the execution of each transaction’s action, before the state changes are committed. The following ABCI methods implemented by the application are requested, in order. Each is a Request sent to the application and returns a Response.
#### BeginBlock
BeginBlock communicates information such as block `header`, `hash`, `LastCommitInfo`, and `ByzantineValidators` to the application. The application may use this information in execution, to determine validator rewards/punishments or determine changes to the validator set.
#### DeliverTx
The`DeliverTx` for all transactions in the block are sent in sequential order as agreed upon by the block and processed by all the nodes maintaining the state machine for the application. Note that, since the state transitions for each transaction are deterministic, this should yield a single, unambiguous result across all nodes.

On the application side, both the **AnteHandler** and **MsgHandler** are run. It is possible for a transaction to have been invalid even though the proposer should have rejected it prior to inclusion in the block; if `CheckTx` does not pass here, TODO: find out what happens.

`BlockGasMeter` is used to keep track of how much gas is left for each transaction; `GasUsed` is deducted from it and returned in the Response. If `BlockGasMeter` runs out, the execution is terminated. If there is gas leftover after execution, it is returned to the user.

Since the SDK has a modular design, each transaction may have messages that need to be handled by different modules; the **router** provided by baseapp routes them appropriately.

#### EndBlock
`EndBlock` is always run at the end of the block and allows for automatic function calls (sparingly to avoid getting into infinite loops). More importantly, this function allows changes to the validator set by returning ValidatorUpdate objects or governnace changes by returning ConsensusParams.
#### Commit
A Commit Request is sent to application. After the state changes are made, a new state root should be sent back in Response to serve as merkle proof for state change. This completes the commit phase of consensus, and a new block can be proposed.


The transaction’s work is done, but its legacy lives on forever in the blockchain.
