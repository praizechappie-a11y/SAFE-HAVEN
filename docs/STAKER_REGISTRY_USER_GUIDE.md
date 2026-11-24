# Staker Registry User Guide

This guide covers the staker registry exposed by the current SAFE-HAVEN Soroban
contract. It explains registration, penalty accounting, reward claims, and
TypeScript SDK calls. Confirm the deployed contract version before relying on
these details.

> **Source/build caveat:** This checkout contains unresolved staker error
> variants and event helper references in the Rust contract. The walkthrough
> describes the operations as written in the source, but this checkout's
> deployability and runtime behavior have not been verified. Resolve the
> contract build errors before relying on these methods.

## Important limitations

- `register_staker(staker, amount)` records an accounting weight. It does not
  transfer or escrow tokens, and registering a new amount replaces the previous
  amount rather than adding to it.
- Early-exit penalties are tracked in one numeric rewards pool. The pool does
  not record which token a penalty came from, so amounts from different token
  denominations must not be treated as interchangeable.
- `claim_staker_rewards(staker)` updates the claimed-rewards counter and pool,
  but the current contract implementation does **not** transfer tokens to the
  staker. It returns `Result<(), VaultError>`, not a payout amount.
- Each claim is calculated against the current remaining pool and the current
  total registered stake. As the pool is reduced after each claim, claim order
  affects later claim amounts; this is not a fixed snapshot distribution.

These are current contract behaviors, not a promise of token custody, yield, or
a withdrawable reward. Check the deployed contract and its events before
assuming a claim results in a token transfer.

## How rewards enter the pool

When a timestamp- or ledger-based deposit is cancelled before unlock, the
contract calculates its penalty:

```text
penalty = deposit_amount × penalty_bps ÷ 10,000
fee_recipient_share = floor(penalty × 3,000 ÷ 10,000)
staker_pool_share = penalty - fee_recipient_share
```

The fee recipient gets 30% of the penalty (3,000 basis points); the remaining
70% is added to the staker rewards pool. For example, a 100-token penalty adds
70 to the pool and sends 30 to the fee recipient. Integer division truncates
fractions.

## Step-by-step: register

1. Select the address that will register and sign the transaction.
2. Choose a positive `amount` in the contract's accounting units.
3. Invoke `register_staker` with that address and amount.
4. To change the registered weight later, call it again with the new total
   amount; the contract replaces the old value.

Soroban CLI example:

```bash
soroban contract invoke \
  --id "$CONTRACT_ID" \
  --source "$USER_SECRET" \
  --network "$NETWORK" \
  --rpc-url "$RPC_URL" \
  --network-passphrase "$NETWORK_PASSPHRASE" \
  -- register_staker \
  --staker "$USER_ADDRESS" \
  --amount 1000
```

Zero and negative amounts fail with `InvalidStakeAmount`. Registration requires
authorization from the staker.

## Step-by-step: accumulate and claim rewards

1. Register with a positive amount before attempting to claim.
2. Wait for one or more eligible early deposit cancellations to add penalties
   to the rewards pool.
3. Invoke `claim_staker_rewards` using the registered address as the signer.
4. A successful invocation updates accounting. Remember the limitations above:
   it does not transfer tokens, and its return value should not be interpreted
   as a token payout.

CLI example:

```bash
soroban contract invoke \
  --id "$CONTRACT_ID" \
  --source "$USER_SECRET" \
  --network "$NETWORK" \
  --rpc-url "$RPC_URL" \
  --network-passphrase "$NETWORK_PASSPHRASE" \
  -- claim_staker_rewards \
  --staker "$USER_ADDRESS"
```

The call fails with `StakerNotFound` if the address is not registered. It fails
with `NoRewardsToClaim` if the pool is empty or the calculated integer reward
rounds to zero. A successful response is `()`, not an amount.

## Reward calculation

For each claim, the contract currently calculates:

```text
claim_amount = floor(staker_amount × current_rewards_pool / total_staked)
```

`total_staked` is the sum of all registered amounts. For example, with 1,000
registered units for Alice, 3,000 for Bob, and 700 in the pool, Alice's first
claim is `floor(1,000 × 700 / 4,000) = 175`. The pool then becomes 525. If Bob
claims next, the current calculation yields
`floor(3,000 × 525 / 4,000) = 393`, not 525. Fractions are truncated, and
claiming does not reserve a fixed share for each staker.

## TypeScript SDK example

The following uses `@stellar/stellar-sdk` v12, consistent with the project
frontend. Set the transaction source to the staker's account, simulate the
operation, and pass the prepared transaction to your wallet for signing and
submission. Keep signing keys in a wallet; do not put secret keys in source
code.

```ts
import {
  Address,
  BASE_FEE,
  Contract,
  Networks,
  TransactionBuilder,
  nativeToScVal,
  rpc,
} from "@stellar/stellar-sdk";

const server = new rpc.Server(RPC_URL);
const contract = new Contract(CONTRACT_ID);
const networkPassphrase = Networks.TESTNET;

// Use the staker's account as the transaction source.
const account = await server.getAccount(STAKER_ADDRESS);

// Registration: amount is an i128.
const registerOperation = contract.call(
  "register_staker",
  Address.fromString(STAKER_ADDRESS).toScVal(),
  nativeToScVal(1000n, { type: "i128" }),
);

// For a claim, use this operation in place of registerOperation:
const claimOperation = contract.call(
  "claim_staker_rewards",
  Address.fromString(STAKER_ADDRESS).toScVal(),
);

const transaction = new TransactionBuilder(account, {
  fee: BASE_FEE,
  networkPassphrase,
})
  .addOperation(registerOperation) // Replace with claimOperation to claim.
  .setTimeout(30)
  .build();

const simulation = await server.simulateTransaction(transaction);
if (rpc.Api.isSimulationError(simulation)) {
  throw new Error(simulation.error);
}

const prepared = rpc.assembleTransaction(transaction, simulation).build();
// Have the staker's wallet sign `prepared`, then submit it with this RPC server.
```

The invocation argument order and types are `Address, i128` for registration
and `Address` for claims. A claim's successful return value is unit.

## FAQ

### Does registration take custody of my tokens?

No. The registered amount is stored as a weight only; this method does not
transfer or lock tokens.

### Do I receive tokens when I claim?

Not with the current implementation. A claim updates the accounting pool and
claimed counter, but performs no token transfer.

### How is a new amount registered?

Calling `register_staker` again replaces the existing amount. Pass the new total
weight, not just the increase.

### Why did a later claim calculate a different amount?

Claims use the pool remaining at the time of each call. Earlier claims reduce
that pool, so claim order affects later calculations.

### What if I have registered but the pool is empty?

The claim fails with `NoRewardsToClaim`. A penalty must first be added to the
pool, and the calculated amount must be greater than zero after integer
rounding.

### Can rewards from different tokens be combined?

The pool has no token identifier. It is not safe to treat values denominated in
different tokens as a common reward balance.

### Can I query my registered amount or unregister?

The current public contract API does not expose a staker/pool read method or an
unregister operation. Registration amounts must be positive, so setting the
weight to zero is not a supported way to unregister.
