# Sepolia Order Book Showcase Runbook

This runbook demonstrates the essential lifecycle of the deployed Sepolia order book: full BUY and SELL fills,
cancellation, expiry and permissionless sweeping, amendment, partial fills, and cancellation after a partial fill.
It also includes short authorization and settlement-bound demonstrations.

## Deployment

| Item | Value |
|---|---|
| Network | Ethereum Sepolia (`11155111`) |
| Factory | [`0xa0DBa24F937E94d5fDE97CBA401a80e7570D1f67`](https://eth-sepolia.blockscout.com/address/0xa0DBa24F937E94d5fDE97CBA401a80e7570D1f67) |
| Order book | [`0x1D0fEc5b3d24af2b33A5107019871CA38F4f92D0`](https://eth-sepolia.blockscout.com/address/0x1D0fEc5b3d24af2b33A5107019871CA38F4f92D0) |
| Base token | [`0xD8d3947a5E73d7C6e305875ddF809dcB90eabf95`](https://eth-sepolia.blockscout.com/address/0xD8d3947a5E73d7C6e305875ddF809dcB90eabf95) (`TREX01`, 6 decimals) |
| Quote token | `0x756EfF2C697835bE39ea6D4B897305A207374cBE` (`FakeUSDC`, 6 decimals) |
| Initial showcase price | 1 base = 1 quote |

## Wallet roles

| Wallet | Role | Required assets |
|---|---|---|
| Wallet A | Maker | Base for SELL orders and quote for BUY orders |
| Wallet B | Taker | Quote for taking SELL orders and base for taking BUY orders |
| Wallet C | Independent recipient and expiry sweeper | Sepolia ETH for gas; no tokens required for sweeping |

Use different addresses for A and B. A maker cannot take its own order. Wallet C is used to demonstrate that the
settlement recipient can differ from the payer and that expired orders can be swept by anyone.

All three wallets need enough Sepolia ETH for their transactions. Fund Wallet A and Wallet B with at least 100
units of each test token so the scenarios can be run without replenishing balances.

## Units and common values

Both tokens have 6 decimals:

| Human amount | Contract amount |
|---:|---:|
| 1 token | `1_000_000` |
| 4 tokens | `4_000_000` |
| 6 tokens | `6_000_000` |
| 10 tokens | `10_000_000` |
| 12.5 tokens | `12_500_000` |

The price is represented by the ratio:

```text
price of one base token = totalQuoteAmount / totalBaseAmount
```

Therefore `(10_000_000 base, 10_000_000 quote)` is 1:1, while `(10_000_000 base, 12_500_000 quote)` is
1 base = 1.25 quote.

Use a normal expiry of at least the current Unix timestamp plus one hour. For the expiry scenario, use the current
timestamp plus 3–5 minutes so the make transaction has enough time to be mined.

`Side` is encoded as:

- `0` = `SELL`
- `1` = `BUY`

`OrderStatus` is encoded as:

- `0` = `NONE`
- `1` = `OPEN`
- `2` = `FILLED`
- `3` = `CANCELLED`
- `4` = `EXPIRED`

## Required quote, approval, and settlement-bound flow

Do not hard-code approval amounts or assume that fees remain unchanged. Before every write, call the matching quote
function from the wallet that will send the transaction:

| Write function | Quote first | Caller |
|---|---|---|
| `makeOrder` | `quoteMakeOrder` | Wallet A |
| `takeOrder` | `quoteTakeOrder(orderId, baseAmount, recipient)` | Wallet B |
| `amendOrder` | `quoteAmendOrder` | Wallet A |

Each returned `Quote` contains `(token, amount)`. The caller must execute ERC-20 `approve(orderBook, amount)` on
that token before the write. If an approval array is empty, no additional approval is required.

For `takeOrder`, copy the fresh `SettlementQuote` into the protection fields as follows:

| Order side | `maxBaseIn` | `maxQuoteIn` | `minBaseOut` | `minQuoteOut` |
|---|---:|---:|---:|---:|
| SELL | `0` | `settlement.totalInputAmount` | `settlement.netOutputAmount` | `0` |
| BUY | `settlement.totalInputAmount` | `0` | `0` | `settlement.netOutputAmount` |

The zero fields above are unused for that side. Use the three-argument `quoteTakeOrder` overload with the same
recipient that will be passed to `takeOrder`; changing the recipient can change the live fee quote.

For every scenario, record token balances and `escrowLiability(baseToken)` / `escrowLiability(quoteToken)` before
and after. This is a public shared contract, so validate deltas rather than assuming the contract's global liability
or token balance starts at zero.

## Scenario 1A — Full SELL order at 1:1

### Purpose

Show Wallet A selling 10 base tokens for 10 quote tokens, fully filled by Wallet B.

### Steps

1. From Wallet A, call `quoteMakeOrder` with:

   ```text
   params = (
     side: 0,
     totalBaseAmount: 10_000_000,
     totalQuoteAmount: 10_000_000,
     expiry: NORMAL_EXPIRY
   )
   ```

   Expect one approval for the base token, normally `10_000_000`.

2. From Wallet A, call base-token `approve`:

   ```text
   spender: ORDER_BOOK
   amount: amount returned by quoteMakeOrder
   ```

3. From Wallet A, call `makeOrder` with the same tuple. Capture `orderId` from `OrderMade`.

4. Confirm:

   - `statusOf(orderId)` is `OPEN` (`1`).
   - `getOrder(orderId).remainingBaseAmount` is `10_000_000`.
   - Base-token escrow liability increased by `10_000_000`.
   - Wallet A's base balance decreased by `10_000_000`.

5. From Wallet B, call `quoteTakeOrder(orderId, 10_000_000, WALLET_B)`.

6. From Wallet B, approve the returned input token and amount. For a SELL, the input token is quote.

7. From Wallet B, call `takeOrder`:

   ```text
   params = (
     orderId: SELL_ORDER_ID,
     baseAmount: 10_000_000,
     recipient: WALLET_B,
     maxBaseIn: 0,
     maxQuoteIn: settlement.totalInputAmount,
     minBaseOut: settlement.netOutputAmount,
     minQuoteOut: 0
   )
   ```

### Expected result

- `OrderTaken` is emitted with `remainingBaseAmount = 0`.
- `statusOf(orderId)` is `FILLED` (`2`).
- `remainingQuoteAmountOf(orderId)` is `0`.
- Base-token escrow liability decreases by `10_000_000`.
- Wallet B pays `settlement.totalInputAmount` quote and receives `settlement.netOutputAmount` base.
- Wallet A receives `settlement.grossInputAmount - settlement.makerFeeAmount` quote.
- `OrderFeePaid` events and the fee recipient's balance reflect any live maker or taker fees.

## Scenario 1B — Full BUY order at 1:1 with a separate recipient

### Purpose

Show Wallet A buying 10 base tokens with 10 quote tokens. Wallet B pays for the fill while Wallet C receives the
quote output, demonstrating payer/recipient separation.

### Steps

1. From Wallet A, call `quoteMakeOrder`:

   ```text
   params = (
     side: 1,
     totalBaseAmount: 10_000_000,
     totalQuoteAmount: 10_000_000,
     expiry: NORMAL_EXPIRY
   )
   ```

   Expect approval for the quote token, normally `10_000_000`.

2. Wallet A approves the returned quote amount and calls `makeOrder` with the same tuple. Capture `orderId`.

3. Confirm quote-token escrow liability increased by `10_000_000` and the order is `OPEN`.

4. From Wallet B, call `quoteTakeOrder(orderId, 10_000_000, WALLET_C)`. It is important to quote with Wallet C
   as the recipient because recipient-specific fee overrides can affect settlement.

5. Wallet B approves the returned input token and amount. For a BUY, the input token is base.

6. From Wallet B, call `takeOrder`:

   ```text
   params = (
     orderId: BUY_ORDER_ID,
     baseAmount: 10_000_000,
     recipient: WALLET_C,
     maxBaseIn: settlement.totalInputAmount,
     maxQuoteIn: 0,
     minBaseOut: 0,
     minQuoteOut: settlement.netOutputAmount
   )
   ```

### Expected result

- The order becomes `FILLED` and its remaining base and quote amounts become zero.
- Quote-token escrow liability decreases by `10_000_000`.
- Wallet B pays `settlement.totalInputAmount` base.
- Wallet C receives `settlement.netOutputAmount` quote.
- Wallet A receives `settlement.grossInputAmount - settlement.makerFeeAmount` base.
- The transaction emits `OrderTaken`; its `taker` is Wallet B and `recipient` is Wallet C.

## Scenario 2 — Make and cancel

### Purpose

Show that only the maker can cancel an open order and that all unfilled escrow is refunded.

### Steps

1. Wallet A quotes, approves, and makes a fresh 1:1 SELL order for 10 base / 10 quote with `NORMAL_EXPIRY`.
2. Optional negative check: from Wallet B, call `cancelOrder(orderId)`.
3. Confirm Wallet B's call reverts with `NotOrderMaker` and the order remains `OPEN`.
4. From Wallet A, call:

   ```text
   cancelOrder(orderId)
   ```

### Expected result

- Wallet B cannot cancel Wallet A's order.
- Wallet A's successful cancellation emits `OrderCancelled`.
- `statusOf(orderId)` is `CANCELLED` (`3`).
- `remainingQuoteAmountOf(orderId)` is `0`.
- Wallet A receives all `10_000_000` escrowed base tokens back.
- Base-token escrow liability decreases by `10_000_000`.
- A second cancellation attempt fails with `OrderNotOpen`.

## Scenario 3 — Expiry, failed take, and permissionless sweep

### Purpose

Show that expired liquidity cannot be taken and that Wallet C can release Wallet A's escrow without being the maker.

### Steps

1. Choose `SHORT_EXPIRY` equal to the current Unix timestamp plus 3–5 minutes.
2. Wallet A quotes, approves, and makes a 1:1 SELL order for 10 base / 10 quote using `SHORT_EXPIRY`.
3. Before expiry, confirm `statusOf(orderId)` is `OPEN`.
4. Wallet B obtains a take quote and approval while the order is still open, but does not take it yet. Retain the
   quoted `TakeOrderParams` for the negative transaction.
5. Wait until a Sepolia block has `block.timestamp >= SHORT_EXPIRY`.
6. Confirm `statusOf(orderId)` now reports `EXPIRED` (`4`), even before the refund is finalized.
7. From Wallet B, submit `takeOrder` using the retained parameters.
8. Confirm it reverts with `OrderExpired`; no balances or liabilities change.
9. From Wallet C, call:

   ```text
   sweepExpiredOrder(orderId)
   ```

### Expected result

- The post-expiry take fails and the order cannot settle.
- Wallet C can sweep even though it is neither maker nor taker.
- `ExpiredOrderSwept(orderId, WALLET_C, WALLET_A)` is emitted.
- `statusOf(orderId)` remains `EXPIRED`.
- Wallet A receives the full remaining base escrow back.
- `remainingQuoteAmountOf(orderId)` becomes `0`.
- Base-token escrow liability decreases by the remaining base amount.
- Calling `sweepExpiredOrder` before expiry would instead revert with `OrderNotExpired`.

## Scenario 4 — Amend an order and change its price

### Purpose

Show that amendment atomically cancels the old order and creates a replacement order with a new ID and price.

### Steps

1. Wallet A quotes, approves, and makes a SELL order for 10 base / 10 quote at 1:1.
2. Record the original `orderId` and confirm its remaining base is `10_000_000`.
3. From Wallet A, call `quoteAmendOrder`:

   ```text
   params = (
     orderId: ORIGINAL_ORDER_ID,
     expectedRemainingBaseAmount: 10_000_000,
     newBaseAmount: 10_000_000,
     newQuoteAmount: 12_500_000,
     newExpiry: NEW_NORMAL_EXPIRY
   )
   ```

   Because this changes only the price and keeps SELL base escrow at 10 tokens, the additional approvals and
   refunds should normally both be empty.

4. From Wallet A, call `amendOrder` with the same tuple. Capture `newOrderId` from `OrderAmended` or `OrderMade`.

### Expected result

- The original order becomes `CANCELLED` and cannot be taken.
- `OrderCancelled`, `OrderMade`, and `OrderAmended` are emitted atomically.
- `newOrderId` differs from the original ID.
- The replacement is `OPEN` with `10_000_000` base and `12_500_000` quote.
- Its new price is 1 base = 1.25 quote.
- Base escrow and base liability remain unchanged because the replacement requires the same base quantity.
- `expectedRemainingBaseAmount` protects against a fill racing the amendment; a stale value reverts with
  `RemainingBaseAmountChanged`.

Clean up by having Wallet A cancel the replacement order after the assertions.

## Scenario 5 — Partial take

### Purpose

Show proportional settlement while the order remains open.

### Steps

1. Wallet A quotes, approves, and makes a 1:1 SELL order for 10 base / 10 quote.
2. From Wallet B, call `quoteTakeOrder(orderId, 4_000_000, WALLET_B)`.
3. Wallet B approves the returned quote-token amount.
4. Wallet B calls `takeOrder`:

   ```text
   params = (
     orderId: ORDER_ID,
     baseAmount: 4_000_000,
     recipient: WALLET_B,
     maxBaseIn: 0,
     maxQuoteIn: settlement.totalInputAmount,
     minBaseOut: settlement.netOutputAmount,
     minQuoteOut: 0
   )
   ```

### Expected result

- `OrderTaken` reports `takenBaseAmount = 4_000_000`, `takenQuoteAmount = 4_000_000`, and
  `remainingBaseAmount = 6_000_000`.
- The order remains `OPEN`.
- `getOrder(orderId).remainingBaseAmount` is `6_000_000`.
- `remainingQuoteAmountOf(orderId)` is `6_000_000`.
- Base-token escrow liability decreases by `4_000_000`, leaving a 6-token liability delta for this order.
- Wallet B's balance changes follow the fresh settlement quote, including any taker fee.
- Wallet A receives the filled quote proceeds less any maker fee; the unfilled base remains escrowed.

To keep this scenario distinct from cancellation, either fill the remaining `6_000_000` using a fresh quote and
confirm the order becomes `FILLED`, or cancel it after recording the partial-fill assertions.

## Scenario 6 — Partial take followed by cancellation

### Purpose

Show that completed settlement remains final while only the unfilled maker escrow is refundable.

### Steps

1. Make a fresh 1:1 SELL order from Wallet A for 10 base / 10 quote.
2. Wallet B quotes, approves, and takes `4_000_000` base as described in Scenario 5.
3. Confirm the order is still `OPEN` with `6_000_000` remaining base and quote.
4. Record Wallet A's base balance immediately before cancellation.
5. From Wallet A, call:

   ```text
   cancelOrder(orderId)
   ```

### Expected result

- The order becomes `CANCELLED`.
- Wallet A receives exactly the remaining `6_000_000` base tokens, not the original 10 tokens.
- The 4 base already received by Wallet B is not reversed.
- `remainingQuoteAmountOf(orderId)` becomes `0`.
- Base-token escrow liability decreases by `6_000_000`.
- Wallet A retains the quote proceeds already earned from the partial fill.

## Scenario 7 — Basic authorization checks

These checks are short but useful during a contract showcase:

1. Wallet A makes a fresh order.
2. Wallet B calls `cancelOrder(orderId)`; expect `NotOrderMaker`.
3. Wallet B calls `amendOrder` for Wallet A's order; expect `NotOrderMaker`.
4. Wallet A calls `quoteTakeOrder` or `takeOrder` against its own order; expect `MakerCannotTakeOwnOrder`.
5. Confirm the order remains `OPEN` after every failed transaction.
6. Wallet A cancels the order to recover escrow.

## Final evidence checklist

Capture the following for the team presentation:

- Transaction link for every successful state change and at least one reverted transaction trace.
- `OrderMade`, `OrderTaken`, `OrderCancelled`, `OrderAmended`, `ExpiredOrderSwept`, and `OrderFeePaid` events.
- Old and replacement IDs for amendment.
- `statusOf`, `getOrder`, and `remainingQuoteAmountOf` results before and after each transition.
- Wallet A, B, and C token-balance deltas.
- Base and quote `escrowLiability` deltas.
- The quote used for every approval and take bound.
- Confirmation that full BUY and SELL fills exercised both token-transfer directions.

Run each scenario with a fresh order ID and finish or cancel every remaining open order. Do not reuse a stale quote
after changing the settlement recipient, fill amount, order remainder, or fee configuration.
