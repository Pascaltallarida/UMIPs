## Headers
| UMIP-28     |                                                                                                                                          |
|------------|------------------------------------------------------------------------------------------------------------------------------------------|
| UMIP Title | Add STABLESPREAD as a price identifier              |
| Authors    | Bae (bae@youmychicfila.com), K (k@youmychicfila.com) |
| Status     | Draft                                                                                                                                    |
| Created    | December 12, 2020                                                                                                                           |
 
## Summary (2-5 sentences)
The DVM should support price requests for the STABLESPREAD price index. STABLESPREAD is defined as: `max(A - B + 1, 0)`, where `A` refers to an equally weighted basket of {UST, CUSD, BUSD}. B is `MUSD`. UST is TerraUSD, the interchain stablecoin connected by the Cosmos IBC. CUSD is Celo Dollar, the stablecoin on the Celo network. BUSD is Binance USD, a stablecoin issued by Paxos in partnership with Binance. `MUSD` is an autonomous and non-custodial stablecoin on mStable, representing a basket of stablecoins on Ethereum, namely USDT, USDC, TUSD, and DAI. 

## Motivation
The DVM currently does not support the STABLESPREAD price index. 
 
Supporting the STABLESPREAD price identifier would enable the creation of a linear combination of stablecoin baskets, both existing on Ethereum and other blockchains. Token minters could go short on the STABLESPREAD index, while token holders could go long or use synthetic STABLESPREAD for functional purposes.
 
There has been an increasing growth of non-Ethereum based stablecoins over the last few months, and especially over the last week in the case of UST, given the Mirror launch earlier this week. Each of these stablecoins have their own quirks and mechanisms in which they aim to target their peg. There appears to be somewhat of a growing dichotomy in terms of folks who believe that non-Ethereum based stable coins are able to retain their peg better. Currently, there does not exist a mechanism for users to express a view on the relative performance of which basket of stablecoins actually retains it peg better without actually acquiring the underlying constituents of the long basket and picking up a synthetic short position of the other basket. 

Further, we also believe STABLESPREAD can act as a defi credit spread primitive where folks can build other products on top, such as an insurance based product where the premium of the insurance is tied to the divergence between the two baskets of stablecoins. Additionaly, one can view the probability of default of a stablecoin as `1 - min(price, 1)`. Given that each basket of stablecoins is a convex combination, then it is also a convex combination of default probabilities. STABLESPREAD is then a difference of convex combination of default probabilities. We also believe that a CDS (credit default swap) can be built on top of this since they are effectively a function of default probabilities, and once STABLESPREAD gets whitelisted as a price identifier, we will start development on such products.

Let's walk through some cases to better understand what happens in various cases for STABLESPREAD. If both of these baskets were to trade at their theoretical value, then the resulting value of STABLESPREAD will be 1. If the Ethereum based stablecoin basket loses its peg and appreciates in value relative to the non-Ethereum stablecoin basket, then the value of STABLESPREAD will be less than 1. If the opposite happens, then it will be more than 1. It is important that this only measures "relative ability to maintain peg". That is, if both of them lose their peg in the same form, the resulting value still may be 1. 
 
There is little cost associated with adding this price identifier, as there are multiple free and easily accessible data sources available.
 
## Technical Specification
The definition of this identifier should be:
 
- Identifier name: STABLESPREAD
- Base Currency: STABLESPREAD
- Data Sources: {Bittrex: UST/USDT, Uniswap V2: UST/USDT} for UST, {Binance: BUSD/USDT, Uniswap V2: BUSD/USDT} for BUSD, {Bittrex: CUSD/USDT, OKCoin: CUSD/USD} for Celo Dollar, {Balancer: MUSD/USDC, Uniswap V2: MUSD/USDC} for MUSD
- Result Processing: For each constituent asset of the basket, the average of both exchanges
- Input Processing: None. Human intervention in extreme circumstances where the result differs from broad market consensus.
- Decimals: 8
- Rounding: Closest, 0.5 up
- Pricing Interval: 60 seconds
- Dispute timestamp rounding: down
 
## Rationale
Prices are primarily used by Priceless contracts to calculate a synthetic token’s redemptive value in case of liquidation or expiration. Contract counterparties also use the price index to ensure that sponsors are adequately collateralized.

For each of the constituents of STABLESPREAD- UST, CUSD, BUSD, MUSD, we picked the two most liquid exchanges, including both CeFi and DeFi sources for diversity. Further, each of the CeFi exchanges has "crypto SOTA" APIs with free REST + WSS functionality with generous rate limits and robust endpoints for prices that any user may hit, both real time and historically speaking. Most of these CeFi exchanges have 24/7 customer support + devops team ensuring highly available programmatic access and have strong redundancy across a variety of geographies. On the DeFi side, Uniswap V2 and Balancer are widely regarded as high quality AMMs. 

The objective for STABLESPREAD was to construct a portfolio of the most liquid and active non-Ethereum based stablecoins, and rather than using some complicated weighting scheme, we are initially imposing uniform weighting. If it turns out there is some user feedback around doing a market-cap weighting, ADV-weighting, etc. that can be future work for us. On the MUSD side, we were deciding between that and DUSD as both are convex combinations of liquid stablecoins but the latter is not liquid or active compared to the former. We also experimented with a variety of functional forms of computing a divergence of these two baskets to represent a credit spread. Computing the ratio of them had some promising properties, namely non-negativity constraints, but the nonlinearity of it had some subpar properties that might cause unfriendly UX as liquidations might cascade if the denominator went down X% as opposed to the numerator. So, we settled on an affine transformation, where if both baskets are indeed the same then the result will be 1.

## Implementation
 
The value of STABLESPREAD for a given timestamp can be determined with the following process.
 
1. UST, CUSD, BUSD, MUSD should be queried for from the exchanges listed in the "Technical Specification" section for the given timestamp rounded to the nearest second. The results of these queries should be kept at the level of precision they are returned at.
2. For each one, the average of the prices should be calculated.
3. Then, calculate 1/3 * UST + 1/3 * CUSD + 1/3 * BUSD, denote this as `A`
4. Perform A - MUSD + 1
5. Perform the maximum of the result of step 4 and 0. This is to ensure non-negativity. 
6. This result should be rounded to eight decimal places.
 
Additionally, [CryptoWatch API](https://docs.cryptowat.ch/rest-api/) is a useful reference. This feed should be used as a convenient way to query the price in realtime, but should not be used as a canonical source of truth for voters. Users are encouraged to build their own offchain price feeds that depend on other sources.
 
Voters are responsible for determining if the result of this process differs from broad market consensus. This is meant to be vague as $UMA tokenholders are responsible for defining broad market consensus.
 
This is only a reference implementation, how one queries the exchanges should be varied and determined by the voter to ensure that there is no central point of failure.
 
## Security considerations
Adding this new identifier by itself poses little security risk to the DVM or priceless financial contract users given this is dealing with some of the most liquid stablecoins in the whole cryptoverse, and the relative price volatility of them has been rather low due to their stablecoin properties. However, anyone deploying a new priceless token contract referencing this identifier should take care to parameterize the contract appropriately to avoid the loss of funds for synthetic token holders. Additionally, the contract deployer should ensure that there is a network of liquidators and disputers ready to perform the services necessary to keep the contract solvent.
 
$UMA-holders should evaluate the ongoing cost and benefit of supporting price requests for this identifier and also contemplate de-registering this identifier if security holes are identified.

