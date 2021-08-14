# Overview

Vaults are a security mechanism to prevent cryptocurrency from being immediately withdrawn. When users want to withdraw some crypto from a vault, they must first issue a request, and the withdrawal is finalized only after a certain wait time has passed since the request. During the wait time, the request can be cancelled by using a recovery key. Vaults mitigate the risk that the user's private key is stolen: whenever an adversary attempts to withdraw using a stolen private key, the legit user can cancel the operation through the recovery key.

Vaults are quite popular in blockchain ecosystems. For instance, they are available on the following crypto wallets:
* [Coinbase](https://help.coinbase.com/en/coinbase/getting-started/other/vaults-faq)
* [Bitcoin Suisse](https://www.bitcoinsuisse.com/vault)
* [Electrum Vault](https://github.com/bitcoinvault/electrum-vault)

The purpose of this tutorial is to create a **decentralized** vault as an Algorand smart contract.

Since the TEAL implementation of vaults is quite complex, we first specify their functionality in AlgoML (after *Algorand Modelling Language*), a novel DSL for Algorand contracts, that compiles into TEAL scripts.

## Table of contents
- [Overview](#overview)
  - [Table of contents](#table-of-contents)
- [AlgoML specification](#algoml-specification)
  - [Contract state](#contract-state)
  - [Escrow account](#escrow-account)
  - [Creating the vault](#creating-the-vault)
  - [Initializing the escrow](#initializing-the-escrow)
  - [Depositing funds](#depositing-funds)
  - [Requesting a withdrawal](#requesting-a-withdrawal)
  - [Finalizing a request](#finalizing-a-request)
  - [Cancelling a request](#cancelling-a-request)
- [TEAL implementation](#teal-implementation)
  - [Escrow account](#escrow-account-1)
  - [Creating the vault](#creating-the-vault-1)
  - [Escrow initialization](#escrow-initialization)
  - [Requesting a withdrawal](#requesting-a-withdrawal-1)
  - [Finalizing a request](#finalizing-a-request-1)
  - [Cancelling a request](#cancelling-a-request-1)
- [Calling the functions](#calling-the-functions)
  - [Creating the vault](#creating-the-vault-2)
  - [Initializing the escrow](#initializing-the-escrow-1)
  - [Depositing funds](#depositing-funds-1)
  - [Requesting a withdrawal](#requesting-a-withdrawal-2)
  - [Finalizing a request](#finalizing-a-request-2)
  - [Cancelling a request](#cancelling-a-request-2)
- [Full Code](#full-code)
  - [AlgoML](#algoml)
  - [Teal](#teal)
    - [Stateful contract's approval program](#stateful-contracts-approval-program)
    - [Stateful contract's clear program](#stateful-contracts-clear-program)
    - [Stateless contract](#stateless-contract)


# AlgoML specification

We specify vaults in [AlgoML](https://github.com/petitnau/algoml), a high-level DSL for Algorand contracts that compiles into TEAL. Roughly, an AlgoML specification is a sequence of clauses of the form:
```java
@precondition1
...
@preconditionK
foo(x1,...,xn) {
  // state update
  ...
}
```
The intuition is that the function ``foo`` is enabled whenever all the preconditions are respected. Executing ``foo`` results in a state update, specified in the function body. Preconditions may have various forms: for instance, they can be predicates on the contract state, or checks that certain transactions belong to the group where the function is called.

We refer to the [AlgoML whitepaper](https://notion.so/petitnau/Alg0ML) for More details on the use of AlgoML.

## Contract state

The contract state is stored in the following variables:

* `wait_time` is the withdrawal wait time, i.e. the number of rounds that must pass between a withdrawal request and its finalization
* `recovery` is the address from which the cancel action must originate
* `vault` is the address of the escrow account where the deposited funds are stored
* `request_time` is the round at which the withdrawal request has been submitted
* `amount` is the amount of algos to be withdrawn
* `receiver` is the address which can withdraw the algos
* `gstate` is the contract state: 
  * `init_escrow`: the contract is waiting for escrow initialization
  * `waiting`: there is no pending withdrawal request
  * `requested`: there is a pending withdrawal request

The variables `wait_time`, `recovery` and `vault` are initialized at contract creation, and they remain constant throughout the contract lifetime. Instead, the other variables are updated upon each withdrawal request.

## Escrow account

The escrow account used by the vault is a stateless contract that releases funds provided that:
1. the stateful contract participates in the transaction group
1. the escrow does not pay any transaction fees
1. the escrow does not send a rekey transaction

## Creating the vault

Any user can create a vault, providing the recovery address and the withdrawal wait time. 
<!-- Once the contract gets created, the user will need to provide the escrow account address (it won't be possible to provide it on creation, since the escrow account needs the application ID). -->

We specify the behaviour of this action in AlgoML as follows:
```java
@gstate ->init_escrow
Create vault(address recovery, int wait_time) {
    glob.recovery = recovery
    glob.wait_time = wait_time
}
```
The `create` function requires two parameters: the recovery address, and the withdrawal wait time. When this function is called, the contract is created and its two global state variables `recovery` and `wait_time` are initializated.
The clause 
```java
@gstate ->init_escrow
```
means that after the create action is performed, the new state of the contract is `init_escrow`. 

## Initializing the escrow 

Once the contract has been created, the vault creator must invoke the `set_escrow` function to connect the contract with an escrow account. The escrow will store all the funds deposited in the vault.
To call the `set_escrow` function, the contract must be in the `init_escrow` state, and must be called from the contract creator. The application call must also come bundled with a pay transaction, with an amount of 100'000 micro-algos (the amount needed to initialize an account). When called, the vault address is saved into the global state, and the contract state is set to `waiting` (waiting for a withdrawal request).
This is specified in AlgoML as follows: 
```java
@gstate init_escrow->waiting
@from creator
@pay 100000 : * -> *$vault
set_escrow() {
    glob.vault = vault
}
```

The three preconditions have the following meaning: 
* `@gstate init_escrow->waiting`: the current contract state must be `init_escrow` (the next state will be `waiting`)
* `@from creator`: only the vault creator can call this function.
* `@pay 100000 : * -> *$vault`: 100.000 micro-algos must be deposited in the contract, and this can be done by *any* user. 

## Depositing funds 

Any user can deposit algos into the vault. Since paying algos to an account cannot be constrained in Algorand, this part of the specification is given by default, so it does not require a specific clause in AlgoML. 

## Requesting a withdrawal

Once the contract is created and the escrow connected to the contract, the vault creator can request a withdrawal. This requires the creator to declare the `amount` of algos to be withdrawn, and the address of the `receiver`. The contract stores these values, as well as the round when the withdrawal is requested. This is specified in AlgoML as follows:

```java
@gstate waiting->requested
@round *$curr_round
@from creator
withdraw(int amount, address receiver) {
    glob.amount = amount
    glob.receiver = receiver
    glob.request_time = curr_round
}
```
The `withdraw` function can only be called by the creator, and only while the contract is in state waiting for a withdrawal request. The function body saves the values of the parameters and the current round in the contract state. The precondition `@gstate waiting->requested` also ensures that the next state will be `requested`. Note that we do *not* require that the vault contains at least `amount` algos: indeed, this is not strictly necessary, as the creator can fund the vault after the request has been made.

## Finalizing a request

After the withdrawal wait period has passed, the vault creator can ask for the request to be finalized, thus receiving the funds and putting the contract back into a state where other requests are accepted.

```java
@gstate requested->waiting
@round (glob.request_time + glob.wait_time,)
@from creator
@pay glob.amount : vault -> glob.receiver
finalize() { }
```

The `finalize` function can only be called by the vault creator while, provided that the current state is `requested`, and `wait_time` rounds have passed since the `requested_time`. To call this function, the function call must be bundled with a pay transaction , as required by the precondition:
```java
@pay glob.amount : vault -> glob.receiver
```
This transaction must transfer the amount of algos specified in the request from the vault to the declared receiver. After the function call, the contract state is set to `waiting`, where the contract can accept another request.

## Cancelling a request 

The vault creator can abort an unexpected withdrawal (which probably means that that someone has access to the private key of the creator). This is done by calling the `cancel` function, which transfers all the funds to a recovery account. Since the private key of the recovery account is needed to cancel a withdrawal request, an adversary who steals the private key of the vault creator but not the recovery key will not be able to cancel withdrawal requests.

```java
@gstate requesting->waiting
@from glob.recovery
cancel() { }
```

The preconditions ensure that the function is called from the recovery address, and only after a withdrawal request has been sent (i.e., only when the contract is in the `requesting` state). After the call, the contract will return to the `waiting` state, thus disabling the finalize function.


# TEAL implementation

The following code, apart from the comments, is completely generated by the AlgoML compiler.

The TEAL code for the escrow account gives all of its "decisional power" to the stateful contract, which will control the escrow's spendings.

The TEAL code for the stateful contract is split into blocks, one per AlgoML function. 
Each block, is split into two parts: the head for dispatching, and the body for state changes. 

The program starts by checking the requirements of the first block. If at least one of the requirements in the head of the block are not met, the program jumps to the next block, and its requirements are checked.
If all the requirements of a block are met, the program will execute the body of the block, before approving the transaction.

If instead the transaction doesn't meet all the requirements of any block, it fails. 

## Escrow account

```java
#pragma version 3

// aseert that the stateful contracts participates in the transaction group
gtxn 1 TypeEnum
int appl
==
assert

gtxn 1 ApplicationID
int <APP-ID>
==
assert

// assert that this transaction is "non-rekeying"
txn RekeyTo
global ZeroAddress
==
assert

// assert that no fee is paid by this contract (all fees must be paid by the caller)
txn Fee
int 0
==
assert

// approve
int 1
```

## Creating the vault

```java
//* Check if we're calling the create function *//

// check if there are no other transactions in this atomic group
global GroupSize
int 1
==
bz not_create

// check if the application is being created with this transaction
txn ApplicationID
int 0
==
bz not_create

// check if the call has 3 arguments ("vault" + the two actual arguments: recovery account and wait_time)
txn NumAppArgs
int 3
==
bz not_create

txna ApplicationArgs 0
byte "vault"
==
bz not_create

//* Change the contract state *//

// set the contract state to init_escrow (waiting for escrow initialization)
byte "gstate"
byte "init_escrow"
app_global_put

// save the first argument (the address of the recovery account) into the global variable "recovery"
byte "recovery"
txna ApplicationArgs 1
app_global_put

// save the second argument (the time that must pass between withdrawal request and finalization) into the global variable "wait_time"
byte "wait_time"
txna ApplicationArgs 2
btoi
app_global_put

b approve
```

## Escrow initialization

```java
not_create:

//* Check if we're calling the init_escrow function *//

// check if there is one other transactions in this atomic group: the payment transaction needed to initialize the escrow account
global GroupSize
int 2
==
bz not_setescrow

// check if the application is currently waiting to initialize the escrow
byte "gstate"
app_global_get
byte "init_escrow"
==
bz not_setescrow

// check if this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_setescrow

// check if the other transaction is a pay transaction of 100'000 algos (the amount required to initialize an account)
gtxn 0 TypeEnum
int pay
==
bz not_setescrow

gtxn 0 Amount
int 100000
==
bz not_setescrow

// check if the other transaction is not a closing transaction
gtxn 0 CloseRemainderTo
global ZeroAddress
==
bz not_setescrow

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_setescrow

// check if the application call has 1 argument: the byte string "set_escrow"
txn NumAppArgs
int 1
== 
bz not_setescrow

txna ApplicationArgs 0
byte "set_escrow"
==
bz not_setescrow

//* Change the contract state *//

// set the contract state to waiting (waiting for a withdrawal request)
byte "gstate"
byte "waiting"
app_global_put

// save the vault address into the global state
byte "vault"
gtxn 0 Receiver
app_global_put

b approve
```

## Requesting a withdrawal

```java
not_setescrow:

//* Check if we're calling the withdraw function *//

// check if there are no other transactions in this atomic group
global GroupSize
int 1
==
bz not_withdraw

// check if the contract is in the waiting state (waiting for a withdrawal request)
byte "gstate"
app_global_get
byte "waiting"
==
bz not_withdraw

// check if this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_withdraw

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_withdraw

// check if the call has 3 arguments ("withdraw" and the two actual arguments: amount and receiver)
txn NumAppArgs
int 3
==
bz not_withdraw

txna ApplicationArgs 0
byte "withdraw"
==
bz not_withdraw

//* Change the contract state *//

// set the contract state to requested (withdrawal request ongoing)
byte "gstate"
byte "requested"
app_global_put

// save the first argument (the amount that the user is requesting) into global amount
byte "amount"
txna ApplicationArgs 1
btoi
app_global_put

// save the second argument (the receiver of the withdrawal) into global receiver
byte "receiver"
txna ApplicationArgs 2
app_global_put

// save the current round into global request_time (so that the request can only be finalized wait_time rounds after)
byte "request_time"
global Round
app_global_put

b approve
```

## Finalizing a request

```java
not_withdraw:

//* Check if we're calling the finalize function *//

// check if the application call transactions is bundled with a pay transaction (the withdrawal)
global GroupSize
int 2
==
bz not_finalize

// check if the contract is in the requested state (a withdrawal has been requested)
byte "gstate"
app_global_get
byte "requested"
==
bz not_finalize

// check if the withdrawal wait time has passed since the withdraw request
global Round
byte "request_time"
app_global_get
byte "wait_time"
app_global_get
+
>=
bz not_finalize

// check if this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_finalize

// check if the other transaction is a pay transaction from the escrow account to the requested receiver of the amount previously requested 
gtxn 0 TypeEnum
int pay
==
bz not_finalize

gtxn 0 Amount
byte "amount"
app_global_get
==
bz not_finalize

gtxn 0 Sender
byte "vault"
app_global_get
==
bz not_finalize

gtxn 0 Receiver
byte "receiver"
app_global_get
==
bz not_finalize

// check if the pay transaction is non-closing
gtxn 0 CloseRemainderTo
global ZeroAddress
==
bz not_finalize

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_finalize

// Check if the call has 1 argument ("finalize")
txn NumAppArgs
int 1
==
bz not_finalize

txna ApplicationArgs 0
byte "finalize"
==
bz not_finalize

//* Change the cntract state *//

// set the contract state back to waiting (waiting for a withdrawal request to be made)
byte "gstate"
byte "waiting"
app_global_put

b approve
```

## Cancelling a request

```java
not_finalize:

//* Check if we're calling the cancel function *//

// check if there aren't other transactions in this atomic group
global GroupSize
int 1
==
bz not_cancel

// check if the contract is in state requested (withdrawal request ongoing)
byte "gstate"
app_global_get
byte "requested"
==
bz not_cancel

// check if this transaction is sent by the recovery account
txn Sender
byte "recovery"
app_global_get
==
bz not_cancel

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_cancel

// check if the application call has 1 argument ("cancel")
txn NumAppArgs
int 1
==
bz not_cancel

txna ApplicationArgs 0
byte "cancel"
==
bz not_cancel

//* Change the contract state *//

// set the contract state back to waiting (waiting for a withdrawal request)
byte "gstate"
byte "waiting"
app_global_put

b approve
```

# Calling the functions

## Creating the vault

To call the create function, a simple Application Create transaction can be sent to the blockchain, with the string "create" as a parameter.

```bash
goal app create --app-arg "str:vault" --app-arg "addr:{RECOVERY-ADDRESS}" --app-arg "int:{WAIT-TIME}" --creator "{CREATOR-ADDRESS}" --approval-prog "vault_approval.teal" --clear-prog "vault_clear.teal" --global-byteslices 4 --global-ints 3 --local-byteslices 0 --local-ints 0
```
## Initializing the escrow

To call the set_escrow function, the user must submit a group transaction with an application call (with set_escrow as an argument), and a payment transaction of 100'000 micro-algos to the escrow account.

```bash
goal clerk compile "vault_escrow.teal"
goal clerk send --from="{CREATOR-ADDRESS}" --to="7WAUA5CBMY7BX6NRCYAV2DRRC5ATCIZQRGCCTCKNUNSAWDIFXZVT6CQOLI" --amount=100000 --out=txn1.tx
goal app call --app-id {APP-ID} --app-arg "str:set_escrow" --from="{CREATOR-ADDRESS}"  --out=txn2.tx
cat txn1.tx txn2.tx > txn_combined.tx 
goal clerk group -i txn_combined.tx -o txn_grouped.tx
goal clerk sign -i txn_grouped.tx -o txn_signed.tx 
goal clerk rawsend -f txn_signed.tx
```

## Depositing funds

To deposit funds into the contract, the vault creator can simply send a pay transaction to the escrow account.

```bash
goal clerk send --from="{CREATOR-ADDRESS}" --to="7WAUA5CBMY7BX6NRCYAV2DRRC5ATCIZQRGCCTCKNUNSAWDIFXZVT6CQOLI" --amount={DEPOSIT-AMOUNT}
```

## Requesting a withdrawal

To call the withdraw function, an application call with the parameters "withdraw", an integer (the amount to be withdrawn), and a string (the receiving address), must be submitted.

```bash
goal app call --app-id {APP-ID} --app-arg "str:withdraw" --app-arg "int:{WITHDRAW-AMOUNT}" --app-arg "addr:{WITHDRAW-RECEIVER}" --from="{CREATOR-ADDRESS}"
```

## Finalizing a request

To call the finalize function, a pay transaction from the escrow account to the previously declared receiving account, of the previously declared amount must be sent, together with an application call with the string "finalize" as a parameter.

```bash
goal clerk send --from-program="vault_escrow.teal" --to="{WITHDRAW-RECEIVER}" --amount={WITHDRAW-AMOUNT} --fee 0 --out=txn1.tx 
goal app call --app-id {APP-ID} --app-arg "str:finalize" --from="{CREATOR-ADDRESS}" --fee 2000 --out=txn2.tx

cat txn1.tx txn2.tx > txn_combined.tx
goal clerk group -i txn_combined.tx -o txn_grouped.tx 
goal clerk split -i txn_grouped.tx -o txn_split.tx 

goal clerk sign -i txn_split-1.tx -o txn1_signed.tx
cat txn_split-0.tx txn1_signed.tx > txn_signed.tx
goal clerk rawsend -f txn_signed.tx
```

## Cancelling a request

To call the cancel function, a simple application call with the string "finalize" as a parameter can be sent (from the recovery address).

```bash
goal app call --app-id {APP-ID} --app-arg "str:cancel" --from="{RECOVERY-ADDRESS}"
``` 

# Full Code

## AlgoML
```java
glob int wait_time
glob address recovery
glob address vault

glob mut int request_time
glob mut int amount
glob mut address receiver

@gstate ->init_escrow
Create vault(address recovery, int wait_time) {
    glob.recovery = recovery
    glob.wait_time = wait_time
}

@gstate init_escrow->waiting
@from creator
@pay 100000 : * -> *$vault
set_escrow() {
    glob.vault = vault
}
    
@gstate waiting->requesting
@round *$curr_round
@from creator
withdraw(int amount, address receiver) {
    glob.amount = amount
    glob.receiver = receiver
    glob.request_time = curr_round
}

@gstate requesting->waiting
@round (glob.request_time + glob.wait_time,)
@from creator
@pay glob.amount : escrow -> glob.receiver
finalize() { }

@gstate requesting->waiting
@from glob.recovery
cancel() { }
```

## Teal 
### Stateful contract's approval program

```java
#pragma version 4

//**************************
//*   Creating the vault   *
//**************************

//* Check if we're calling the create function *//

// check if there are no other transactions in this atomic group
global GroupSize
int 1
==
bz not_create

// check if the application is being created with this transaction
txn ApplicationID
int 0
==
bz not_create

// check if the call has 3 arguments ("vault" + the two actual arguments: recovery account and wait_time)
txn NumAppArgs
int 3
==
bz not_create

txna ApplicationArgs 0
byte "vault"
==
bz not_create

//* Change the contract state *//

// set the contract state to init_escrow (waiting for escrow initialization)
byte "gstate"
byte "init_escrow"
app_global_put

// save the first argument (the address of the recovery account) into the global variable "recovery"
byte "recovery"
txna ApplicationArgs 1
app_global_put

// save the second argument (the time that must pass between withdrawal request and finalization) into the global variable "wait_time"
byte "wait_time"
txna ApplicationArgs 2
btoi
app_global_put

b approve

//*******************************
//*   Initializing the escrow   *
//*******************************

not_create:

//* Check if we're calling the set_escrow function *//

// check if there is one other transactions in this atomic group: the payment transaction needed to initialize the escrow account
global GroupSize
int 2
==
bz not_setescrow

// check if the application is currently waiting to initialize the escrow
byte "gstate"
app_global_get
byte "init_escrow"
==
bz not_setescrow

// check if this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_setescrow

// check if the other transaction is a pay transaction of 100'000 algos (the amount required to initialize an account)
gtxn 0 TypeEnum
int pay
==
bz not_setescrow

gtxn 0 Amount
int 100000
==
bz not_setescrow

// check if the other transaction is not a closing transaction
gtxn 0 CloseRemainderTo
global ZeroAddress
==
bz not_setescrow

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_setescrow

// check if the application call has 1 argument: the byte string "set_escrow"
txn NumAppArgs
int 1
== 
bz not_setescrow

txna ApplicationArgs 0
byte "set_escrow"
==
bz not_setescrow

//* Change the contract state *//

// set the contract state to waiting (waiting for a withdrawal request)
byte "gstate"
byte "waiting"
app_global_put

// save the vault address into the global state
byte "vault"
gtxn 0 Receiver
app_global_put

b approve

//*******************************
//*   Requesting a withdrawal   *
//*******************************

not_setescrow:

//* Check if we're calling the withdraw function *//

// check if there are no other transactions in this atomic group
global GroupSize
int 1
==
bz not_withdraw

// check if the contract is in the waiting state (waiting for a withdrawal request)
byte "gstate"
app_global_get
byte "waiting"
==
bz not_withdraw

// check if this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_withdraw

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_withdraw

// check if the call has 3 arguments ("withdraw" and the two actual arguments: amount and receiver)
txn NumAppArgs
int 3
==
bz not_withdraw

txna ApplicationArgs 0
byte "withdraw"
==
bz not_withdraw

//* Change the contract state *//

// set the contract state to requested (withdrawal request ongoing)
byte "gstate"
byte "requested"
app_global_put

// save the first argument (the amount that the user is requesting) into global amount
byte "amount"
txna ApplicationArgs 1
btoi
app_global_put

// save the second argument (the receiver of the withdrawal) into global receiver
byte "receiver"
txna ApplicationArgs 2
app_global_put

// save the current round into global request_time (so that the request can only be finalized wait_time rounds after)
byte "request_time"
global Round
app_global_put

b approve

//*******************************
//*   Finalizing a withdrawal   *
//*******************************

not_withdraw:

//* Check if we're calling the finalize function *//

// check if the application call transactions is bundled with a pay transaction (the withdrawal)
global GroupSize
int 2
==
bz not_finalize

// check if the contract is in the requested state (a withdrawal has been requested)
byte "gstate"
app_global_get
byte "requested"
==
bz not_finalize

// check if the withdrawal wait time has passed since the withdraw request
global Round
byte "request_time"
app_global_get
byte "wait_time"
app_global_get
+
>=
bz not_finalize

// check if this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_finalize

// check if the other transaction is a pay transaction from the escrow account to the requested receiver of the amount previously requested 
gtxn 0 TypeEnum
int pay
==
bz not_finalize

gtxn 0 Amount
byte "amount"
app_global_get
==
bz not_finalize

gtxn 0 Sender
byte "vault"
app_global_get
==
bz not_finalize

gtxn 0 Receiver
byte "receiver"
app_global_get
==
bz not_finalize

// check if the pay transaction is non-closing
gtxn 0 CloseRemainderTo
global ZeroAddress
==
bz not_finalize

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_finalize

// Check if the call has 1 argument ("finalize")
txn NumAppArgs
int 1
==
bz not_finalize

txna ApplicationArgs 0
byte "finalize"
==
bz not_finalize

//* Change the cntract state *//

// set the contract state back to waiting (waiting for a withdrawal request to be made)
byte "gstate"
byte "waiting"
app_global_put

b approve

//*******************************
//*   Cancelling a withdrawal   *
//*******************************

not_finalize:

//* Check if we're calling the cancel function *//

// check if there aren't other transactions in this atomic group
global GroupSize
int 1
==
bz not_cancel

// check if the contract is in state requested (withdrawal request ongoing)
byte "gstate"
app_global_get
byte "requested"
==
bz not_cancel

// check if this transaction is sent by the recovery account
txn Sender
byte "recovery"
app_global_get
==
bz not_cancel

// check if the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_cancel

// check if the application call has 1 argument ("cancel")
txn NumAppArgs
int 1
==
bz not_cancel

txna ApplicationArgs 0
byte "cancel"
==
bz not_cancel

//* Change the contract state *//

// set the contract state back to waiting (waiting for a withdrawal request)
byte "gstate"
byte "waiting"
app_global_put

b approve

//****************************************
//*   Function end / No function found   *
//****************************************

not_cancel:
err

approve:
int 1
```

### Stateful contract's clear program
```java
#pragma version 4

// reject any transaction
err
```

### Stateless contract
```java
#pragma version 3

// aseert that the stateful contracts participates in the transaction group
gtxn 1 TypeEnum
int appl
==
assert

gtxn 1 ApplicationID
int <APP-ID>
==
assert

// assert that this transaction is "non-rekeying"
txn RekeyTo
global ZeroAddress
==
assert

// assert that no fee is paid by this contract (all fees must be paid by the caller)
txn Fee
int 0
==
assert

// approve
int 1
```

