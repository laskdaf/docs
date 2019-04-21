# Transaction State Changes

The **Commit** phase of consensus finalizes the order of transactions, but the distributed execution of each transaction’s action is necessary before the state changes are committed. The following ABCI methods implemented by the application are run, in order, by every node in the network upon receiving a valid block.
