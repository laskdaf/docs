# Transaction Creation
A [**transaction**](https://github.com/tendermint/tendermint/blob/master/types/tx.go) is comprised of user messages that will change the current state of the state machine and metadata. Not all transactions are transfers of tokens; they are specific to an application and thus can involve many types of state changes.

Transactions are application-specific, but following the Cosmos SDK allows for interfacing with ABCI and, thus, Tendermint Core. 
Using the SDK, an application developer defines `Msgs` by implementing the [Msg interface]((https://github.com/cosmos/cosmos-sdk/blob/0c6d53dc077ee44ad72681b0bffafa1958f8c16d/types/tx_msg.go#L7-L31)), and the Msgs are bundled into transactions by the SDK. A user of the application sends Msgs and provides a maximum amount of gas he is willing to spend, `GasWanted`. The gas will pay for the computational costs associated with processing this transaction. 
