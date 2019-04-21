# Transaction State Changes

Blocks contain transactions listed in a particular order, and at the Commit phase of consensus, this order is considered agreed-upon by the validators. While consensus is necessary to commit transactions to the blockchain, the distributed execution of each transaction’s action is necessary before the state changes in the application are finalized. The following ABCI methods implemented by the application are run, in order, by every node in the network upon receiving a valid block. Note that, while each node runs these processes locally, because all state transitions are deterministic and the order of transactions has already been established, this process yields a single, unambiguous new state.
* **BeginBlock** provides important information to the nodes running the application, including details about the block and consensus round. 
* **DeliverTx** is run for each transaction, in the order specified in the block, to create the state changes. These changes haven't been committed yet, 
* **EndBlock** is always run after all state transitions and can make changes to the validator set and governance, as well as whatever the application developer writes into the function.
* **Commit** finally commits the state changes resulting from the previous processes. As evidence of this commit, a merkle proof is returned.
