# Transaction Creation

Transactions are application-specific, but following the Cosmos SDK allows for interfacing with ABCI and, thus, Tendermint Core. 
An application developer defines [`Msgs`](https://github.com/cosmos/cosmos-sdk/blob/0c6d53dc077ee44ad72681b0bffafa1958f8c16d/types/tx_msg.go#L7-L31) by implementing the Msg interface, and the Msgs are bundled into transactions by the SDK. A user of the application sends Msgs and provides a maximum amount of gas he is willing to spend, GasWanted.
