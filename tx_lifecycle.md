# Transaction Lifecycle

We use `transaction` as a blanket term for anything that will change the current state of the state machine. Not all transactions are transfers of tokens, and each wraps multiple `Msgs` specified by the application developer.


## High Level Overview
1. **Creation:** Transactions are comprised of multiple `Msgs` specified in an application.
1. **Addition to Mempool:** Nodes validate transactions they receive, enabled by a check function implemented by the application, and broadcast them.
1. **Consensus:** The Proposer of the current round accumulates transactions into a block and validators in the network execute Tendermint BFT consensus to commit that block, thereby committing to the order of the transactions.
1. **State Changes:** The nodes running the application process the block’s transactions in order, deterministically committing to the new state of the application.

## Key Components
* The **ABCI** is an interface that allows for compatibility with nodes running Tendermint Core: nodes don’t need to know anything about the application but can easily query state, validate transactions, and execute transactions. Applications implement this interface.
* Applications built on the Cosmos SDK inherit from **BaseApp**, which implements ABCI for them provided the application implements specified interfaces. [BaseApp](https://github.com/cosmos/cosmos-sdk/blob/cec3065a365f03b86bc629ccb6b275ff5846fdeb/baseapp/baseapp.go) includes, among other things, a Router to direct messages to their respective modules, Multistore to handle internal state, and Handlers to implement the necessary logic for checking and executing transactions.
* **Internal state** is represented in a collection of `KVStores` each handled by a different module: e.g. if there is a currency, an [auth](https://github.com/cosmos/cosmos-sdk/tree/develop/x/auth) module may keep track of keys and a [bank](https://github.com/cosmos/cosmos-sdk/tree/develop/x/bank) module may keep track of account balances. There exist three distinct, simultaneous internal states during any given round that are synchronized after the completion of every Commit:
    * `CheckTxState` is maintained by the Mempool Connection and modified every time a transaction is validated.
    * `DeliverTxState` is maintained by the Consensus Connection and modified by transactions already committed in a block.
    * `QueryState` is maintained by the Query connection and used to query the last committed state.
## Transaction Lifecycle
### Creation
Transactions are technically only touched by the SDK; it wraps and unwraps `Msgs` into transactions. An application developer defines `Msgs` by implementing an interface. A user of the application sends `Msgs` and provides maximum amount of gas to be spent `GasWanted`. The originator of the transaction is a node that broadcasts the transaction, represented as a `[] bytes`, to its peers.
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
An example `MsgBuyName` defined for the [Nameservice](https://cosmos.network/docs/tutorial/#requirements) application example:
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

A more complex example [MsgCreateValidator](https://github.com/cosmos/cosmos-sdk/blob/59765cecb11612a85d15acceb73bea677953057c/x/staking/types/msg.go#L23-L151) from the SDK.


### Addition to Mempool
An ABCI procedure, `CheckTX`, interacts with the application to ensure the transaction is valid before the transaction is included in the Mempool. The transaction is unwrapped into its comprising `Msgs` and `validateBasic`, a stateless validation function implemented by all `Msgs`, is run on each one. The default is for all nodes to broadcast via gossip all of the transactions that pass `CheckTx` and meet the  gas requirements.

`CheckTx` as implemented in BaseApp.
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

`CheckTx` is handled by the **AnteHandler**, provided by BaseApp. Only the `checkTxState` is modified here and persists throughout this round’s series of `CheckTxs`; no actual state transitions happen yet. It also returns an estimated gas cost.

Tendermint enforces a maximum `GasWanted` per block; the application enforces that it is sufficient for `GasUsed`, i.e. enough funds are provided.

Nodes also keep a **Mempool cache**: each may keep 0 or more of the most recent transactions. It is not functionally necessary but used to prevent replay attacks.
### Consensus
Now we are entirely in the realm of Tendermint Core. The proposer designated for this round of consensus accumulates transactions it has seen and verified (assuming it is honest) in the block. The network goes through the pre-vote and pre-commit stages. Abstracting away some of the details of [Tendermint BFT Consensus](https://tendermint.com/docs/spec/consensus/consensus.html#terms), with 2/3 approval (by voting power) from the validators, the block is committed.
### State Changes
The **Commit** Phase of consensus actually includes many steps, including the execution of each transaction’s action, before the state changes are committed. The following ABCI methods implemented by the application are requested, in order. Each is a Request sent to the application and returns a Response.

The Consensus Connection described by the [ABCI](https://github.com/tendermint/tendermint/blob/75ffa2bf1c7d5805460d941a75112e6a0a38c039/abci/types/application.go#L11-L26
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
BeginBlock communicates information such as block `header`, `hash`, `LastCommitInfo`, and `ByzantineValidators` to the application. The application may use this information in execution, to determine validator rewards/punishments or determine changes to the validator set. For example, it may decide to slash malicious validators.
#### DeliverTx
The`DeliverTx` for all transactions in the block are sent in sequential order as agreed upon by the block and processed by all the nodes maintaining the state machine for the application. Note that, since the state transitions for each transaction are deterministic, this should yield a single, unambiguous result across all nodes.

`DeliverTx` as implemented in BaseApp. Note the key difference with `CheckTx` is the argument `runTxModeDeliver` instead of `runTxModeCheck`. 
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

On the application side, both the **AnteHandler** and **MsgHandler** are run. It is possible for a transaction to have been invalid even though the proposer should have rejected it prior to inclusion in the block; if `CheckTx` does not pass here, it aborts without writing state changes.

`BlockGasMeter` is used to keep track of how much gas is left for each transaction; `GasUsed` is deducted from it and returned in the Response. If `BlockGasMeter` runs out, the execution is terminated. If there is gas leftover after execution, it is returned to the user.

Most of the logic is in [runTx](https://github.com/cosmos/cosmos-sdk/blob/cec3065a365f03b86bc629ccb6b275ff5846fdeb/baseapp/baseapp.go#L757-L873), shortened here for brevity:
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

Since the SDK has a modular design, each transaction may have `Msgs` that need to be handled by different modules. BaseApp includes a **Router** [interface](https://github.com/cosmos/cosmos-sdk/blob/59765cecb11612a85d15acceb73bea677953057c/baseapp/router.go) and each `Msg` must implement a `Route` function to return the name of the module to route the message to. 

#### EndBlock
`EndBlock` is always run at the end of the block and allows for automatic function calls (sparingly to avoid getting into infinite loops). More importantly, this function allows changes to the validator set by returning ValidatorUpdate objects or governnace changes by returning ConsensusParams.

#### Commit
A Commit Request is sent to application. After the state changes are made, a new state root should be sent back in Response to serve as merkle proof for state change. 
Commit as implemented in [BaseApp](https://github.com/cosmos/cosmos-sdk/blob/cec3065a365f03b86bc629ccb6b275ff5846fdeb/baseapp/baseapp.go#L888-L912):
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

This completes the commit phase of consensus, and a new block can be proposed. The transaction’s work is done, but its legacy lives on forever in the blockchain.
