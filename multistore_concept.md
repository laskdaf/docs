# Multistore

Every blockchain application stores **internal state**, which encompasses all of the data relevant to the application and is updated through transactions. Depending on the application, the internal state may store account balances, ownership of items, governance logic, etc. This doc describes the [Multistore interface](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L41-L66), a generalized data structure defined by the SDK for the purpose of storing internal state. State also includes [`Context`](https://github.com/cosmos/cosmos-sdk/blob/9a16e2675f392b083dd1074ff92ff1f9fbda750d/types/context.go#L29-L56), described in a separate doc. 

## Multistore Design
### Design Requirements 
To build a data structure for a blockchain application, there are several necessary functionalities. In a distributed computation model, state changes are executed by nodes locally before and after consensus, and executions may sometimes fail. Thus, the store should be able to make changes but also revert back to a previous state if needed. Also, following with the SDK's modular design, applications may use multiple modules to handle various functionalities, but those modules should not have access to portions of state that don't pertain to them.

Other functionalities, which are not included in the Multistore interface but also requirements for implemented stores, are also included below. They include the abilities to be queried by the consensus engine, to commit to a current state, and to be iterable. 

Below, the Multistore interface is described, with these requirements in mind.
```go
type MultiStore interface {
	Store
	CacheMultiStore() CacheMultiStore
	GetStore(StoreKey) Store
	GetKVStore(StoreKey) KVStore
	TracingEnabled() bool
	SetTracer(w io.Writer) MultiStore
	SetTracingContext(TraceContext) MultiStore
}
```

### Revert Capability and Tracing
To start off, Multistore is a [Store](https://github.com/cosmos/cosmos-sdk/blob/9a16e2675f392b083dd1074ff92ff1f9fbda750d/store/types/store.go#L12-L15) which means it has a [StoreType](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L201-L209) and is a [CacheWrapper](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L157-L178). As a `CacheWrapper`, a store can `CacheWrap` or be duplicated to make changes without affecting the underlying store, and can later `Write` or sync those changes to the underlying store. If a transaction makes a few state changes, then errors or runs out of gas, this Cache Wrapping functionality allows the state to be reverted back. 

Multistore also enables tracing of operations on the store: a `traceWriter` is a [`Writer`](https://golang.org/pkg/io/#Writer) that logs each change to the store, and `traceContext` is the state that is being traced. Both are initialized using their respective Setters, `SetTracer` and `SetTracingContext`.

### Access Management
Multistores may have multiple substores, which themselves can be Multistores or some other flavor of Stores. To enable access management, most applications have an `app.go` or main module that holds the application's internal state in a Multistore and controls access to each portion of it. To gain access to a store itself, a module calls a Getter function on the Multistore with the corresponding `StoreKey`. These functions make it convenient to access substores but restrict access to only modules that have the correct key.

### More
More functionality is described by other interfaces such as [Queryable](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L30-L36) which enables **ABCI queries**. Implementing ABCI is important for the application to communicate with the consensus engine (i.e. Tendermint). More on the ABCI can be found in [this doc](https://tendermint.com/docs/spec/abci/).

In blockchain applications, after blocks have been finalized, nodes not only execute the state changes of that block but also **commit** them. This means that, in the future, they should be able to prove that they arrived at this state. Thus, Multistores should also implement [CommitStore](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L17-L28), which enables commits - state persists after state changes are committed, and each one has a [CommitID](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L180-L197) which can be used to prove committed state changes. In practice, this typically includes a Merkle Root or Hash of the current state.

### KVStore
In most cases, instead of Store, the interface implemented is a [KVStore](https://github.com/cosmos/cosmos-sdk/blob/5344e8d768f306c29eb5451177499bfe540a80e9/store/types/store.go#L103-L133) which is the same as a Store but structured as an iterable key-value mapping (similar to a dictionary). The methods to retrieve, set, delete, or query the store all require a Key to be passed in. An Iterator must be implemented to enable iterating over the keys.

## Implementation
### Existing Stores
There are a few [stores](https://github.com/cosmos/cosmos-sdk/tree/9a16e2675f392b083dd1074ff92ff1f9fbda750d/store) in the SDK that implement the Multistore interface, each with varying functionalities and purposes.

### IAVL 
One example is [IAVL](https://github.com/cosmos/cosmos-sdk/blob/master/store/iavl/store.go), which implements the `KVStore` and  `CommitStore` interfaces, enables ABCI queries, and is structured as a self-balancing Merkle Tree. Information on the underlying IAVL tree can be found [here](github.com/tendermint/iavl). Read this doc further for how the IAVL Store implements the Multistore interfaces.

To implement a `Committer` for CommitStore, it obtains the current `hash` (which is the merkle root) and `version` of the underlying IAVL tree by calling [`SaveVersion`](https://github.com/tendermint/iavl/blob/de0740903a67b624d887f9055d4c60175dcfa758/mutable_tree.go#L320-L364) and returns it as a CommitID. The `LastCommitID` is retrieved simply by getting the tree's currently saved hash and version instead of calculating a new one.

To implement the `Store`, it returns `StoreTypeIAVL` as the StoreType and creates a new store as a copy of the current one in order to Cache Wrap.

To implement `KVStore`, it uses the underlying IAVL tree's `Set`, `Get`, `Has` and `Remove`, and defines an [IAVLIterator](https://github.com/cosmos/cosmos-sdk/blob/f4a96fd6b65ff24d0ccfe55536a2c3d6abe3d3fa/store/iavl/store.go#L256-L283) used to iterate through the data structure.

The IAVL Store also defines a [`Query`](https://github.com/cosmos/cosmos-sdk/blob/f4a96fd6b65ff24d0ccfe55536a2c3d6abe3d3fa/store/iavl/store.go#L178-L252) function to implement the ABCI interface. The interaction starts with a [`Request`](https://github.com/tendermint/tendermint/blob/4514842a631059a4148026ebce0e46fdf628f875/abci/types/types.pb.go) sent to the Store and ends with the Store outputting a [`Response`](https://github.com/tendermint/tendermint/blob/4514842a631059a4148026ebce0e46fdf628f875/abci/types/types.pb.go). `Query` first sets which height it will query the tree for, then gets the data appropriate for what the Request is asking for. It could ask for the value corresponding to a certain `Key`, as well as the Merkle Proof for it, or it may ask for `subspace`, which is the list of all `KVPairs` with a certain prefix. 

### BaseApp Usage
[BaseApp](https://github.com/cosmos/cosmos-sdk/blob/master/baseapp/baseapp.go), which implements basic functionalities for an ABCI application including stores, uses a [CommitMultiStore](https://github.com/cosmos/cosmos-sdk/blob/36dcd7b7ad94cf59a8471506e10b937507d1dfa5/store/types/store.go#L74-L98) (More about BaseApp can be found in [this doc](https://cosmos.network/docs/concepts/baseapp.html#baseapp)). Any application can use BaseApp. To initialize stores, the application can use [MountStores](https://github.com/cosmos/cosmos-sdk/blob/5344e8d768f306c29eb5451177499bfe540a80e9/baseapp/baseapp.go#L134-L153): the function takes as input any number of keys and, optionally, a already-existing database can be provided. Depending on the type of store the key is used for (IAVL, DB, or Transient), BaseApp will mount the stores and link the keys for access. The application is able to set the tracer, load the latest version, query the latest CommitID, set checkState, etc. by interfacing with BaseApp.

