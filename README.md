# Order Book Test

This repository contains the Sepolia order-book showcase runbook and supporting test-token instructions.

## Mint FakeUSDC on Sepolia

FakeUSDC is deployed at `0x756EfF2C697835bE39ea6D4B897305A207374cBE` and uses 6 decimals.

1. Switch the connected wallet to Ethereum Sepolia.
2. Open [FakeUSDC — Write as Proxy, function 14](https://sepolia.etherscan.io/token/0x756eff2c697835be39ea6d4b897305a207374cbe?a=0xf7894d02ed910ab31edfd29500766281cd5ed366#writeProxyContract#F14).
3. Select **Connect to Web3** and connect the wallet that should receive the tokens.
4. Expand function **14. `mint(uint256)`**.
5. Enter `_amount` in 6-decimal base units, then select **Write** and confirm the transaction.

Common amounts:

| FakeUSDC | `_amount` |
|---:|---:|
| 1 | `1_000_000` |
| 10 | `10_000_000` |
| 100 | `100_000_000` |

This permissionless test function mints the requested FakeUSDC amount to the connected caller.

## Obtain TREX01 and bridge it to Sepolia

There are two ways to obtain TREX01:

1. Ask an admin to send TREX01 directly to your wallet.
2. Mint FakeUSDC on Pruv Test, use it to buy TREX01, and bridge the TREX01 to Sepolia.

### Buy TREX01 on Pruv Test

Pruv Test uses chain ID `7336` and RPC URL `https://rpc.testnet.pruv.network`.

| Token | Pruv Test address |
|---|---|
| FakeUSDC | [`0x756EfF2C697835bE39ea6D4B897305A207374cBE`](https://explorer.testnet.pruv.network/token/0x756EfF2C697835bE39ea6D4B897305A207374cBE) |
| TREX01 | [`0x5d5C787E29a807194F8bC82eE2CFBddce2407370`](https://explorer.testnet.pruv.network/token/0x5d5C787E29a807194F8bC82eE2CFBddce2407370) |

1. Add or switch the connected wallet to Pruv Test.
2. Open the Pruv Test [FakeUSDC contract](https://explorer.testnet.pruv.network/token/0x756EfF2C697835bE39ea6D4B897305A207374cBE), open **Contract → Write as Proxy**, and call `mint(uint256)`. FakeUSDC uses 6 decimals, so `10_000_000` mints 10 FakeUSDC to the connected caller.
3. Open the [TREX01 product](https://client.sto-dev.d3labs.io/product/96be7d1e-dcb6-46a3-8845-878dcd223244) in the Pruv client.
4. Connect the same wallet and buy TREX01 with FakeUSDC. The test product currently uses a 1 FakeUSDC = 1 TREX01 conversion rate.
5. After receiving TREX01 on Pruv Test, bridge it to the same wallet address on Sepolia.

### Bridge TREX01 to Sepolia

Complete the bridge manually through [Pruv Test Explorer](https://explorer.testnet.pruv.network/). The example below
bridges 10 TREX01 (`10_000_000` base units) to the same wallet address on Sepolia.

1. Switch the connected wallet to Pruv Test.
2. Open the Pruv Test [TREX01 contract](https://explorer.testnet.pruv.network/address/0x5d5C787E29a807194F8bC82eE2CFBddce2407370?tab=contract), select **Write as Proxy**, and connect the wallet.
3. Call `approve(address,uint256)` with:

   | Parameter | Value |
   |---|---|
   | `spender` | `0x660F4183C4bFBaA52569467E465FF320dB05e44E` |
   | `amount` | `10_000_000` |

4. Submit the approval and wait for it to succeed.
5. Convert the Sepolia recipient address to `bytes32`: remove its `0x` prefix, prepend 24 zeroes, and add `0x`
   again. For example, `0xF7894D02ed910aB31EDfD29500766281cD5Ed366` becomes
   `0x000000000000000000000000f7894d02ed910ab31edfd29500766281cd5ed366`.
6. Open the Pruv Test [TREX01 bridge router](https://explorer.testnet.pruv.network/address/0x660F4183C4bFBaA52569467E465FF320dB05e44E?tab=contract), select **Write as Proxy**, and connect the same wallet.
7. Call `transferRemote(uint32,bytes32,uint256)` with:

   | Parameter | Value |
   |---|---|
   | `_destination` | `11155111` |
   | `_recipient` | The `bytes32` recipient produced in step 5 |
   | `_amountOrId` | `10_000_000` |

8. Submit the bridge transaction and wait for it to succeed on Pruv Test. The destination delivery is a separate
   transaction and may take additional time to appear on Sepolia.

The bridged TREX01 is minted on Sepolia at [`0xD8d3947a5E73d7C6e305875ddF809dcB90eabf95`](https://eth-sepolia.blockscout.com/token/0xD8d3947a5E73d7C6e305875ddF809dcB90eabf95).
