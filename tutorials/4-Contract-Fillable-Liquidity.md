One of the most powerful things that 0x has to offer is large pool of liquidity with competitive pricing that can be used to implement “contract fillable liquidity”. Any developer that needs exchange functionality from within their own smart contract can use 0x to do so. Here are some examples of contract fillable liquidity:

-   A Dapp such as [CDP Saver](https://cdpsaver.com/) that aims to abstract the usage of MKR to interact with your CDP by performing swaps from ETH or DAI to MKR without any thought from the user.
-   A margin trading protocol, like [dYdX](https://medium.com/dydxderivatives/dai-usdc-market-launches-on-dydx-bde957c48e2e) that wants to give their users access to deep liquidity pools.
-   A simple script that sells ETH for DAI on a consistent basis over a week and deposits it into a lending protocol like Compound.

The following content will help explain 0x key concepts and describe how you can start implementing some of these examples, today.

### Key Concepts

There are two key concepts that will be good to understand in order to make best use of this guide:

#### 0x order

The 0x order is a message with a [specific structure](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#order-message-format). Conceptually, you can think of a 0x order as a promise from one user (the maker) to swap their token (the makerAsset) with another token (the takerAsset) at a specified rate. These orders can be fulfilled by interacting with the [0x exchange contract](https://etherscan.io/address/0x4f833a24e1f95d70f028921e27040ca56e09ab0b).

#### 0x exchange contract

The 0x exchange contract is a smart contract deployed on the [Ethereum mainnet](https://etherscan.io/address/0x4f833a24e1f95d70f028921e27040ca56e09ab0b) and most [testnets](https://0x.org/wiki#Deployed-Addresses). Users send transactions to this smart contract in order to fulfill 0x orders. There are many options for filling orders using the contract, they are listed [here](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#filling-orders).

### Enabling Contract Fillable Liquidity

Enabling contract fillable liquidity using 0x requires 2 steps:

1. Find 0x orders
2. Fill 0x orders

#### Step 1: Find 0x Orders

An easy way to find 0x orders is to query a 0x relayer via the [Standard Relayer REST API specification](http://sra-spec.s3-website-us-east-1.amazonaws.com/). Interactions with these APIs occur entirely off-chain. This means that Ethereum transactions are not involved yet. Try running the command below:

```bash
curl -i https://api.radarrelay.com/0x/v2/orderbook?baseAssetData=0xf47261b000000000000000000000000089d24a6b4ccb1b6faa2625fe562bdd9a23260359&quoteAssetData=0xf47261b0000000000000000000000000c02aaa39b223fe8d0a0e5c4f27ead9083c756cc2
```

If your request successfully completes, congrats! You have retrieved orders from Radar Relay’s [ETH/DAI market](https://app.radarrelay.com/WETH/DAI).

We also provide a suite of browser-ready JavaScript libraries (complete with TypeScript types) that aid in finding orders. Take a look at the following TypeScript example that combines two of our npm packages, [0x.js](https://github.com/0xProject/0x-monorepo/tree/development/packages/0x.js) and [@0x/connect](https://github.com/0xProject/0x-monorepo/tree/development/packages/connect).

```javascript
import { assetDataUtils } from '0x.js';
import { HttpClient } from '@0x/connect';

const mainAsync = async () => {
    // ERC-20 Token addresses that you are interested in
    const WETH_TOKEN_ADDRESS = '0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2';
    const DAI_TOKEN_ADDRESS = '0x89d24a6b4ccb1b6faa2625fe562bdd9a23260359';
    // Standard relayer api url
    const RELAYER_API_URL = 'https://api.radarrelay.com/0x/v2/';
    // Construct client to talk to a relayer
    const relayer = new HttpClient(RELAYER_API_URL);
    // Construct wETH/DAI orderbook request
    const baseAssetData = assetDataUtils.encodeERC20AssetData(WETH_TOKEN_ADDRESS);
    const quoteAssetData = assetDataUtils.encodeERC20AssetData(DAI_TOKEN_ADDRESS);
    const orderbookRequest = { baseAssetData, quoteAssetData };
    // Get wETH/DAI orderbook
    const response = await httpClient.getOrderbookAsync(orderbookRequest);
    const buyOrders = response.bids.records.map(r => r.order);
    const sellOrders = response.asks.records.map(r => r.order);
    // View orders that want to buy wETH with DAI
    console.log(buyOrders);
    // View orders that want to sell wETH for DAI
    console.log(sellOrders);
};
mainAsync().catch(console.error);
```

Note: Before trying to fill an order, make sure to verify its price. By sending the order to the 0x exchange contract for settlement, you are accepting to fill the order at the price specified. In general, it is good to include some validation and pruning between these two steps in order to maximize the chance for a successful fill. Worst case, if you try to fill an expired or invalid order, the Ethereum transaction will revert and no tokens will change hands.

#### Step 2: Fill 0x Orders

Now that you have 0x orders, you can use them to help fulfill the token swap needs of your users. By leveraging the 0x smart contracts, we can fill the 0x orders that we have found in step 1.

The 0x exchange contract is available for developers to use within their own smart contracts. This is a powerful feature as it allows you to guarantee atomicity among multiple actions. Check out the following solidity example. In this case, a smart contract first gets a Compound loan for wETH before performing a token swap from wETH to DAI. Because these actions are occurring in the same transactions, if the trade fails, the loan is also reverted.

```javascript
pragma solidity ^0.5.10;
pragma experimental ABIEncoderV2;

import "@0x/contracts-exchange-libs/contracts/src/LibOrder.sol";
import "@0x/contracts-exchange/contracts/src/interfaces/IExchange.sol";

function loanAndFill(
    uint256 fillAmount,
    LibOrder.Order[] memory orders,
    bytes[] memory signatures
)
    external
{
    // Borrow token
    CErc20 cToken = CErc20(0x4ddc2d193948926d02f9b1fe9e1daa0718270ed5);
    require(
        cToken.borrow(100) == 0,
        "got collateral?"
    );

    // Approve 0x ERC20Proxy to spend cToken
    require(
        cToken.approve(0x2240dab907db71e64d3e0dba4800c83b5c502d4e, 2**256 - 1),
        "ERC20 approval required"
    );

    // Fill orders using borrowed tokens
    IExchange exchange = IExchange(0x4f833a24e1f95d70f028921e27040ca56e09ab0b);
    exchange.marketBuyOrders(orders, fillAmount, signatures);
}

```

There are many different ways to fill orders and `marketBuyOrders` is just one of them. For a complete list of options for filling check out the [0x protocol specification](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#filling-orders). Each of these options has a corresponding convenience method in the [0x.js library](https://0x.org/docs/contract-wrappers#ExchangeWrapper-fillOrderAsync).

### Conclusion

That’s it, with a flexible suite of tools available, you too can find and fill 0x orders in order to provide your users with UX enhancing token swaps. Please reach out to us in the #questions channel under Dev Support in [Discord](https://discord.gg/d3FTX3M).

We also provide a range of other material and documentation to help you get started playing with 0x. Check them out:

-   Get familiar with other 0x key concepts and features by coding right in your browser: https://codesandbox.io/s/github/0xproject/0x-codesandbox

-   Check out the 0x starter project for additional examples: https://github.com/0xProject/0x-starter-project

### Faq

#### What if I don’t use JavaScript or TypeScript?

0x smart contracts are deployed on the ethereum mainnet and are free and open for anyone to use. Although we provide convenience libraries in JavaScript, any language is perfectly capable with interacting with ethereum. That being said, we do provide some additional libraries for python tooling here and we are actively working on broader multi-language support in the future.

#### What is 0x Mesh and how will it help me to program token swaps?

0x Mesh (https://blog.0xproject.com/0x-roadmap-2019-part-3-networked-liquidity-0x-mesh-9a24026202b3) helps connect you with other users that have the orders you want. In the near future you will be able to leverage 0x Mesh as an alternative source for finding orders.

#### What is the roadmap for future tooling + resources by the 0x core team?

We are currently working on revamping of our asset-buyer library to support ERC to ERC liquidity needs and provide varying forms of the order for on-chain consumption (calldata, signed orders, parameters for a web3 call). In addition to a new library, we will be adding comprehensive guides to tapping into the output of the new asset-buyer on-chain.
