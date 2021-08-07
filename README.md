# Overview

Vaults are contracts that force users to send two separate transactions in order to spend money from their wallet: a first transaction to "request" a withdrawal, and a second one, issued after a certain amount of time, to finalize it. 
If a transaction were to be issued by a malevolent user M who stole user A's key, A would have time to notice the issued payment and cancel it, making M's effort to steal the user's key futile.

We will approach this issue with Teal together with AlgoML, a language that will provide an effective solution to understanding the functioning of vaults.

# Design Overview

To create a contract that implements vaults, six different steps must be supported:

1. [__Creating the vault.__](#creating-the-vault) Any user should be able to create a vault, providing the recovery address and the amount of rounds that must pass between payment request and finalization

2. [__Initialize the escrow__](#escrow-initialization) The vault owner must provide the address of the stateless account that keeps the funds. 

3. [__Depositing funds.__](#depositing-funds) The vault owner should be able to deposit funds into the vault.

4. [__Requesting a withdrawal.__](#requesting-a-withdrawal) The vault owner (or anyone who has access to the owner's key) must be able to request a withdrawal of a certain amount of algos from the vault.

5. [__Finalizing a request.__](#finalizing-a-withdrawal-request) The vault owner must be able to finalize a withdrawal after a certain amount of time has passed since the request.

6. [__Cancelling a request.__](#cancelling-a-withdrawal-request) The vault owner, using his recovery account, must be able to cancel any withdrawal request during the time window between request and finalization.

## The contract state

The contract must maintain the state of seven different variables.

* `wait_time` indicates how many rounds must pass between the withdrawal request and the withdrawal finalization
* `recovery` is the address from which the cancel call must originate
* `vault` is the escrow address, where the deposited funds will be kept

* `request_time` is the round at which the withdrawal request has been submitted
* `amount` is the amount of algos that were requested
* `receiver` is the receiving address of the requested withdrawal
* `gstate` is the contract state (waiting for escrow initialization, no withdrawal request in progress, withdrawal request in progress)

The first 3 values are constant throughout the life of the contract, while the last 4 change with each withdrawal request.

## The escrow account

The vault's escrow account is a stateless contract that approves every transaction as long as:
1. the stateful contract partecipates in the approval decision (a call to the stateful contract is in the group)
2. the stateless contract doesn't have to pay any fee
3. the stateless contract is not sending a rekey transaction

### Teal

```java
gtxn 1 TypeEnum
int appl
assert

gtxn 1 ApplicationID
int <APP-ID>
assert

txn RekeyTo
global ZeroAddress
assert

txn Fee
int 0
assert
```

## Creating the vault

To initialize the contract, the user must provide a recovery address and the amount of rounds that must pass between withdrawal request and withdrawal finalization. Once the contract gets created, the user will need to provide the escrow account address (it won't be possible to provide it on creation, since the escrow account needs the application ID).

### AlgoML
AlgoML splits the TEAL code into functions, which can be executed if certain preconditions are met.

```java
@gstate ->init_escrow
Create create(address recovery, int wait_time) {
    glob.recovery = recovery
    glob.wait_time = wait_time
}
```
To call the *create* function the user must pass two parameters: the recovery address, and the time that must pass between a withdrawal request and its finalization. When this function is called, the contract gets created and its two state variables `recovery` and `wait_time` get initializated, while the contract is put into the `init_escrow` state (waiting for the escrow initialization).

The following Teal code corresponds to the previous AlgoML function.

### Teal
```java
/* check that we're calling the create function */

// Check that there are no other transactions in this atomic groupp
global GroupSize
int 1
==
bz not_create

// check that the application is being created with this transaction
txn ApplicationID
int 0
==
bz not_create

// Check that the call has 3 arguments ("create" + the two actual arguments)
txn NumAppArgs
int 3
==
bz not_create

// Check that the first argument is the byte string "create"
txna ApplicationArgs 0
byte "create"
==
bz not_create

/* change state */

// Change the current state into init_escrow (waiting for escrow initialization)
byte "gstate"
byte "init_escrow"
app_global_put

/* change statevars */

// Save the first argument (the address of the recovery account) into the global variable "recovery"
byte "recovery"
txna ApplicationArgs 1
app_global_put

// Save the second argument (the time that must pass between withdrawal request and finalization) into the global variable "wait_time"
byte "wait_time"
txna ApplicationArgs 2
btoi
app_global_put

b approve
```

### Goal

To call the create function, a simple Application Create transaction can be sent to the blockchain, with the string "create" as a parameter.

```bash
$ goal app create --app-args "string:create" --creator {ACCOUNT} --approval-prog "approvalprog.teal" --clear-prog "clearprog.teal" --global-byteslices 4 --global-ints 3 --from={ACCOUNT}
```

## Escrow initialization

After the contract gets created, the user must connect the escrow account to the stateful contract, in order to have a place to store the deposited funds.

### AlgoML
```java
@gstate init_escrow->waiting
@from creator
@pay 100000 : * -> *$vault
NoOp init_escrow() {
    glob.vault = vault
}
```

To call the *init_escrow* function, the contract must be in the `init_escrow` state (the state that immediately follows the contract creation), and must be called from the contract creator. The application call must also come bundled with a pay transaction, with an amount equal to 100'000 algos (the amount needed to initialize an account). When called, the vault address is saved into the global state, and the contract state gets set to `waiting` (waiting for a withdrawal request).

### Teal
```java
/* check that we're calling the init_escrow function */

// Check that there is one other transactions in this atomic group, the payment transaction needed to initialize the escrow account
global GroupSize
int 2
==
bz not_connect

// check that the application is currently waiting to initialize the escrow
byte "gstate"
app_global_get
byte "init_escrow"
==
bz not_connect

// Check that this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_connect

// Check that the other transaction is a pay transaction of 100'000 algos (the amount required to initialize an account)
gtxn 0 TypeEnum
int pay
==
bz not_connect

gtxn 0 Amount
int 100000
==
bz not_connect

// Check that the other transaction is not a closing transaction
gtxn 0 CloseRemainderTo
global ZeroAddress
==
bz aclause_5

// check that the application call is NoOp
txn OnCompletion
int NoOp
==
bz not_connect

// Check that the call has 1 argument: the byte string "init_escrow"
txn NumAppArgs
int 1
== 
bz not_connect

txna ApplicationArgs 0
byte "init_escrow"
==
bz not_connect

/* change state */

// Change the contract state into waiting (waiting for a withdrawal request)
byte "gstate"
byte "waiting"
app_global_put

/* change statevars */

// Save the vault address into the global state
byte "vault"
gtxn 0 Receiver
app_global_put

b approve
```

### Goal

To call the init_escrow function, the user must submit a group transaction with an application call (with init_escrow as an argument), and a payment transaction of 100'000 algos to the escrow account.
```bash
$ goal clerk send --from={ACCOUNT} --to={ESCROWACCOUNT} --amount=100000 --out=txn1.tx -d ~/node/data
$ goal app call --app-id {APPID}  --app-arg "str:init_escrow" --from={ACCOUNT}  --out=txn2.tx -d ~/node/data
$ cat txn1.tx txn2.tx > txn_combined.tx 
$ goal clerk group -i txn_combined.tx -o txn_grouped.tx
$ goal clerk sign -i txn_grouped.tx -o txn_signed.tx 
$ goal clerk rawsend -f txn_signed.tx
```

## Depositing funds


### Goal
To deposit funds into the contract, the vault owner can simply send a pay transaction to the escrow account.

```bash
$ goal clerk send --from={ACCOUNT} --to={ESCROWACCOUNT} --amount=9876 --out=txn.tx -d ~/node/data
$ goal clerk sign -i txn.tx -o txn_signed.tx 
$ goal clerk rawsend -f txn_signed.tx
```

## Requesting a withdrawal

When the vault owner (or anyone that has access to his private key) wants to request a withdrawal, they must declare the amount that they want to withdraw, and the address of the account that will receive the funds. The contract will need to store the declared amount and receiver, while also storing the round at which the withdrawal was requested.

### AlgoML
```java
@gstate waiting->requested
@round *$curr_round
@from creator
NoOp withdraw(int amount, address receiver) {
    glob.amount = amount
    glob.receiver = receiver
    glob.request_time = curr_round
}
```
The *withdraw* function can only be called by the creator, and only while the contract is in state `waiting` (waiting for a withdrawal request) 

To call it, the creator must pass the amount that he wants to withdraw, and the address where the withdrawn funds will be sent. The contract will then save those parameters, as well as the current round, into its global state, while also putting the contract into the `requested` state (A withdrawal request has been made). 

### Teal
```java
/* check that we're calling the withdraw function */

// Check that there are no other transactions in this atomic group
global GroupSize
int 1
==
bz not_withdraw

// check that the contract is in the waiting state (waiting for a withdrawal request)
byte "gstate"
app_global_get
byte "waiting"
==
bz not_connect

// Check that this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_withdraw

// check that the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_withdraw

// Check that the call has 3 arguments ("withdraw" and the two actual arguments)
txn NumAppArgs
int 3
==
bz not_withdraw

txna ApplicationArgs 0
byte "withdraw"
==
bz not_withdraw

/* change state */

// Set the global state to requested (withdrawal request ongoing)
byte "gstate"
byte "requested"
app_global_put

/* change statevars */

// Save the first argument (the amount that the user is requesting) into global amount
byte "amount"
txna ApplicationArgs 1
btoi
app_global_put

// Save the second argument (the receiver of the withdrawal) into global receiver
byte "receiver"
txna ApplicationArgs 2
app_global_put

// Save current round into global request_time (so that the request can only be finalized n rounds afterwards)
byte "request_time"
global Round
app_global_put

b approve
```

### Goal

To call the withdraw function, an application call with the parameters "withdraw", an integer (the amount to be withdrawn), and a string (the receiving address), must be submitted.

```bash
$ goal app call --app-arg "string:withdraw" "int:{AMOUNT}" "string:{RECEIVER}" --from={ACCOUNT} --out=txn1.tx -d ~/node/data
$ goal clerk sign -i txn1.tx txn_signed.tx
$ goal clerk rawsend -i txn_signed.tx
```

## Finalizing a withdrawal request

After `wait_time` rounds have passed since a withdrawal request, the vault owner can ask for the request to be finalized, thus receiving the funds and putting the contract back into a state where other requests are accepted.

### AlgoML
```java
@gstate requested->waiting
@round (glob.request_time + glob.wait_time,)
@from creator
@pay glob.amount : vault -> glob.receiver
NoOp finalize() { }
```

The *finalize* function can only be called by the contract creator while in the `requested` state, and only after `wait_time` rounds have passed since the `requested_time`. To call this function, a pay transaction must be bundled with the application call. This pay transaction must have the vault address as a sender, and the amount and receiver saved in the glbal state (the ones that were declared with the withdraw call). 

After this function call, the contract will be put back into `waiting` state.

### Teal
```java
/* check that we're calling the finalize function */

// Check that the application call transactions goes with a pay transaction (the withdrawal)
global GroupSize
int 2
==
bz not_finalize

// check that the contract is in the withdrawal state (a withdrawal has been requested)
byte "gstate"
app_global_get
byte "requesting"
==
bz not_finalize

// Check that at least wait_time rounds have passed since the withdraw request
global Round
byte "request_time"
app_global_get
byte "wait_time"
app_global_get
+
>=
bz not_finalize

// Check that this transaction is sent by the contract creator
txn Sender
global CreatorAddress
==
bz not_finalize

// Check that the other transaction is a pay transaction of the amount requested from the escrow account to the requested receiver 
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

// Check that the pay transaction is non-closing
gtxn 0 CloseRemainderTo
global ZeroAddress
==
bz not_finalize

// check that the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_finalize

// Check that the call has 1 argument ("finalize")
txn NumAppArgs
int 1
==
bz not_finalize

txna ApplicationArgs 0
byte "finalize"
==
bz not_finalize

/* change state */

// Set the contract state back to waiting (waiting for a withdrawal request to be made)
byte "gstate"
byte "waiting"
app_global_put

b approve
```

### Goal

To call the finalize function, a pay transaction from the escrow account to the previously declared receiving account, of the previously declared amount must be sent, together with an application call with the string "finalize" as a parameter.

```bash
$ goal clerk send --from={ACCOUNT} --to={ESCROWACCOUNT} --amount={AMOUNT} --out=txn1.tx -d ~/node/data
$ goal app call --app-id {APPID}  --app-arg "str:finalize" --from={ACCOUNT}  --out=txn2.tx -d ~/node/data
$ cat txn1.tx txn2.tx > txn_combined.tx 
$ goal clerk group -i txn_combined.tx -o txn_grouped.tx
$ goal clerk sign -i txn_grouped.tx -o txn_signed.tx 
$ goal clerk rawsend -f txn_signed.tx

#! TODO: non va firmata la paytxn dall'escrow
```

## Cancelling a withdrawal request

If the owner of the contract notices a withdrawal request that he has not made (and therefore that someone has access to his private key), they can cancel the transaction using their recovery account, and then proceed to withdraw all the funds into a secure account.

Notice that, since to cancel a withdrawal request the recovery account private key is needed, anyone who steals the vault creator private key (but not the recovery key) won't be able to cancel any withdrawal request.

### AlgoML
```java
@gstate requesting->waiting
@from glob.recovery
NoOp cancel() { }
```

The cancel function can only be called **from the recovery address**, and only after a withdrawal request has been sent (while the contract is in state `requesting`). Once called, the contract will go back to the `waiting` state, thus disabling the finalize function, and requiring another withdraw call for any further withdrawal request.

### Teal
```java
/* check that we'ree calling the cancel function */

// Check that there aren't other transactions in this atomic group
global GroupSize
int 1
==
bz not_withdraw

// Check that the contract is in state requesting (withdrawal request ongoing)
byte "gstate"
app_global_get
byte "requesting"
==
bz not_connect

// Check that this transaction is sent by the recovery account
txn Sender
byte "recovery"
app_global_get
==
bz not_withdraw

// check that the application call has a NoOp oncompletion
txn OnCompletion
int NoOp
==
bz not_finalize

// Check that the call has 1 argument ("cancel")
txn NumAppArgs
int 1
==
bz not_finalize

txna ApplicationArgs 0
byte "cancel"
==
bz not_withdraw

/* change state */

// Set the contract state back to waiting (waiting for a withdrawal request)
byte "gstate"
byte "waiting"
app_global_put

b approve
```

### Goal

To call the cancel function, a simple application call with the string "finalize" as a parameter can be sent (from the recovery address).
```
$ goal app call --app-id {APPID}  --app-arg "str:finalize" --from={RECOVERYACCOUNT}  --out=txn1.tx -d ~/node/data
$ goal clerk sign -i txn1.tx -o txn_signed.tx 
$ goal clerk rawsend -f txn_signed.tx
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
Create create(address recovery, int wait_time) {
    glob.recovery = recovery
    glob.wait_time = wait_time
}

@gstate init_escrow->waiting
@from creator
@pay 100000 : * -> *$vault
NoOp init_escrow() {
    glob.vault = vault
}
    
@gstate waiting->requesting
@round *$curr_round
@from creator
NoOp withdraw(int amount, address receiver) {
    glob.amount = amount
    glob.receiver = receiver
    glob.request_time = curr_round
}

@gstate requesting->waiting
@round (glob.request_time + glob.wait_time,)
@from creator
@pay glob.amount : escrow -> glob.receiver
NoOp finalize() { }

@gstate requesting->waiting
@from glob.recovery
NoOp cancel() { }
```

## Teal
```
...
full teal code
```
