---
description: >-
  Building a prediction market that uses the UMA Optimistic Oracle V3 for
  settlement and event identification
---

# 🕖 Prediction Market

This section covers the [prediction market contract](https://github.com/UMAprotocol/dev-quickstart-oov3/blob/master/src/PredictionMarket.sol), which is available in the Optimistic oracle [quick-start repo](https://github.com/UMAprotocol/dev-quickstart-oov3/blob/master/src/PredictionMarket.sol) repo and enables the creation of a binary prediction market using [Optimistic Oracle V3 (OOV3)](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/optimistic-oracle-v3/implementation/OptimisticOracleV3.sol) assertions.

### Prediction Market

A prediction market smart contract allows individuals to make predictions and place bets on the likelihood of different outcomes for a future event. These types of contracts can be used for any kind of events such as:

1. Sports games
2. Cryptocurrency price predictions
3. Product launches
4. Political policy decisions

This smart contract enables the creation of prediction markets based on any off-chain event with three possible outcomes, two of which are chosen by the market creator and the third reflecting the remaining outcomes (i.e. tie or a draw in a sporting event).

By participating in the prediction market, individuals can create outcome tokens (shares) and buy and sell them of the different outcomes based on their assessment of the likelihood of each outcome occurring. The market determines the price of each share based on supply and demand, and individuals can profit by purchasing shares whose value grows as the likelihood of a certain event increases. Nevertheless, this contract does not cover the selling of outcome tokens; only their creation is covered.

### Development environment and tests

#### Clone the dev-quickstart-oov3 repo

```
git clone git@github.com:UMAprotocol/dev-quickstart-oov3.git
cd dev-quickstart-oov3
yarn
```

This project uses [forge](https://github.com/foundry-rs/foundry/tree/master/forge) as the ethereum testing framework. You will need to install foundry, refer to [foundry installation documentation](https://book.getfoundry.sh/getting-started/installation) if you don't have it already.

#### Compiling your contracts

We will need to run the following command to compile the contracts and make the typescript interfaces so that they are easy to work with:

```
forge build
```

#### Running the tests

To run the contracts tests written in solidity with forge run the following command:

```bash
forge test --match-path *PredictionMarket*
```

### Contract design

This contract allows the creation of any number of Markets, where each Market is a prediction market that consists of a pair of binary outcome tokens that represent the two possible outcomes of the prediction, such as "yes" and "no".

#### Contract creation and initialization

To initialize the state variables of the contract, the constructor takes three parameters:

1. `_finder`: address that represents the Finder contract, which is used to locate and retrieve other contracts within the system.
2. `_currency`: address that represents the currency that will be used for trading in the markets created by this contract.
3. `_optimisticOracleV3`: address that represents the OptimisticOracleV3, contract to assert truths about the world which are verified using an optimistic escalation game.

```solidity
    constructor(
        address _finder,
        address _currency,
        address _optimisticOracleV3
    ) {
        finder = FinderInterface(_finder);
        require(_getCollateralWhitelist().isOnWhitelist(_currency), "Unsupported currency");
        currency = IERC20(_currency);
        oo = OptimisticOracleV3Interface(_optimisticOracleV3);
        defaultIdentifier = oo.defaultIdentifier();
    }
```

#### Market creation

The `initializeMarket()` function is used to create a new market in the `PredictionMarket` contract.

Once the contract has been deployed, anyone can call `initializeMarket()` after approving the optional `reward` amount to be paid to the wallet that runs the assertion.

```solidity
    function initializeMarket(
        string memory outcome1, // Short name of the first outcome.
        string memory outcome2, // Short name of the second outcome.
        string memory description, // Description of the market.
        uint256 reward, // Reward available for asserting true market outcome.
        uint256 requiredBond // Expected bond to assert market outcome (OOv3 can require higher bond).
    ) public returns (bytes32 marketId) { ... }
```

To create a new market, the `initializeMarket()` function is called with the following parameters:

* `outcome1`: A short name for the first outcome (i.e., "yes").
* `outcome2`: A short name for the second outcome (i.e., "no").
* `description`: A description of the market and the event being predicted.
* `reward`: The amount of currency (in Wei) that will be available as a reward for the user that runs that creates the assertion in the OOV3, necessary to settle the market once it's ready.
* `requiredBond`: The amount of currency (in Wei) that users must bond to assert the outcome of the event.

Once the input checks are passed, the function creates two new ERC20 tokens (one for each outcome) using the `ExpandedERC20` contract, which extends the standard ERC20 contract by adding the ability to mint and burn tokens. The PredictionMarket is assigned the minter and burner roles to simplify how the outcome tokens handled when creating, redeeming or settling the tokens. \


The `Market` is then created and stored internally associated to a `marketId` returned by the `initializeMarket()` and emitted in the `MarketInitialized` event at the end of the process together with the parameters defining the market.

#### Create outcome tokens

The `createOutcomeTokens` function mints a pair of tokens representing the value of outcome1 and outcome2 for a given market identified by `marketId`. This allows participants to trade on the outcome of the market by exchanging these tokens with each other outside of the scope of the `PredictionMarket` contract.

The `createOutcomeTokens` function takes two parameters:

* `tokensToCreate`: an unsigned integer (`uint256`) value representing the number of tokens to be created for each outcome. The total number of tokens created will be twice this value, as two outcome tokens are created for each market.
* `marketId`: a `bytes32` value representing the unique identifier of the market for which the outcome tokens are being created.

\
The function first checks if the market exists. If the market exists, it transfers the specified amount of currency tokens from the caller to the contract using `safeTransferFrom` function from `currency` contract instance.

Then, the function mints `tokensToCreate` amount of both outcome1 and outcome2 tokens to the caller using the `mint` function of the respective `ExpandedERC20` token instances. Finally, it emits an event to notify that tokens have been created.

It is important to note that before calling this function, the caller must approve the contract to spend the required amount of currency tokens by calling the `approve` function on the `currency` contract instance.

#### Redeem outcome tokens

The `redeemOutcomeTokens` function burns an equal amount of `outcome1Token` and `outcome2Token` tokens for a given market and transfers the corresponding settlement currency tokens to the caller's account.

The function takes two parameters:

* `marketId`: a `bytes32` value representing the unique identifier of the market for which the tokens are being redeemed.
* `tokensToRedeem`: an unsigned integer (`uint256`) value representing the number of tokens of each outcome to be redeemed. The total number of tokens redeemed will be twice this value, as two outcome tokens are burned for each market.

Both parameters are passed to the function as arguments when it is called.

The function first retrieves the Market data from the storage using the provided marketId. Then, it checks if the outcome1Token is not equal to the zero address, which indicates the market exists.

Next, it burns the specified number of tokens for both outcome1Token and outcome2Token tokens using the `burnFrom()` function of their respective contracts, which decreases the balance of the caller's account.

Finally, it transfers the specified number of settlement currency tokens from the contract's account to the caller's account using the `safeTransfer()` function. The function emits a `TokensRedeemed` event, which includes the marketId, the caller's address, and the number of tokens redeemed.

#### Settle outcome tokens

The `settleOutcomeTokens` function allows a user to settle their outcome tokens for a given market and receive a payout in the settlement currency. The payout depends on the resolved market outcome and the number of tokens burned for each outcome.

The function takes one parameter:

* `marketId`: a `bytes32` value representing the unique identifier of the market for which the outcome tokens are being settled.

The `marketId` parameter is passed to the function as an argument when it is called.

The function first retrieves the Market data from the storage using the provided `marketId`. It then checks if the market has been resolved by verifying that the `resolved` flag in the market struct is set to `true`. If the market has not been resolved, the function throws an error.

Next, the function determines how many `outcome1Token` and `outcome2Token` tokens the caller has by calling the `balanceOf()` function of their respective contracts. The balances are stored in `outcome1Balance` and `outcome2Balance` variables.

After that, the function calculates the payout based on the resolved market outcome and the number of tokens burned for each outcome. If the market was resolved to the first outcome, then the payout equals the balance of `outcome1Token` while `outcome2Token` provides nothing. If the market was resolved to the second outcome, then the payout equals the balance of `outcome2Token` while `outcome1Token` provides nothing. If the market was resolved to the split outcome, then both outcome tokens provide half of their balance as currency payout.

Finally, the function burns the caller's outcome tokens using the `burnFrom()` function of their respective contracts, transfers the calculated payout from the contract's account to the caller's account using the `safeTransfer()` function, and emits a `TokensSettled` event, which includes the marketId, the caller's address, the payout amount, and the number of `outcome1Token` and `outcome2Token` tokens burned. The calculated payout is returned by the function.

#### Assert market

The function `assertMarket` is used to assert a market with any of the three possible outcomes: outcome1, outcome2, or unresolvable. The function takes two arguments:

* `marketId`, which is a unique identifier for the market
* `assertedOutcome`, which is a string representing the asserted outcome.

The function first checks that the market exists by verifying that the outcome1 token is not a null address. It then computes the hash of the asserted outcome using the `keccak256` function and checks that the market does not already have an active or resolved assertion. It also checks that the asserted outcome is one of the three allowed outcomes.

If all checks pass, the function sets the `assertedOutcomeId` of the market to the hash of the asserted outcome and computes the bond required to make the assertion. If the bond required by the market is higher than the minimum bond required by the oracle, the market bond is used instead. The function then composes a claim with the asserted outcome and the market description, pulls the bond from the caller's account, approves it for the oracle, and makes the assertion using the `_assertTruthWithDefaults` function.

Finally, the function stores information about the asserted market and emits a `MarketAsserted` event with the market ID, the asserted outcome, and the assertion ID.

#### Deployment

Run a local node with anvil (included in foundry toolkit) in a console:

```bash
anvil
```

In a second console continue running the rest of commands. Let's first export the environment variables for the wallets to use:

```bash
export PRIVATE_KEY_ONE="0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
export PRIVATE_KEY_TWO="0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d"
export PRIVATE_KEY_THREE="0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a"
export DEPLOYER_WALLET=$(cast wallet address --private-key "$PRIVATE_KEY_ONE")
export USER_WALLET=$(cast wallet address --private-key "$PRIVATE_KEY_TWO")
export ASSERTER_WALLET=$(cast wallet address --private-key "$PRIVATE_KEY_THREE")
export NODE_URL="http://127.0.0.1:8545"
```

Then Deploy the UMA Oracle Sandbox contracts to be used by running:

```bash
forge script script/OracleSandbox.s.sol \
--fork-url $NODE_URL \
--private-key "$PRIVATE_KEY_ONE" \
--sender $DEPLOYER_WALLET \
--broadcast
```

Find in the logs the `FINDER_ADDRESS` and export it with:

```bash
export FINDER_ADDRESS=<FINDER_ADDRESS>
```

Then use the Finder to export rest of addresses by running the following commands:

```bash
export OOV3_ADDRESS=$(cast call $FINDER_ADDRESS "getImplementationAddress(bytes32)(address)" $(cast --format-bytes32-string "OptimisticOracleV3"))
export DEFAULT_CURRENCY_ADDRESS=$(cast call $OOV3_ADDRESS "defaultCurrency()(address)")
```

We are ready to deploy the `PredictionMarket` contract with the following command:

```bash
export PREDICTION_MARKET_ADDRESS=$(forge create src/PredictionMarket.sol:PredictionMarket \
--private-key "$PRIVATE_KEY_ONE" \
--json \
--constructor-args $FINDER_ADDRESS $DEFAULT_CURRENCY_ADDRESS $OOV3_ADDRESS \
| jq -r .deployedTo)
echo "PREDICTION MARKET DEPLOYED TO " $PREDICTION_MARKET_ADDRESS
```

#### Interaction with the PredictionMarket

It's time initialise a market. Wes can first export the market parameters to use.\
\
Here the description shows that it's a market based on the outcome of a match between two teams. The two possible outcomes are `yes` or `no.` We offer `100` units of `DEFAULT_CURRENCY` to the asserter of the claim and require a bond of `5000` units of `DEFAULT_CURRENCY`to assert or dispute the assertion.

```bash
export DESCRIPTION="The Glacial Storms beat the Electric Titans on March 8, 2023 at 3:00 PM UTC, which is equivalent to the Unix timestamp 1686258000 seconds." \
export OUTCOME_ONE="yes"
export OUTCOME_TWO="no"
export REWARD=$(cast --to-wei 100)
export REQUIRED_BOND=$(cast --to-wei 5000)
```

We need to have the amount of rewards to the asserter and approve it before creating the market:

```bash
cast send --private-key "$PRIVATE_KEY_ONE" $DEFAULT_CURRENCY_ADDRESS "allocateTo(address,uint256)" $DEPLOYER_WALLET $REWARD
cast send --private-key "$PRIVATE_KEY_ONE" $DEFAULT_CURRENCY_ADDRESS "approve(address,uint256)" $PREDICTION_MARKET_ADDRESS $REWARD
```

Then we are ready to initialise the market with the `DEPLOYER_WALLET`

```bash
export MARKET_ID_TX=$(cast send \
--private-key "$PRIVATE_KEY_ONE" \
--json \
$PREDICTION_MARKET_ADDRESS "initializeMarket(string,string,string,uint256,uint256)(bytes32)" $OUTCOME_ONE $OUTCOME_TWO $DESCRIPTION $REWARD $REQUIRED_BOND \
| jq -r .transactionHash)

export MARKET_ID=$(cast receipt --json $MARKET_ID_TX | jq -r .logs[-1].topics[1])

export OUTCOME_TOKEN_ONE_ADDRESS=$(cast --abi-decode "MarketInitializedNonIndexed()(string,string,string,address,address,uint256,uint256)" $(cast receipt --json $MARKET_ID_TX | jq -r .logs[-1].data) | awk 'NR==4 {print $1}')
export OUTCOME_TOKEN_TWO_ADDRESS=$(cast --abi-decode "MarketInitializedNonIndexed()(string,string,string,address,address,uint256,uint256)" $(cast receipt --json $MARKET_ID_TX | jq -r .logs[-1].data) | awk 'NR==5 {print $1}')
```

Then we can mint the necessary tokens to then create the outcome tokens:

```bash
export AMOUNT=$(cast --to-wei 10000)
cast send --private-key "$PRIVATE_KEY_ONE" $DEFAULT_CURRENCY_ADDRESS "allocateTo(address,uint256)" $DEPLOYER_WALLET $AMOUNT
cast send --private-key "$PRIVATE_KEY_ONE" $DEFAULT_CURRENCY_ADDRESS "approve(address,uint256)" $PREDICTION_MARKET_ADDRESS $AMOUNT
```

We can now create the outcome tokens. With an amount `10000` units of `DEFAULT_CURRENCY` we get `10000` `OUTCOME_TOKEN_ONE` and 10000 `OUTCOME_TOKEN_TWO`

```bash
cast send \
--private-key "$PRIVATE_KEY_ONE" \
$PREDICTION_MARKET_ADDRESS "createOutcomeTokens(bytes32,uint256)" $MARKET_ID $AMOUNT
```

```bash
echo "BALANCE OUTCOME TOKEN ONE" $(cast call $OUTCOME_TOKEN_ONE_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
echo "BALANCE OUTCOME TOKEN TWO" $(cast call $OUTCOME_TOKEN_TWO_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
echo "BALANCE DEFAULT_CURRENCY" $(cast call $DEFAULT_CURRENCY_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
```

At any point before the market is settled we can redeem outcome tokens. By redeeming an amount we are burning the same amount of `OUTCOME_TOKEN_ONE` and `OUTCOME_TOKEN_TWO` to receive that amount of `DEFAULT_CURRENCY`

```bash
cast send \
--private-key "$PRIVATE_KEY_ONE" \
$PREDICTION_MARKET_ADDRESS "redeemOutcomeTokens(bytes32,uint256)" $MARKET_ID $(cast --to-wei 5000)
```

After redeeming 5000 tokens we can see how both balances of `OUTCOME_TOKEN_ONE` and `OUTCOME_TOKEN_TWO` have decreased 5000 and `DEFAULT_CURRENCY` has increased that same amount.

```bash
echo "BALANCE OUTCOME TOKEN ONE" $(cast call $OUTCOME_TOKEN_ONE_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
echo "BALANCE OUTCOME TOKEN TWO" $(cast call $OUTCOME_TOKEN_TWO_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
echo "BALANCE DEFAULT_CURRENCY" $(cast call $DEFAULT_CURRENCY_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
```

Now, let's simulate how the `DEPLOYER_WALLET` would trade one position of the market by transferring the remaining 5000  `OUTCOME_TOKEN_ONE` to another user. By doing this, `DEPLOYER_WALLET` is now only exposed to the outcome two ("no") because he only holds `OUTCOME_TOKEN_TWO`. On the other side, `USER_WALLET` is exposed to the outcome one ("yes") as he has traded some other currency against `OUTCOME_TOKEN_ONE`. This trade is out of the scope of this example, thats why we simulate it by running the following transfer:

```bash
cast send \
--private-key "$PRIVATE_KEY_ONE" \
$OUTCOME_TOKEN_ONE_ADDRESS "transfer(address,uint256)" $USER_WALLET $(cast --to-wei 5000)
```

At this point, let's imagine that the match between The Glacial Storms and the Electric Titans has taken place and that The Glacial Storms won. Then anyone can now `assert` that this has occoured by calling `assertMarket` with outcome `"yes"` as the claim defined in `DESCRIPTION` is true. We can do it, from the `ASSERTER_WALLET`, by running the following command:

```bash
cast send --private-key "$PRIVATE_KEY_ONE" $DEFAULT_CURRENCY_ADDRESS "allocateTo(address,uint256)" $ASSERTER_WALLET $REQUIRED_BOND
cast send --private-key "$PRIVATE_KEY_THREE" $DEFAULT_CURRENCY_ADDRESS "approve(address,uint256)" $PREDICTION_MARKET_ADDRESS $REQUIRED_BOND

export ASSERTION_TX=$(cast send \
--private-key "$PRIVATE_KEY_THREE" \
--json \
$PREDICTION_MARKET_ADDRESS "assertMarket(bytes32,string)" $MARKET_ID "yes" \
| jq -r .transactionHash)

export ASSERTION_ID=$(cast receipt --json $ASSERTION_TX | jq -r .logs[-1].topics[2])
```

Now, let's move forward 2 hours to go pass the challenge window of the assertion:

```bash
cast rpc evm_increaseTime 7200
cast rpc evm_min
```

No the assertion can be settled in the `OptimisticOracleV3`. We can do it by running the following command:

```bash
cast send \
--private-key "$PRIVATE_KEY_ONE" \
$OOV3_ADDRESS "settleAssertion(bytes32)" $ASSERTION_ID
```

We can now check how the `ASSERTER_WALLET` has received back the assertion bond plus the reward:

```bash
echo "ASSERTER WALLET BALANCE DEFAULT_CURRENCY" $(cast call $DEFAULT_CURRENCY_ADDRESS "balanceOf(address)(uint256)" $ASSERTER_WALLET)
```

Now, both the `DEPLOYER_WALLET` and `USER_WALLET` can settle their outcome tokens:

```bash
cast send \
--private-key "$PRIVATE_KEY_ONE" \
$PREDICTION_MARKET_ADDRESS "settleOutcomeTokens(bytes32)" $MARKET_ID

cast send \
--private-key "$PRIVATE_KEY_TWO" \
$PREDICTION_MARKET_ADDRESS "settleOutcomeTokens(bytes32)" $MARKET_ID
```

Finally we can see how the `USER_WALLET` won the bet, as he got `OUTCOME_TOKEN_ONE` so he now has `5000 DEFAULT_CURRENCY` and the deployer wallet only has `5000 DEFAULT_CURRENCY` from his initial 10000:

```bash
echo "DEPLOYER WALLET BALANCE OUTCOME TOKEN ONE" $(cast call $OUTCOME_TOKEN_ONE_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
echo "DEPLOYER WALLET BALANCE OUTCOME TOKEN TWO" $(cast call $OUTCOME_TOKEN_TWO_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)
echo "DEPLOYER WALLET BALANCE DEFAULT_CURRENCY" $(cast call $DEFAULT_CURRENCY_ADDRESS "balanceOf(address)(uint256)" $DEPLOYER_WALLET)

echo "USER WALLET BALANCE OUTCOME TOKEN ONE" $(cast call $OUTCOME_TOKEN_ONE_ADDRESS "balanceOf(address)(uint256)" $USER_WALLET)
echo "USER WALLET BALANCE OUTCOME TOKEN TWO" $(cast call $OUTCOME_TOKEN_TWO_ADDRESS "balanceOf(address)(uint256)" $USER_WALLET)
echo "USER WALLET BALANCE DEFAULT_CURRENCY" $(cast call $DEFAULT_CURRENCY_ADDRESS "balanceOf(address)(uint256)" $USER_WALLET)
```

\
