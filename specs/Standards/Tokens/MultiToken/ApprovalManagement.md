# Multi Token Standard Approval Management

:::caution
This is part of the proposed spec [NEP-245](https://github.com/near/NEPs/blob/master/neps/nep-0245.md) and is subject to change.
:::


Version `1.0.0`

## Summary

A system for allowing a set of users or contracts to transfer specific tokens on behalf of an owner. Similar to approval management systems in standards like [ERC-721] and [ERC-1155].

  [ERC-721]: https://eips.ethereum.org/EIPS/eip-721
  [ERC-1155]: https://eips.ethereum.org/EIPS/eip-1155

## Motivation

People familiar with [ERC-721] may expect to need an approval management system for basic transfers, where a simple transfer from Alice to Bob requires that Alice first _approve_ Bob to spend one of her tokens, after which Bob can call `transfer_from` to actually transfer the token to himself.

NEAR's [core Multi Token standard](README.md) includes good support for safe atomic transfers without such complexity. It even provides "transfer and call" functionality (`mt_transfer_call`) which allows  specific tokens to be "attached" to a call to a separate contract. For many token workflows, these options may circumvent the need for a full-blown Approval Management system.

However, some Multi Token developers, marketplaces, dApps, or artists may require greater control. This standard provides a uniform interface allowing token owners to approve other NEAR accounts, whether individuals or contracts, to transfer specific tokens on the owner's behalf.

Prior art:

- Ethereum's [ERC-721]
- Ethereum's [ERC-1155]

## Example Scenarios

Let's consider some examples. Our cast of characters & apps:

- Alice: has account `alice` with no contract deployed to it
- Bob: has account `bob` with no contract deployed to it
- MT: a contract with account `mt`, implementing only the [Multi Token Standard](Core.md) with this Approval Management extension
- Market: a contract with account `market` which sells tokens from `mt` as well as other token contracts
- Bazaar: similar to Market, but implemented differently (spoiler alert: has no `mt_on_approve` function!), has account `bazaar`

Alice and Bob are already [registered](../../StorageManagement.md) with MT, Market, and Bazaar, and Alice owns a token on the MT contract with ID=`"1"` and a fungible style token with ID =`"2"` and AMOUNT =`"100"`.

Let's examine the technical calls through the following scenarios:

1. [Simple approval](#1-simple-approval): Alice approves Bob to transfer her token.
2. [Approval with cross-contract call (XCC)](#2-approval-with-cross-contract-call): Alice approves Market to transfer one of her tokens and passes `msg` so that MT will call `mt_on_approve` on Market's contract.
3. [Approval with XCC, edge case](#3-approval-with-cross-contract-call-edge-case): Alice approves Bazaar and passes `msg` again, but what's this? Bazaar doesn't implement `mt_on_approve`, so Alice sees an error in the transaction result. Not to worry, though, she checks `mt_is_approved` and sees that she did successfully approve Bazaar, despite the error.
4. [Approval IDs](#4-approval-ids): Bob buys Alice's token via Market.
5. [Approval IDs, edge case](#5-approval-ids-edge-case): Bob transfers same token back to Alice, Alice re-approves Market & Bazaar. Bazaar has an outdated cache. Bob tries to buy from Bazaar at the old price.
6. [Revoke one](#6-revoke-one): Alice revokes Market's approval for this token.
7. [Revoke all](#7-revoke-all): Alice revokes all approval for this token.

### 1. Simple Approval

Alice approves Bob to transfer her tokens.

#### High-level explanation

1. Alice approves Bob
2. Alice queries the token to verify

#### Technical calls

1. Alice calls `mt::mt_approve({ "token_ids": ["1","2"], amounts:["1","100"], "account_id": "bob" })`. She attaches 1 yoctoⓃ, (.000000000000000000000001Ⓝ). Using [NEAR CLI](https://docs.near.org/tools/near-cli) to make this call, the command would be:

    ```bash
    near call mt mt_approve \
        '\{ "token_ids": ["1","2"], amounts: ["1","100"], "account_id": "bob" }' \
        --accountId alice --amount .000000000000000000000001
    ```

    The response:

    ```bash
    ''
    ```

2. Alice calls view method `mt_is_approved`:

    ```bash
    near view mt mt_is_approved \
        '\{ "token_ids": ["1", "2"], amounts:["1","100"], "approved_account_id": "bob" }'
    ```

    The response:

    ```bash
    true
    ```

### 2. Approval with cross-contract call

Alice approves Market to transfer some of her tokens and passes `msg` so that MT will call `mt_on_approve` on Market's contract. She probably does this via Market's frontend app which would know how to construct `msg` in a useful way.

#### High-level explanation

1. Alice calls `mt_approve` to approve `market` to transfer her token, and passes a `msg`
2. Since `msg` is included, `mt` will schedule a cross-contract call to `market`
3. Market can do whatever it wants with this info, such as listing the token for sale at a given price. The result of this operation is returned as the promise outcome to the original `mt_approve` call.

#### Technical calls

1. Using near-cli:

    ```bash
    near call mt mt_approve '\{
        "token_ids": ["1","2"],
        "amounts": ["1", "100"],
        "account_id": "market",
        "msg": "\{\"action\": \"list\", \"price\": [\"100\",\"50\"],\"token\": \"nDAI\" }"
    }' --accountId alice --amount .000000000000000000000001
    ```

    At this point, near-cli will hang until the cross-contract call chain fully resolves, which would also be true if Alice used a Market frontend using [near-api](https://docs.near.org/tools/near-api). Alice's part is done, though. The rest happens behind the scenes.

2. `mt` schedules a call to `mt_on_approve` on `market`. Using near-cli notation for easy cross-reference with the above, this would look like:

    ```bash
    near call market mt_on_approve '\{
        "token_ids": ["1","2"],
        "amounts": ["1","100"],
        "owner_id": "alice",
        "approval_ids": ["4","5"],
        "msg": "\{\"action\": \"list\", \"price\": [\"100\",\"50\"], \"token\": \"nDAI\" }"
    }' --accountId mt
    ```

3. `market` now knows that it can sell Alice's tokens for 100 [nDAI](https://explorer.mainnet.near.org/accounts/6b175474e89094c44da98b954eedeac495271d0f.factory.bridge.near) and 50 [nDAI](https://explorer.mainnet.near.org/accounts/6b175474e89094c44da98b954eedeac495271d0f.factory.bridge.near), and that when it transfers it to a buyer using `mt_batch_transfer`, it can pass along the given `approval_ids` to ensure that Alice hasn't changed her mind. It can schedule any further cross-contract calls it wants, and if it returns these promises correctly, Alice's initial near-cli call will resolve with the outcome from the final step in the chain. If Alice actually made this call from a Market frontend, the frontend can use this return value for something useful.

### 3. Approval with cross-contract call, edge case

Alice approves Bazaar and passes `msg` again. Maybe she actually does this via near-cli, rather than using Bazaar's frontend, because what's this? Bazaar doesn't implement `mt_on_approve`, so Alice sees an error in the transaction result.

Not to worry, though, she checks `mt_is_approved` and sees that she did successfully approve Bazaar, despite the error. She will have to find a new way to list her token for sale in Bazaar, rather than using the same `msg` shortcut that worked for Market.

#### High-level explanation

1. Alice calls `mt_approve` to approve `bazaar` to transfer her token, and passes a `msg`.
2. Since `msg` is included, `mt` will schedule a cross-contract call to `bazaar`.
3. Bazaar doesn't implement `mt_on_approve`, so this call results in an error. The approval still worked, but Alice sees an error in her near-cli output.
4. Alice checks if `bazaar` is approved, and sees that it is, despite the error.

#### Technical calls

1. Using near-cli:

    ```bash
    near call mt mt_approve '\{
        "token_ids": ["1"],
        "amounts: ["1000"],
        "account_id": "bazaar",
        "msg": "\{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
    }' --accountId alice --amount .000000000000000000000001
    ```

2. `mt` schedules a call to `mt_on_approve` on `market`. Using near-cli notation for easy cross-reference with the above, this would look like:

    ```bash
    near call bazaar mt_on_approve '\{
        "token_ids": ["1"],
        "amounts": ["1000"],
        "owner_id": "alice",
        "approval_ids": [3],
        "msg": "\{\"action\": \"list\", \"price\": \"100\", \"token\": \"nDAI\" }"
    }' --accountId mt
    ```

3. 💥 `bazaar` doesn't implement this method, so the call results in an error. Alice sees this error in the output from near-cli.

4. Alice checks if the approval itself worked, despite the error on the cross-contract call:

    ```bash
    near view mt mt_is_approved \
        '{ "token_ids": ["1","2"], "amounts":["1","100"], "approved_account_id": "bazaar" }'
    ```

    The response:

    ```bash
    true
    ```

### 4. Approval IDs

Bob buys Alice's token via Market. Bob probably does this via Market's frontend, which will probably initiate the transfer via a call to `ft_transfer_call` on the nDAI contract to transfer 100 nDAI to `market`. Like the MT standard's "transfer and call" function, [Fungible Token](../FungibleToken/Core.md)'s `ft_transfer_call` takes a `msg` which `market` can use to pass along information it will need to pay Alice and actually transfer the MT. The actual transfer of the MT is the only part we care about here.

#### High-level explanation

1. Bob signs some transaction which results in the `market` contract calling `mt_transfer` on the `mt` contract, as described above. To be trustworthy and pass security audits, `market` needs to pass along `approval_id` so that it knows it has up-to-date information.

#### Technical calls

Using near-cli notation for consistency:

    ```bash
    near call mt mt_transfer '{
        "receiver_id": "bob",
        "token_id": "1",
        "amount": "1",
        "approval_id": 2,
    }' --accountId market --amount .000000000000000000000001
    ```

### 5. Approval IDs, edge case

Bob transfers same token back to Alice, Alice re-approves Market & Bazaar, listing her token at a higher price than before. Bazaar is somehow unaware of these changes, and still stores `approval_id: 3` internally along with Alice's old price. Bob tries to buy from Bazaar at the old price. Like the previous example, this probably starts with a call to a different contract, which eventually results in a call to `mt_transfer` on `bazaar`. Let's consider a possible scenario from that point.

#### High-level explanation

Bob signs some transaction which results in the `bazaar` contract calling `mt_transfer` on the `mt` contract, as described above. To be trustworthy and pass security audits, `bazaar` needs to pass along `approval_id` so that it knows it has up-to-date information. It does not have up-to-date information, so the call fails. If the initial `mt_transfer` call is part of a call chain originating from a call to `ft_transfer_call` on a fungible token, Bob's payment will be refunded and no assets will change hands.

#### Technical calls

Using near-cli notation for consistency:

```bash
near call mt mt_transfer '{
    "receiver_id": "bob",
    "token_id": "1",
    "amount": "1",
    "approval_id": 3,
}' --accountId bazaar --amount .000000000000000000000001
```

### 6. Revoke one

Alice revokes Market's approval for this token.

#### Technical calls

Using near-cli:

```bash
near call mt mt_revoke '\{
    "account_id": "market",
    "token_ids": ["1"],
}' --accountId alice --amount .000000000000000000000001
```

Note that `market` will not get a cross-contract call in this case. The implementors of the Market app should implement [cron](https://en.wikipedia.org/wiki/Cron)-type functionality to intermittently check that Market still has the access they expect.

### 7. Revoke all

Alice revokes all approval for these tokens

#### Technical calls

Using near-cli:

```bash
near call mt mt_revoke_all '\{
  "token_ids": ["1", "2"],
}' --accountId alice --amount .000000000000000000000001
```

Again, note that no previous approvers will get cross-contract calls in this case.


## Reference-level explanation

The `TokenApproval` structure returned by `mt_token_approvals` returns `approved_account_ids` field, which is a map of account IDs to `Approval` and `approval_owner_id` which is the associated account approved for removal from. The `amount` field though wrapped in quotes and treated like strings, the number will be stored as an unsigned integer with 128 bits.
 in approval is  Using TypeScript's [Record type](https://www.typescriptlang.org/docs/handbook/utility-types.html#recordkeystype) notation:

```diff
+ type Approval = {
+   amount: string
+   approval_id: string
+ }
+
+ type TokenApproval = {
+  approval_owner_id: string,
+  approved_account_ids: Record<string, Approval>,
+ };
```

Example token approval data:

```json
[{
  "approval_owner_id": "alice.near",
  "approved_account_ids": {
    "bob.near": {
      "amount": "100",
      "approval_id":1,
    },
    "carol.near": {
      "amount":"2",
      "approval_id": 2,
    }
  }
}]
```

### What is an "approval ID"?

This is a unique number given to each approval that allows well-intentioned marketplaces or other 3rd-party MT resellers to avoid a race condition. The race condition occurs when:

1. A token is listed in two marketplaces, which are both saved to the token as approved accounts.
2. One marketplace sells the token, which clears the approved accounts.
3. The new owner sells back to the original owner.
4. The original owner approves the token for the second marketplace again to list at a new price. But for some reason the second marketplace still lists the token at the previous price and is unaware of the transfers happening.
5. The second marketplace, operating from old information, attempts to again sell the token at the old price.

Note that while this describes an honest mistake, the possibility of such a bug can also be taken advantage of by malicious parties via [front-running](https://users.encs.concordia.ca/~clark/papers/2019_wtsc_front.pdf).

To avoid this possibility, the MT contract generates a unique approval ID each time it approves an account. Then when calling `mt_transfer`, `mt_transfer_call`, `mt_batch_transfer`, or `mt_batch_transfer_call` the approved account passes `approval_id` or `approval_ids` with this value to make sure the underlying state of the token(s) hasn't changed from what the approved account expects.

Keeping with the example above, say the initial approval of the second marketplace generated the following `approved_account_ids` data:

```json
{
  "approval_owner_id": "alice.near",
  "approved_account_ids": {
    "marketplace_1.near": {
      "approval_id": 1,
      "amount": "100",
    },
    "marketplace_2.near": 2,
    "approval_id": 2,
     "amount": "50",
  }
}
```

But after the transfers and re-approval described above, the token might have `approved_account_ids` as:

```json
{
  "approval_owner_id": "alice.near",
  "approved_account_ids": {
    "marketplace_2.near": {
      "approval_id": 3,
      "amount": "50",
    }
  }
}
```

The marketplace then tries to call `mt_transfer`, passing outdated information:

```bash
# oops!
near call mt-contract.near mt_transfer '{"account_id": "someacct", "amount":"50", "approval_id": 2 }'
```


### Interface

The MT contract must implement the following methods:

```ts
/******************/
/* CHANGE METHODS */
/******************/

// Add an approved account for a specific set of tokens.
//
// Requirements
// * Caller of the method must attach a deposit of at least 1 yoctoⓃ for
//   security purposes
// * Contract MAY require caller to attach larger deposit, to cover cost of
//   storing approver data
// * Contract MUST panic if called by someone other than token owner
// * Contract MUST panic if addition would cause `mt_revoke_all` to exceed
//   single-block gas limit. See below for more info.
// * Contract MUST increment approval ID even if re-approving an account
// * If successfully approved or if had already been approved, and if `msg` is
//   present, contract MUST call `mt_on_approve` on `account_id`. See
//   `mt_on_approve` description below for details.
//
// Arguments:
// * `token_ids`: the token ids for which to add an approval
// * `account_id`: the account to add to `approved_account_ids`
// * `amounts`: the number of tokens to approve for transfer, wrapped in quotes and treated
//    like an array of string, although the numbers will be stored as an array of
//    unsigned integer with 128 bits.

// * `msg`: optional string to be passed to `mt_on_approve`
//
// Returns void, if no `msg` given. Otherwise, returns promise call to
// `mt_on_approve`, which can resolve with whatever it wants.
function mt_approve(
  token_ids: [string],
  amounts: [string],
  account_id: string,
  msg: string|null,
): void|Promise<any> {}

// Revoke an approved account for a specific token.
//
// Requirements
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security
//   purposes
// * If contract requires >1yN deposit on `mt_approve`, contract
//   MUST refund associated storage deposit when owner revokes approval
// * Contract MUST panic if called by someone other than token owner
//
// Arguments:
// * `token_ids`: the token for which to revoke approved_account_ids
// * `account_id`: the account to remove from `approvals`
function mt_revoke(
  token_ids: [string],
  account_id: string
) {}

// Revoke all approved accounts for a specific token.
//
// Requirements
// * Caller of the method must attach a deposit of 1 yoctoⓃ for security
//   purposes
// * If contract requires >1yN deposit on `mt_approve`, contract
//   MUST refund all associated storage deposit when owner revokes approved_account_ids
// * Contract MUST panic if called by someone other than token owner
//
// Arguments:
// * `token_ids`: the token ids with approved_account_ids to revoke
function mt_revoke_all(token_ids: [string]) {}

/****************/
/* VIEW METHODS */
/****************/

// Check if tokens are approved for transfer by a given account, optionally
// checking an approval_id
//
// Requirements:
// * Contract MUST panic if `approval_ids` is not null and the length of
//   `approval_ids` is not equal to `token_ids`
//
// Arguments:
// * `token_ids`: the tokens for which to check an approval
// * `approved_account_id`: the account to check the existence of in `approved_account_ids`
// * `amounts`: specify the positionally corresponding amount for the `token_id`
//    that at least must be approved. The number of tokens to approve for transfer,
//    wrapped in quotes and treated like an array of string, although the numbers will be
//    stored as an array of unsigned integer with 128 bits.
// * `approval_ids`: an optional array of approval IDs to check against
//    current approval IDs for given account and `token_ids`.
//
// Returns:
// if `approval_ids` is given, `true` if `approved_account_id` is approved with given `approval_id`
// and has at least the amount specified approved  otherwise, `true` if `approved_account_id`
// is in list of approved accounts and has at least the amount specified approved
// finally it returns false for all other states
function mt_is_approved(
  token_ids: [string],
  approved_account_id: string,
  amounts: [string],
  approval_ids: number[]|null
): boolean {}

// Get a the list of approvals for a given token_id and account_id
//
// Arguments:
// * `token_id`: the token for which to check an approval
// * `account_id`: the account to retrieve approvals for
//
// Returns a TokenApproval object, as described in Approval Management standard
function mt_token_approval(
  token_id: string,
  account_id: string,
): TokenApproval {}


// Get a list of all approvals for a given token_id
//
// Arguments:
// * `from_index`: a string representing an unsigned 128-bit integer,
//    representing the starting index of tokens to return
// * `limit`: the maximum number of tokens to return
//
// Returns an array of TokenApproval objects, as described in Approval Management standard, and an empty array if there are no approvals
function mt_token_approvals(
  token_id: string,
  from_index: string|null, // default: "0"
  limit: number|null,
): TokenApproval[] {}
```

### Why must `mt_approve` panic if `mt_revoke_all` would fail later?

In the description of `mt_approve` above, it states:

    Contract MUST panic if addition would cause `mt_revoke_all` to exceed
    single-block gas limit.

What does this mean?

First, it's useful to understand what we mean by "single-block gas limit". This refers to the [hard cap on gas per block at the protocol layer](https://docs.near.org/docs/concepts/gas#thinking-in-gas). This number will increase over time.

Removing data from a contract uses gas, so if an MT had a large enough number of approvals, `mt_revoke_all` would fail, because calling it would exceed the maximum gas.

Contracts must prevent this by capping the number of approvals for a given token. However, it is up to contract authors to determine a sensible cap for their contract (and the single block gas limit at the time they deploy). Since contract implementations can vary, some implementations will be able to support a larger number of approvals than others, even with the same maximum gas per block.

Contract authors may choose to set a cap of something small and safe like 10 approvals, or they could dynamically calculate whether a new approval would break future calls to `mt_revoke_all`. But every contract MUST ensure that they never break the functionality of `mt_revoke_all`.


### Approved Account Contract Interface

If a contract that gets approved to transfer MTs wants to, it can implement `mt_on_approve` to update its own state when granted approval for a token:

```ts
// Respond to notification that contract has been granted approval for a token.
//
// Notes
// * Contract knows the token contract ID from `predecessor_account_id`
//
// Arguments:
// * `token_ids`: the token_ids to which this contract has been granted approval
// * `amounts`: the ositionally corresponding amount for the token_id
//    that at must be approved. The number of tokens to approve for transfer,
//    wrapped in quotes and treated like an array of string, although the numbers will be
//    stored as an array of unsigned integer with 128 bits.
// * `owner_id`: the owner of the token
// * `approval_ids`: the approval ID stored by NFT contract for this approval.
//    Expected to be a number within the 2^53 limit representable by JSON.
// * `msg`: specifies information needed by the approved contract in order to
//    handle the approval. Can indicate both a function to call and the
//    parameters to pass to that function.
function mt_on_approve(
  token_ids: [TokenId],
  amounts: [string],
  owner_id: string,
  approval_ids: [number],
  msg: string,
) {}
```

Note that the MT contract will fire-and-forget this call, ignoring any return values or errors generated. This means that even if the approved account does not have a contract or does not implement `mt_on_approve`, the approval will still work correctly from the point of view of the MT contract.

Further note that there is no parallel `mt_on_revoke` when revoking either a single approval or when revoking all. This is partially because scheduling many `mt_on_revoke` calls when revoking all approvals could incur prohibitive [gas fees](https://docs.near.org/docs/concepts/gas). Apps and contracts which cache MT approvals can therefore not rely on having up-to-date information, and should periodically refresh their caches. Since this will be the necessary reality for dealing with `mt_revoke_all`, there is no reason to complicate `mt_revoke` with an `mt_on_revoke` call.

### No incurred cost for core MT behavior

MT contracts should be implemented in a way to avoid extra gas fees for serialization & deserialization of `approved_account_ids` for calls to `mt_*` methods other than `mt_tokens`. See `near-contract-standards` [implementation of `ft_metadata` using `LazyOption`](https://github.com/near/near-sdk-rs/blob/c2771af7fdfe01a4e8414046752ee16fb0d29d39/examples/fungible-token/ft/src/lib.rs#L71) as a reference example.
