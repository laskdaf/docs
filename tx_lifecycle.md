# Transaction Lifecycle

A **transaction** is comprised of user messages that will change the current state of the state machine. Not all transactions are transfers of tokens; they are specific to an application and thus can involve many types of state changes.


## High Level Overview
The transaction goes through several steps in order to be included in a block and its state changes committed by the application. The goal of the protocols is to ensure that only valid transactions approved by the network are committed, and that all nodes running the application locally agree on the current state. 
1. **Creation:** Transactions are comprised of multiple `Msgs` specified in an application.
2. **Addition to Mempool:** Nodes validate transactions they receive, enabled by a check function implemented by the application, and broadcast them.
3. **Consensus:** The Proposer of the current round accumulates transactions into a block and validators in the network execute Tendermint BFT consensus to commit that block, thereby committing to the order of the transactions.
4. **State Changes:** The nodes running the application process the block’s transactions in order locally, deterministically committing to the new state of the application.
These steps are executed separately by nodes in the network; the result is an agreed-upon state change. 

## Key Components
A few key components are necessary to understand a transaction's lifecycle and develop an application accordingly: the application's connections with Tendermint and how internal state is handled at each step in the transaction's lifecycle. 
* **ABCI** is an interface between Tendermint and the application. Nodes don’t need to know anything about the application but can easily query state, validate transactions, and execute transactions. On the other hand, all applications that implement this interface can utilize Tendermint. 
	* ABCI connects Tendermint and the application using methods grouped into [three connections](https://tendermint.com/docs/spec/abci/apps.html#state), each serving a different purpose in the transaction's lifecycle: The Mempool Connection validates transactions, the Consensus Connection delivers (executes) transactions, and the Info Connection queries state.
	* Each ABCI method inputs a Request to the application and returns a Response. 
* Applications built on the SDK inherit from **BaseApp**, which implements ABCI for them provided the application implements specified interfaces. [BaseApp](https://github.com/cosmos/cosmos-sdk/blob/cec3065a365f03b86bc629ccb6b275ff5846fdeb/baseapp/baseapp.go) includes, among other things, a [Router interface](https://github.com/cosmos/cosmos-sdk/blob/59765cecb11612a85d15acceb73bea677953057c/baseapp/router.go) to direct messages to their respective modules, `Multistore` to handle internal state, and Handlers to implement the necessary logic for checking and executing transactions.
* Each node locally maintains an **Internal state** of the application, and updates it at every commit. It is represented in a collection of [`KVStores`](https://github.com/tendermint/tendermint/blob/c086d0a34102bd42873d20445673ea1d18a539cd/abci/example/kvstore/kvstore.go) each handled by a different module: e.g. if there is a currency, an [auth](https://github.com/cosmos/cosmos-sdk/tree/develop/x/auth) module may keep track of keys and a [bank](https://github.com/cosmos/cosmos-sdk/tree/develop/x/bank) module may keep track of account balances. Because nodes come across transactions at different stages in their lifecycel, there exist three distinct, simultaneous internal states during any given round that are synchronized after the completion of every Commit:
    * `CheckTxState` is maintained by the Mempool Connection and modified every time a transaction is validated. Necessary because some transactions may be affected by preceding transactions in the same block.
    * `DeliverTxState` is maintained by the Consensus Connection and only modified by transactions already committed in a block.
* [**Gas**](https://tendermint.com/docs/spec/abci/apps.html#transaction-results) is used in Tendermint similarly to Ethereum. Users pay a gas fee for the distributed execution of their transactions: `GasWanted` is the maximum they will pay, and `GasUsed` is how much was actually used in execution. Gas is calculated in proportion to the resources needed to process transactions, and helps prevent DOS attacks. 
## Transaction Lifecycle
### Creation
Transactions are application-specific, but follow a certain template in the SDK to fit with aforementioned components. An application developer defines `Msgs` by implementing the `Msg` interface, and the `Msgs` are bundled into transactions by the SDK. A user of the application sends `Msgs` and provides a maximum amount of gas he is willing to spend, `GasWanted`. 

Here is the `Msg` interface. `Route` gives information on which module this `Msg` is handled by. `ValidateBasic` is a check on the validity of a message that should not require state, such as checking for nil strings or negative sums. `GetSigners` returns the addresses of users that must provide signatures for the message to be validated. 
```go
// Transactions messages must fulfill the Msg
type Msg interface {

	// Return the message type.
	// Must be alphanumeric or empty.
	Route() string

	// Returns a human-readable string for the message, intended for utilization
	// within tags
	Type() string

	// ValidateBasic does a simple validation check that
	// doesn't require access to any other information.
	ValidateBasic() Error

	// Get the canonical byte representation of the Msg.
	GetSignBytes() []byte

	// Signers returns the addrs of signers that must sign.
	// CONTRACT: All signatures must be present to be valid.
	// CONTRACT: Returns addrs in some deterministic order.
	GetSigners() []AccAddress
}
```
Here is an example of a `Msg`, `MsgBuyName`, defined for the [Nameservice](https://cosmos.network/docs/tutorial/#requirements) application. The logic here represents a buyer bidding on a specific name. The message must be signed by the buyer, the bid must be valid (positive bid for a nonempty name), and this message is handled by the nameservice module:
```go
type MsgBuyName struct {
	Name  string
	Bid   sdk.Coins
	Buyer sdk.AccAddress
}

func NewMsgBuyName(name string, bid sdk.Coins, buyer sdk.AccAddress) MsgBuyName {
	// ...
}

func (msg MsgBuyName) Route() string { return "nameservice" }

func (msg MsgBuyName) Type() string { return "buy_name" }

func (msg MsgBuyName) ValidateBasic() sdk.Error {
	if msg.Buyer.Empty() { return sdk.ErrInvalidAddress(msg.Buyer.String()) }
	if len(msg.Name) == 0 { return sdk.ErrUnknownRequest("Name cannot be empty") }
	if !msg.Bid.IsAllPositive() { return sdk.ErrInsufficientCoins("Bids must be positive") }
	return nil
}

func (msg MsgBuyName) GetSignBytes() []byte {
	b, err := json.Marshal(msg)
	if err != nil { panic(err) }
	return sdk.MustSortJSON(b)
}

func (msg MsgBuyName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Buyer}
}

```


The node from which a transaction originates broadcasts the transaction, represented as a `[] bytes`, to its peers.

### Addition to Mempool
Nodes in the network receive transactions from their peers, and validate them locally before including them into the Mempool. An ABCI procedure, `CheckTX`, is utilized (see below for code). The transaction is unwrapped into its comprising `Msgs` and `validateBasic`, a stateless validation function implemented by all `Msgs`, is run on each one.  The default is for all nodes to broadcast via gossip all of the transactions that pass `CheckTx` and meet the  gas requirements.

Here is `CheckTx` as implemented in BaseApp. First, the application method `txDecoder` is run to decode `[] bytes` into its respective messages. Then, if there is no error, [runTx](https://github.com/cosmos/cosmos-sdk/blob/9036430f15c057db0430db6ec7c9072df9e92eb2/baseapp/baseapp.go#L814-L855) in the Check mode is run on the transaction: it simply runs `validateBasic` and **AnteHandler**, which returns the Result. 
```go
// NOTE:CheckTx does not run the actual Msg handler function(s).
func (app *BaseApp) CheckTx(txBytes []byte) (res abci.ResponseCheckTx) {
	var result sdk.Result

	tx, err := app.txDecoder(txBytes)
	if err != nil {
		result = err.Result()
	} else {
		result = app.runTx(runTxModeCheck, txBytes, tx)
	}

	return abci.ResponseCheckTx{
		Code:      uint32(result.Code),
		Data:      result.Data,
		Log:       result.Log,
		GasWanted: int64(result.GasWanted), 
		GasUsed:   int64(result.GasUsed),   
		Tags:      result.Tags,
	}
}
```
Only the `checkTxState` is modified here and persists throughout this round’s series of `CheckTxs`. No committed state transitions have happened yet - in fact, as each node is running this locally, each may have a different `checkTxState` from other nodes in the network. This will be fixed in the next step.
Included in the Result is an estimated gas cost. The application enforces that `GasWanted` is sufficient for `GasUsed`, i.e. enough funds are provided to complete execution of the transaction.

Nodes also keep a **Mempool cache**: each may keep 0 or more of the most recent transactions. Thus, the transaction may pass through a node's Mempool cache check before CheckTx.
### Consensus
Now we are entirely in the realm of Tendermint Core, and this step is only executed by the validators. The proposer (presumably honest) designated for this round of consensus accumulates transactions it has seen and verified in the block. The validators go through the pre-vote and pre-commit stages. Abstracting away some of the details of [Tendermint BFT Consensus](https://tendermint.com/docs/spec/consensus/consensus.html#terms), with 2/3 approval (by voting power) from the validators, the block is committed.
### State Changes
The **Commit** phase of consensus finalizes the ordering of transactions, but the distributed execution of each transaction’s action is necessary before the state changes are committed. The following ABCI methods implemented by the application are run, in order, by every node in the network.

For starters, here is the code for the Consensus Connection described by [ABCI](https://github.com/tendermint/tendermint/blob/75ffa2bf1c7d5805460d941a75112e6a0a38c039/abci/types/application.go#L11-L26
).
```go
type Application interface {
	// ... Mempool Connection and  Query Connection
	// Consensus Connection
	InitChain(RequestInitChain) ResponseInitChain    // Initialize blockchain with validators and other info from TendermintCore
	BeginBlock(RequestBeginBlock) ResponseBeginBlock // Signals the beginning of a block
	DeliverTx(tx []byte) ResponseDeliverTx           // Deliver a tx for full processing
	EndBlock(RequestEndBlock) ResponseEndBlock       // Signals the end of a block, returns changes to the validator set
	Commit() ResponseCommit                          // Commit the state and return the application Merkle root hash
}
```
#### BeginBlock
BeginBlock communicates information such as block `header`, `hash`, `LastCommitInfo`, and `ByzantineValidators` to the application. The application may use this information in execution, to determine validator rewards/punishments or determine changes to the validator set. For example, it may decide to slash malicious validators. No transactions are handled here.
#### DeliverTx
The`DeliverTx` for all transactions in the block are run in sequential order as agreed upon by the block and processed by all the nodes maintaining the state machine for the application. Note that, while each node is running these locally, since the state transitions for each transaction are deterministic, this should yield a single, unambiguous result across all nodes.

`DeliverTx` as implemented in BaseApp. Note that it is extremely similar to `CheckTx` except for the argument `runTxModeDeliver` instead of `runTxModeCheck`. This means that `runTx` will not halt after running anteHandler, and will continue with msgHandler. 
```go
// DeliverTx implements the ABCI interface.
func (app *BaseApp) DeliverTx(txBytes []byte) (res abci.ResponseDeliverTx) {
	var result sdk.Result

	tx, err := app.txDecoder(txBytes)
	if err != nil {
		result = err.Result()
	} else {
		result = app.runTx(runTxModeDeliver, txBytes, tx)
	}

	return abci.ResponseDeliverTx{
		//...
	}
}
```

It is possible for a transaction to have been invalid even though the proposer should have rejected it prior to inclusion in the block; if `CheckTx` does not pass here, it aborts without writing state changes.

`BlockGasMeter` is used to keep track of how much gas is left for each transaction; `GasUsed` is deducted from it and returned in the Response. If `BlockGasMeter` runs out, the execution is terminated. If there is gas leftover after execution, it is returned to the user.

Most of the logic is in [runTx](https://github.com/cosmos/cosmos-sdk/blob/cec3065a365f03b86bc629ccb6b275ff5846fdeb/baseapp/baseapp.go#L757-L873), shortened here for brevity. It can halt at various points based on errors or the mode (i.e. check vs run vs simulate run), and checks gas as needed. It first runs `validateBasic`, then the anteHandler, then [runMsgs](https://github.com/cosmos/cosmos-sdk/blob/9036430f15c057db0430db6ec7c9072df9e92eb2/baseapp/baseapp.go#L662-L720), then writes to the state.:
```go 
func (app *BaseApp) runTx(mode runTxMode, txBytes []byte, tx sdk.Tx) (result sdk.Result) {
	var gasWanted uint64
	ctx := app.getContextForTx(mode, txBytes)
	ms := ctx.MultiStore()
   
	if mode == runTxModeDeliver && ctx.BlockGasMeter().IsOutOfGas() {
		result = sdk.ErrOutOfGas("no block gas left to run tx").Result()
		return
	}
	
	// ... gas logic, catches errors and calculates GasWanted and GasUsed 

	var msgs = tx.GetMsgs()
	if err := validateBasicTxMsgs(msgs); err != nil { return err.Result() }

	if app.anteHandler != nil {
		var anteCtx sdk.Context
		var msCache sdk.CacheMultiStore
		anteCtx, msCache = app.cacheTxContext(ctx, txBytes)
		newCtx, result, abort := app.anteHandler(anteCtx, tx, mode == runTxModeSimulate)
		if !newCtx.IsZero() { ctx = newCtx.WithMultiStore(ms) }
		gasWanted = result.GasWanted
		if abort { return result }
		msCache.Write()
	}
   
	if mode == runTxModeCheck { return }

	runMsgCtx, msCache := app.cacheTxContext(ctx, txBytes)
	result = app.runMsgs(runMsgCtx, msgs, mode)
	result.GasWanted = gasWanted

	if mode == runTxModeSimulate { return }

	// only update state if all messages pass
	if result.IsOK() {
		msCache.Write()
	}
	return
}
```

#### EndBlock
[`EndBlock`](https://github.com/cosmos/cosmos-sdk/blob/9036430f15c057db0430db6ec7c9072df9e92eb2/baseapp/baseapp.go#L875-L886) is always run at the end of the block and allows for automatic function calls (sparingly to avoid getting into infinite loops). More importantly, this function allows changes to the validator set or governance.

#### Commit
The application `Commit` is run. After the state changes are made, a new state root should be sent back to serve as merkle proof for state change. 
Here is `Commit` as implemented in [BaseApp](https://github.com/cosmos/cosmos-sdk/blob/cec3065a365f03b86bc629ccb6b275ff5846fdeb/baseapp/baseapp.go#L888-L912). It synchronizes all the states by writing the `deliverTxState` into the application's internal state, updating both `checkTxState` `deliverTxState` afterward.:
```go

func (app *BaseApp) Commit() (res abci.ResponseCommit) {
	header := app.deliverState.ctx.BlockHeader()

	// write the Deliver state and commit the MultiStore
	app.deliverState.ms.Write()
	commitID := app.cms.Commit()
	app.logger.Debug("Commit synced", "commit", fmt.Sprintf("%X", commitID))

	// Reset the Check state to the latest committed.
	// NOTE: safe because Tendermint holds a lock on the mempool for Commit.
	// Use the header from this latest block.
	app.setCheckState(header)

	// empty/reset the deliver state
	app.deliverState = nil

	return abci.ResponseCommit{
		Data: commitID.Hash,
	}
}
```
Both the transaction is committed in the blockchain and its state changes committed in the application.
