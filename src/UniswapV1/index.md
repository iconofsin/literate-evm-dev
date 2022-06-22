# Uniswap v1 - Overview

The original Uniswap was written in Vyper and consists of _two_ rather concise smart contracts.

- [Protocol](https://docs.uniswap.org/protocol/V1/introduction)
- [Repository](https://github.com/Uniswap/v1-contracts)
- [Lightweight Formal Verification of Uniswap Smart Contract](https://github.com/runtimeverification/verified-smart-contracts/tree/uniswap/uniswap)

The two contracts are the Factory and the Exchange.

The [Factory](./factory.md) allows to:
- Create Exchanges (each corresponding to an ETH-ERC20 pair; nowadays such exchanges are simply called _pairs_);
- Look up tokens and exchanges.

The [Exchange](./exchange.md):
- Holds reserves of ETH and of the associated ERC20 token;
- Allows swapping from ETH to the associated ERC20 token and vice versa;
- Allows adding liquidity to the exchange -- the liquidity provider deposits both ETH and the ERC20 token;
- Tracks liquidity contributions through an internal "pool token" (ERC20), which is minted and burned when liquidity is added and withdrawn;
- Allows swapping between any pair of ERC20 tokens using ETH as the connecting intermediary currency.

Uniswap v1 implements a "constant product" automated market maker algorithm, where the mathematical product of the balances of the two assets making up a pair is a constant. The exchange rate is based on the relative size of the ETH and ERC20 reserves.

"_Selling ETH for ERC20 tokens increases the size of the ETH reserve and decreases the size of the ERC20 reserve. This shifts the reserve ratio, increasing the ERC20 token's price relative to ETH for subsequent transactions._"

Swaps executed through Uniswap v1 Exchanges are charged with Liquidity Provider Fee (0.30%), which is added to the pair reserves. When liquidity providers burn their pool tokens after a period of time, they will normally receive more than the original contribution due to the LP Fee.

There can only be one Exchange per ERC20 token; this is to encourage pooling of liquidity in a single reserve for each given ERC20 token.

However, Uniswap v1 supports ERC20/ERC20 trades which involve both Uniswap's "singleton" pools and external, user-specified pools (more on that later.)

The developers made a conscious decision that Uniswap v1 contracts will be updated by releasing new, improved versions of the system, not by upgrading the original contracts; whereby liquidity providers will have an option of moving to the new system or keeping the liquidity in the old one.

