# Multistore

Blockchain applications store **internal state**, which encompasses all of the data relevant to the application and is updated through transactions in each block. The SDK defines a [Multistore interface](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L41-L66) for this purpose. There are a few [stores](https://github.com/cosmos/cosmos-sdk/tree/9a16e2675f392b083dd1074ff92ff1f9fbda750d/store) in the SDK that implement this interface, each with varying functionalities.

## Multistore Design
The Multistore is designed to enable a few functionalities necessary for blockchain applications. In a distributed execution model, transactions are executed by nodes locally before and after consensus, and executions may sometimes fail. Thus, the store should be able to make changes but also revert back to a previous state if needed. Also, following with the SDK's modular design, applications may use multiple modules to handle various functionalities, but those modules should not have access to portions of state that don't pertain to them.

### Revert Capability and Logging
To start off, Multistore is a [Store](https://github.com/cosmos/cosmos-sdk/blob/9a16e2675f392b083dd1074ff92ff1f9fbda750d/store/types/store.go#L12-L15) that can be one of various [StoreTypes](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L201-L209) and is a [CacheWrapper](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L157-L178). As a `CacheWrapper`, the store can `CacheWrap` or be duplicated to make changes without affecting the underlying store, and can later `Write` or sync those changes to the underlying store. If a transaction makes a few state changes, then errors or runs out of gas, this Cache Wrapping functionality allows the state to be reverted back. There is also an option to `trace` operations that are done onto the store: a `traceWriter` is a `io.Writer` logs each change to the store, and `traceContext` is the state that is being traced. Both are initialized using a Setter and the context is updated as operations are done.

### Modularity and Access Management
To enable access management, most applications have an `app.go` or some main module that holds the application's internal state in a Multistore and controls access to each portion of the internal state. To gain access to a store itself, a module calls a Getter function on the Multistore with the corresponding `StoreKey`. The Store functions should not be leaking any of its keys.

### More
More functionality is added in other interfaces such as [Queryable](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L30-L36) which enables ABCI queries. Multistores should also implement [CommitStore](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L17-L28), which enables commits - state persists after state changes are committed, and each one has a [CommitID](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L180-L197). 

Multistores may have multiple substores, which themselves can be Multistores or some other flavor of them. [KVStore](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L103-L133) is a Store and an iterable key-value mapping. 


