# Exchange

Exchange ([Github](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_exchange.vy)) is a much more involved beast than [Factory](./factory.md), mostly due to its broader functionality (check out [Overview](./index.md) and the external links to grasp the entire feature set).

NB: Uniswap v1 Exchange was compiled and deployed ([Etherscan.io](https://etherscan.io/address/0x2157A7894439191e520825fe9399aB8655E0f708), [tx](https://etherscan.io/tx/0x9d30e48fb33a7b760d0b5c33253eb5a774c9dc8e9ee2768ad3425249a3319e09)) using Vyper [v0.1.0b4](https://vyper.readthedocs.io/en/v0.1.0-beta.4/), which is *old*, no matter how you slice it. Consequently, the syntax of the language used at the time differs considerably from what you'll see in the later versions of Vyper.


```python
contract Factory():
    def getExchange(token_addr: address) -> address: constant
```

Same as Factory, Exchange uses `contract` to define interfaces (instead of the more modern `interface`). The one above is a view function interface for Factory that resolves an ERC20 token address into the corresponding Exchange proxy address. 


```python
contract Exchange():
    def getEthToTokenOutputPrice(tokens_bought: uint256) -> uint256(wei): constant
    def ethToTokenTransferInput(min_tokens: uint256, deadline: timestamp, recipient: address) -> uint256: modifying
    def ethToTokenTransferOutput(tokens_bought: uint256, deadline: timestamp, recipient: address) -> uint256(wei): modifying
```

Because Exchange is capable of interacting with other Exchange instances for the more complex trades, it needs to be able to talk to identical Exchange instances. This interface defines a view function `getEthToTokenOutputPrice`, and two state-modifying functions, `ethToTokenTransferInput` and `ethToTokenTransferOutput`. All three functions, naturally, are part of Exchange's code.

Pay attention to the `uint256(wei)` type. Back in the day, Vyper had _unit types_[^unittypes], which were specializers for regular integer-based types that served to strenghten typing and prevent the developer from mixing, for instance, timestamps and wei amounts in math expressions.

// TODO: circle back   


```python
TokenPurchase: event({buyer: indexed(address), eth_sold: indexed(uint256(wei)), tokens_bought: indexed(uint256)})
EthPurchase: event({buyer: indexed(address), tokens_sold: indexed(uint256), eth_bought: indexed(uint256(wei))})
AddLiquidity: event({provider: indexed(address), eth_amount: indexed(uint256(wei)), token_amount: indexed(uint256)})
RemoveLiquidity: event({provider: indexed(address), eth_amount: indexed(uint256(wei)), token_amount: indexed(uint256)})
Transfer: event({_from: indexed(address), _to: indexed(address), _value: uint256})
Approval: event({_owner: indexed(address), _spender: indexed(address), _value: uint256})
```

Next, we declare events. `TokenPurchase`, `EthPurchase`, `AddLiquidity`, and `Remove Liquidity` are native to the protocol, while `Transfer` and `Approval` are intended for compatibility with the ERC20.

`TokenPurchase` is emitted when an ETH amount is traded for an ERC20 amount; with buyer's address (`buyer`), amount of ETH sold (`eth_sold`), and amount of ERC20 bought (`tokens_bought`) as the arguments.

`EthPurchase` is the reverse, emitted when an ERC20 token is tradef for ETH; with buyer's address (`buyer`), amount of ERC20 sold (`tokens_sold`), and ETH received (`eth_bought`) as the arguments.

The next pair of events, `AddLiquidity` and `RemoveLiquidity`, indicates a liquidity provider adding or withdrawing liquidity to/from the exchange. In both cases, the events include the provider's address (`provider`) and the amounts of ETH and ERC20 added or withdrawn (`eth_amount`, `token_amount`).

`Transfer` and `Approval` follow the ERC20 specification[^erc20]. 


```python
name: public(bytes32)                             # Uniswap V1
symbol: public(bytes32)                           # UNI-V1
decimals: public(uint256)                         # 18
totalSupply: public(uint256)                      # total number of UNI in existence
balances: uint256[address]                        # UNI balance of an address
allowances: (uint256[address])[address]           # UNI allowance of one address on another
token: address(ERC20)                             # address of the ERC20 token traded on this contract
factory: Factory                                  # interface for the factory that created this contract
```

Almost every state variable defined in the Exchange contract has to do with supporting the ERC20 specification.

Because each Exchange (trading pair) mints its own ERC20 token to track liquidity contributions, obligations, and LP earnings, it follows that it must also be an ERC20 token. `name`, `symbol`, `decimals`, `totalSupply`, `balances`, and `allowances` all have to do with implementing that logic. 

The ERC20 token in question is known as the _LP Token of a pair_ (and here, the developers have annotated it as the UNI token.)

`name` and `symbol` are supposed to be strings, but back in the old Vyper the separate string type didn't exist; instead, we would store string data in fixed-size arrays, of which `bytes32`, a 32-byte array, is an instance[^strings].

`decimals`, according to the ERC20 spec, is the number of decimal places to be used _for display purposes_. This value does not affect in-contract handling of token amounts, which are always expressed in `wei`, and defaults to `18`, which, as we will see shortly, the Exchange contract honors.

`totalSupply` is the amount of LP Token in existence. Since each Exchange instance mints and burns its own tokens, based on liquidity contributions and withdrawals, the value of `totalSupply` here is based strictly on these two operations. 

`balances` maps liquidity provider addresses to amounts of LP Token they hold. `totalSupply` is expected to equal the sum of all balances both before and after a transaction operating on LP Token takes place.

`allowances` stores spending allowances for LP Token for any potential liquidity provider who might wish to allow a third party to spend tokens on their behalf.

`token` is as the comment indicates. 

NOTE. The type of `token` specializes on `ERC20` (interface), which is to say `token` is supposed to be an address of an ERC20 contract, but normally in a Vyper contract we'd see a prior `from vyper.interfaces import ERC20` declaration to make this possible. Exchange has no such declarations, and this is one of the mysteries of v1's code. 

`factory` is an interface for `Factory` and is going to point to the Factory that deployed this Exchange.
This is used for what is now known as _swap routing_, i.e. achieving a swap by way of multiple Exchange hops,
and for verification purposes.

Note the variables not declared as `public` are private by default. 

NOTE. Mapping declarations of the form `_valueType[_keyType]` follow the syntax of the Vyper version used (`0.1.0b4`), but also `0.1.0b4` will error when compiling Uniswap v1 code. Same goes for [Factory](./factory.md), and I have been unable to solve this mystery so far. Please let me know if _you_ know what's up.

## setup

```python
# @dev This function acts as a contract constructor which is not currently supported in contracts deployed
#      using create_with_code_of(). It is called once by the factory during contract creation.
@public
def setup(token_addr: address):
    assert (self.factory == ZERO_ADDRESS and self.token == ZERO_ADDRESS) and token_addr != ZERO_ADDRESS
    self.factory = msg.sender
    self.token = token_addr
    self.name = 0x556e697377617020563100000000000000000000000000000000000000000000
    self.symbol = 0x554e492d56310000000000000000000000000000000000000000000000000000
    self.decimals = 18
```

Remember how we said `create_with_code_of` is misleading? It's 2022, and calling constructors on proxies still isn't anywhere to be seen (and we're fine with that.)

The first thing `setup` does is checks that the Exchange has not been initialized and so this is the first time `setup` is being executed on this instance of Exchange. Calling `setup` sets state variables `factory` and `token`, so if any of those isn't the zero address, the function will revert. 

The address of the ERC20 that will be traded on the Exchange (`token_addr`) and is passed in as the only argument must, on the other hand, be non-zero.

If the assert doesn't fire, we:
- Set state variable `factory` to `msg.sender`, which is the caller's address and in our case is the address of Factory.
- Set state variable `token` to `token_addr`. 
- Set state variable `name` (the name of the LP Token) to 0x556e697377617020563100000000000000000000000000000000000000000000, which is another way of saying "Uniswap V1".
- Set state variable `symbol` (the ticker of the LP Token) to 0x554e492d56310000000000000000000000000000000000000000000000000000, which translates to "UNI-V1".
- Set state variable `decimals` (for the LP token) to 18. Every 1e18 wei of the LP Token will be displayed as `1` by wallets and token trackers.

(Shout out to [Online String Tools](https://onlinestringtools.com/convert-bytes-to-string) for making the conversion of `bytes32` to `string` trivial.)

Like often happens with EVM, we ignore initialization for `totalSupply` as it is `0` by default. `balances` and `allowances` are "initialized" in a similar manner, which is great since we can't know their values in advance anyway.


## addLiquidity

It's easy to observe creating an Exchange (by calling `setup`) does not automatically make it functional. All `setup` does is configures an ETH/ERC20 pair. But to actually be able to swap ETH for ERC20 and back, someone needs to provide both ETH and ERC20 liquidity to the Exchange.

This is where liquidity providers come in, and `addLiquidity` is the function that allows anyone to contribute liquidity to an existing Exchange. 

So once an Exchage is set up, how much of ETH and of the ERC20 token do you actually supply? 

Can you be a jerk about it and create an ETH/DAI pair where the reserve of ETH is 1e18 (1 ETH), and the reserve of DAI is 1e24 (10,000 DAI) and then just wait for someone to buy the 1 ETH for 10,000 DAI and withdraw liquidity afterwards, thus making a lump amount?

Nope. The potential buyer will quickly realize how dim-witted your ruse is, and instead of buying the 1 ETH, they will buy the 10,000 DAI, only paying 1 ETH.

// TODO: we can't actually buy the entirety of one side of the reserves, can we? check

Okay, then maybe we can set up an ETH/DAI pair where the reserve of ETH is 1e18 (1 ETH), and the reserve of DAI is 1 wei, then wait for someone to buy the 1 wei of DAI for 1 ETH?

Nope. The someone will simply sell 1 wei of DAI to the Exchange and get 1 ETH back.

Uniswap developers were wise enough to know their Automated Market Maker approach will force the initial liquidity contributor to go with the existing market exchange rates, unless, of course, they wanted to lose money.

Once there is liquidity in an Exchange, adding more liquidity happens at the current exchange rate in that Exchange. And the rest is taken care of by the AMM algorithm, which we'll get to later. For now, let's dissect how `addLiquidity` works. 

(Also note here, if someone is stupid enough to attemp to piggyback your genius imbalanced exchange rate ruse, you will both lose liquidity.)

```python
# @notice Deposit ETH and Tokens (self.token) at current ratio to mint UNI tokens.
# @dev min_liquidity does nothing when total UNI supply is 0.
# @param min_liquidity Minimum number of UNI sender will mint if total UNI supply is greater than 0.
# @param max_tokens Maximum number of tokens deposited. Deposits max amount if total UNI supply is 0.
# @param deadline Time after which this transaction can no longer be executed.
# @return The amount of UNI minted.
@public
@payable
def addLiquidity(min_liquidity: uint256, max_tokens: uint256, deadline: timestamp) -> uint256:
    assert deadline > block.timestamp and (max_tokens > 0 and msg.value > 0)
    total_liquidity: uint256 = self.totalSupply
    if total_liquidity > 0:
        assert min_liquidity > 0
        eth_reserve: uint256(wei) = self.balance - msg.value
        token_reserve: uint256 = self.token.balanceOf(self)
        token_amount: uint256 = msg.value * token_reserve / eth_reserve + 1
        liquidity_minted: uint256 = msg.value * total_liquidity / eth_reserve
        assert max_tokens >= token_amount and liquidity_minted >= min_liquidity
        self.balances[msg.sender] += liquidity_minted
        self.totalSupply = total_liquidity + liquidity_minted
        assert self.token.transferFrom(msg.sender, self, token_amount)
        log.AddLiquidity(msg.sender, msg.value, token_amount)
        log.Transfer(ZERO_ADDRESS, msg.sender, liquidity_minted)
        return liquidity_minted
    else:
        assert (self.factory != ZERO_ADDRESS and self.token != ZERO_ADDRESS) and msg.value >= 1000000000
        assert self.factory.getExchange(self.token) == self
        token_amount: uint256 = max_tokens
        initial_liquidity: uint256 = as_unitless_number(self.balance)
        self.totalSupply = initial_liquidity
        self.balances[msg.sender] = initial_liquidity
        assert self.token.transferFrom(msg.sender, self, token_amount)
        log.AddLiquidity(msg.sender, msg.value, token_amount)
        log.Transfer(ZERO_ADDRESS, msg.sender, initial_liquidity)
        return initial_liquidity
```


### Arguments & Return Value
In addition to the three explicit arguments (`min_liquidity`, `max_tokens`, and `deadline`), there's an implicit one, and that's the ETH amount transferred to the contract in a call to `addLiquidity`. To facilitate this, `addLiquidity` is declared _payable_  (note the `@payable` decorator), which means it will accept ETH amounts alongside function arguments.

`msg.value` is how we access this implicit argument, and its value equals the ETH amount transferred in the transaction.

`msg.value` or _ETH value_ is one half (50%) of the total value added in an `addLiquidity` transaction, or at least it is expected to be. The other 50% of the value is added in the form of the ERC20 token.

Since _ETH value_ determines the contributed value, the amount of ERC20 token transferred to the Exchange in an `addLiquidity` transaction is based on `msg.value` and the current exchange rate. (Recall we said earlier that new liquidity is always added at the current exchange rate.)

Now, EVM networks are designed in such a way that submitted transactions first go into **mempool**, and then, after a hopefully short period of time, get included in a mined block. Consequently, there's normally a time gap between the moment when a transaction is submitted and the moment when it's processed. Plus, other transactions will get processed earlier.

So by the moment an `addLiquidity` transaction is executed and included in a mined block the exchange rate could change.

This is known as **slippage**. To avoid suffering too large a slippage, the caller of `addLiquidity` can cap the amount of ERC20 token deducted from their account via the `max_tokens` argument. (All of this obviously doesn't apply to a newly created Exchange that has no liquidity yet. "_For the first liquidity provider, max_tokens is the exact amount of tokens deposited._")

```admonish danger
`min_liquidity` is somewhat obscure. The documentation for V1 states, _"_`min_liquidity` _is used in combination with_ `max_tokens` _and_ `ethAmount` _to bound the rate at which liquidity tokens are minted. For the first liquidity provider, min_liquidity does not do anything and can be set to 0."_

This descriptions makes absolute sense for the first liquidity provider, but it doesn't make a whole lot of sense for the ones that follow. Why on Earth would you want to bound the rate at which liquidity tokens are minted when the LP Tokens generation is deterministic?

What it does look like is like another mechanism to control slippage, where along with tracking the price fluctuations you track fluctuations of your potential share in a liquidity pool and you don't commit to provide liquidity unless you get a certain share. Knowledgeable folks, correct me if this is inaccurate.
```





`deadline` sets a time after which a transaction can no longer be executed. Per V1 documentation, "_This limits the "free option" problem, where Ethereum miners can hold signed transactions and execute them based off market movements_". 




Note the type of `deadline`: `timestamp`. Another strong typing artifact which didn't survive to our days. 

`min_liquidity`
: uint256, max_tokens: uint256

`Return Value`: The amount of LP Token minted.

The ethAmount sent to addLiquidity is the exact amount of ETH that will be deposited into the liquidity reserves. It should be 50% of the total value a liquidity provider wishes to deposit into the reserves.

Since liquidity providers must deposit at the current exchange rate, the Uniswap smart contracts use ethAmount to determine the amount of ERC20 tokens that must be deposited. This token amount is the remaining 50% of total value a liquidity provider wishes to deposit. Since exchange rate can change between when a transaction is signed and when it is executed on Ethereum, max_tokens is used to bound the amount this rate can fluctuate. For the first liquidity provider, max_tokens is the exact amount of tokens deposited.


### General Logic

Second, it will always attempt to do three main things (in addition to several minor ones):
1. Ensure that the deadline for adding liquidity set by the caller hasn't passed
2. Update ETH and ERC20 token reserves
3. Mint LP Token to the liquidity provider based on their share of liquidity in the pool

As stated earlier, there are two scenarios for adding liquidity:
1. The Exchange has no liquidity yet; and
2. The Exchange has liquidity.







```python
# @dev Burn UNI tokens to withdraw ETH and Tokens at current ratio.
# @param amount Amount of UNI burned.
# @param min_eth Minimum ETH withdrawn.
# @param min_tokens Minimum Tokens withdrawn.
# @param deadline Time after which this transaction can no longer be executed.
# @return The amount of ETH and Tokens withdrawn.
@public
def removeLiquidity(amount: uint256, min_eth: uint256(wei), min_tokens: uint256, deadline: timestamp) -> (uint256(wei), uint256):
    assert (amount > 0 and deadline > block.timestamp) and (min_eth > 0 and min_tokens > 0)
    total_liquidity: uint256 = self.totalSupply
    assert total_liquidity > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    eth_amount: uint256(wei) = amount * self.balance / total_liquidity
    token_amount: uint256 = amount * token_reserve / total_liquidity
    assert eth_amount >= min_eth and token_amount >= min_tokens
    self.balances[msg.sender] -= amount
    self.totalSupply = total_liquidity - amount
    send(msg.sender, eth_amount)
    assert self.token.transfer(msg.sender, token_amount)
    log.RemoveLiquidity(msg.sender, eth_amount, token_amount)
    log.Transfer(msg.sender, ZERO_ADDRESS, amount)
    return eth_amount, token_amount
```

```python
# @dev Pricing function for converting between ETH and Tokens.
# @param input_amount Amount of ETH or Tokens being sold.
# @param input_reserve Amount of ETH or Tokens (input type) in exchange reserves.
# @param output_reserve Amount of ETH or Tokens (output type) in exchange reserves.
# @return Amount of ETH or Tokens bought.
@private
@constant
def getInputPrice(input_amount: uint256, input_reserve: uint256, output_reserve: uint256) -> uint256:
    assert input_reserve > 0 and output_reserve > 0
    input_amount_with_fee: uint256 = input_amount * 997
    numerator: uint256 = input_amount_with_fee * output_reserve
    denominator: uint256 = (input_reserve * 1000) + input_amount_with_fee
    return numerator / denominator
```

```python
# @dev Pricing function for converting between ETH and Tokens.
# @param output_amount Amount of ETH or Tokens being bought.
# @param input_reserve Amount of ETH or Tokens (input type) in exchange reserves.
# @param output_reserve Amount of ETH or Tokens (output type) in exchange reserves.
# @return Amount of ETH or Tokens sold.
@private
@constant
def getOutputPrice(output_amount: uint256, input_reserve: uint256, output_reserve: uint256) -> uint256:
    assert input_reserve > 0 and output_reserve > 0
    numerator: uint256 = input_reserve * output_amount * 1000
    denominator: uint256 = (output_reserve - output_amount) * 997
    return numerator / denominator + 1
```

```python
@private
def ethToTokenInput(eth_sold: uint256(wei), min_tokens: uint256, deadline: timestamp, buyer: address, recipient: address) -> uint256:
    assert deadline >= block.timestamp and (eth_sold > 0 and min_tokens > 0)
    token_reserve: uint256 = self.token.balanceOf(self)
    tokens_bought: uint256 = self.getInputPrice(as_unitless_number(eth_sold), as_unitless_number(self.balance - eth_sold), token_reserve)
    assert tokens_bought >= min_tokens
    assert self.token.transfer(recipient, tokens_bought)
    log.TokenPurchase(buyer, eth_sold, tokens_bought)
    return tokens_bought
```

```python
# @notice Convert ETH to Tokens.
# @dev User specifies exact input (msg.value).
# @dev User cannot specify minimum output or deadline.
@public
@payable
def __default__():
    self.ethToTokenInput(msg.value, 1, block.timestamp, msg.sender, msg.sender)
```

```python
# @notice Convert ETH to Tokens.
# @dev User specifies exact input (msg.value) and minimum output.
# @param min_tokens Minimum Tokens bought.
# @param deadline Time after which this transaction can no longer be executed.
# @return Amount of Tokens bought.
@public
@payable
def ethToTokenSwapInput(min_tokens: uint256, deadline: timestamp) -> uint256:
    return self.ethToTokenInput(msg.value, min_tokens, deadline, msg.sender, msg.sender)
```

```python
# @notice Convert ETH to Tokens and transfers Tokens to recipient.
# @dev User specifies exact input (msg.value) and minimum output
# @param min_tokens Minimum Tokens bought.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output Tokens.
# @return Amount of Tokens bought.
@public
@payable
def ethToTokenTransferInput(min_tokens: uint256, deadline: timestamp, recipient: address) -> uint256:
    assert recipient != self and recipient != ZERO_ADDRESS
    return self.ethToTokenInput(msg.value, min_tokens, deadline, msg.sender, recipient)
```

```python
@private
def ethToTokenOutput(tokens_bought: uint256, max_eth: uint256(wei), deadline: timestamp, buyer: address, recipient: address) -> uint256(wei):
    assert deadline >= block.timestamp and (tokens_bought > 0 and max_eth > 0)
    token_reserve: uint256 = self.token.balanceOf(self)
    eth_sold: uint256 = self.getOutputPrice(tokens_bought, as_unitless_number(self.balance - max_eth), token_reserve)
    # Throws if eth_sold > max_eth
    eth_refund: uint256(wei) = max_eth - as_wei_value(eth_sold, 'wei')
    if eth_refund > 0:
        send(buyer, eth_refund)
    assert self.token.transfer(recipient, tokens_bought)
    log.TokenPurchase(buyer, as_wei_value(eth_sold, 'wei'), tokens_bought)
    return as_wei_value(eth_sold, 'wei')
```

```python
# @notice Convert ETH to Tokens.
# @dev User specifies maximum input (msg.value) and exact output.
# @param tokens_bought Amount of tokens bought.
# @param deadline Time after which this transaction can no longer be executed.
# @return Amount of ETH sold.
@public
@payable
def ethToTokenSwapOutput(tokens_bought: uint256, deadline: timestamp) -> uint256(wei):
    return self.ethToTokenOutput(tokens_bought, msg.value, deadline, msg.sender, msg.sender)
```

```python
# @notice Convert ETH to Tokens and transfers Tokens to recipient.
# @dev User specifies maximum input (msg.value) and exact output.
# @param tokens_bought Amount of tokens bought.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output Tokens.
# @return Amount of ETH sold.
@public
@payable
def ethToTokenTransferOutput(tokens_bought: uint256, deadline: timestamp, recipient: address) -> uint256(wei):
    assert recipient != self and recipient != ZERO_ADDRESS
    return self.ethToTokenOutput(tokens_bought, msg.value, deadline, msg.sender, recipient)
```

```python
@private
def tokenToEthInput(tokens_sold: uint256, min_eth: uint256(wei), deadline: timestamp, buyer: address, recipient: address) -> uint256(wei):
    assert deadline >= block.timestamp and (tokens_sold > 0 and min_eth > 0)
    token_reserve: uint256 = self.token.balanceOf(self)
    eth_bought: uint256 = self.getInputPrice(tokens_sold, token_reserve, as_unitless_number(self.balance))
    wei_bought: uint256(wei) = as_wei_value(eth_bought, 'wei')
    assert wei_bought >= min_eth
    send(recipient, wei_bought)
    assert self.token.transferFrom(buyer, self, tokens_sold)
    log.EthPurchase(buyer, tokens_sold, wei_bought)
    return wei_bought
```

```python
# @notice Convert Tokens to ETH.
# @dev User specifies exact input and minimum output.
# @param tokens_sold Amount of Tokens sold.
# @param min_eth Minimum ETH purchased.
# @param deadline Time after which this transaction can no longer be executed.
# @return Amount of ETH bought.
@public
def tokenToEthSwapInput(tokens_sold: uint256, min_eth: uint256(wei), deadline: timestamp) -> uint256(wei):
    return self.tokenToEthInput(tokens_sold, min_eth, deadline, msg.sender, msg.sender)
```

```python
# @notice Convert Tokens to ETH and transfers ETH to recipient.
# @dev User specifies exact input and minimum output.
# @param tokens_sold Amount of Tokens sold.
# @param min_eth Minimum ETH purchased.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output ETH.
# @return Amount of ETH bought.
@public
def tokenToEthTransferInput(tokens_sold: uint256, min_eth: uint256(wei), deadline: timestamp, recipient: address) -> uint256(wei):
    assert recipient != self and recipient != ZERO_ADDRESS
    return self.tokenToEthInput(tokens_sold, min_eth, deadline, msg.sender, recipient)
```

```python
@private
def tokenToEthOutput(eth_bought: uint256(wei), max_tokens: uint256, deadline: timestamp, buyer: address, recipient: address) -> uint256:
    assert deadline >= block.timestamp and eth_bought > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    tokens_sold: uint256 = self.getOutputPrice(as_unitless_number(eth_bought), token_reserve, as_unitless_number(self.balance))
    # tokens sold is always > 0
    assert max_tokens >= tokens_sold
    send(recipient, eth_bought)
    assert self.token.transferFrom(buyer, self, tokens_sold)
    log.EthPurchase(buyer, tokens_sold, eth_bought)
    return tokens_sold
```

```python
# @notice Convert Tokens to ETH.
# @dev User specifies maximum input and exact output.
# @param eth_bought Amount of ETH purchased.
# @param max_tokens Maximum Tokens sold.
# @param deadline Time after which this transaction can no longer be executed.
# @return Amount of Tokens sold.
@public
def tokenToEthSwapOutput(eth_bought: uint256(wei), max_tokens: uint256, deadline: timestamp) -> uint256:
    return self.tokenToEthOutput(eth_bought, max_tokens, deadline, msg.sender, msg.sender)
```

```python
# @notice Convert Tokens to ETH and transfers ETH to recipient.
# @dev User specifies maximum input and exact output.
# @param eth_bought Amount of ETH purchased.
# @param max_tokens Maximum Tokens sold.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output ETH.
# @return Amount of Tokens sold.
@public
def tokenToEthTransferOutput(eth_bought: uint256(wei), max_tokens: uint256, deadline: timestamp, recipient: address) -> uint256:
    assert recipient != self and recipient != ZERO_ADDRESS
    return self.tokenToEthOutput(eth_bought, max_tokens, deadline, msg.sender, recipient)
```

```python
@private
def tokenToTokenInput(tokens_sold: uint256, min_tokens_bought: uint256, min_eth_bought: uint256(wei), deadline: timestamp, buyer: address, recipient: address, exchange_addr: address) -> uint256:
    assert (deadline >= block.timestamp and tokens_sold > 0) and (min_tokens_bought > 0 and min_eth_bought > 0)
    assert exchange_addr != self and exchange_addr != ZERO_ADDRESS
    token_reserve: uint256 = self.token.balanceOf(self)
    eth_bought: uint256 = self.getInputPrice(tokens_sold, token_reserve, as_unitless_number(self.balance))
    wei_bought: uint256(wei) = as_wei_value(eth_bought, 'wei')
    assert wei_bought >= min_eth_bought
    assert self.token.transferFrom(buyer, self, tokens_sold)
    tokens_bought: uint256 = Exchange(exchange_addr).ethToTokenTransferInput(min_tokens_bought, deadline, recipient, value=wei_bought)
    log.EthPurchase(buyer, tokens_sold, wei_bought)
    return tokens_bought
```

```python
# @notice Convert Tokens (self.token) to Tokens (token_addr).
# @dev User specifies exact input and minimum output.
# @param tokens_sold Amount of Tokens sold.
# @param min_tokens_bought Minimum Tokens (token_addr) purchased.
# @param min_eth_bought Minimum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param token_addr The address of the token being purchased.
# @return Amount of Tokens (token_addr) bought.
@public
def tokenToTokenSwapInput(tokens_sold: uint256, min_tokens_bought: uint256, min_eth_bought: uint256(wei), deadline: timestamp, token_addr: address) -> uint256:
    exchange_addr: address = self.factory.getExchange(token_addr)
    return self.tokenToTokenInput(tokens_sold, min_tokens_bought, min_eth_bought, deadline, msg.sender, msg.sender, exchange_addr)
```

```python
# @notice Convert Tokens (self.token) to Tokens (token_addr) and transfers
#         Tokens (token_addr) to recipient.
# @dev User specifies exact input and minimum output.
# @param tokens_sold Amount of Tokens sold.
# @param min_tokens_bought Minimum Tokens (token_addr) purchased.
# @param min_eth_bought Minimum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output ETH.
# @param token_addr The address of the token being purchased.
# @return Amount of Tokens (token_addr) bought.
@public
def tokenToTokenTransferInput(tokens_sold: uint256, min_tokens_bought: uint256, min_eth_bought: uint256(wei), deadline: timestamp, recipient: address, token_addr: address) -> uint256:
    exchange_addr: address = self.factory.getExchange(token_addr)
    return self.tokenToTokenInput(tokens_sold, min_tokens_bought, min_eth_bought, deadline, msg.sender, recipient, exchange_addr)
```

```python
@private
def tokenToTokenOutput(tokens_bought: uint256, max_tokens_sold: uint256, max_eth_sold: uint256(wei), deadline: timestamp, buyer: address, recipient: address, exchange_addr: address) -> uint256:
    assert deadline >= block.timestamp and (tokens_bought > 0 and max_eth_sold > 0)
    assert exchange_addr != self and exchange_addr != ZERO_ADDRESS
    eth_bought: uint256(wei) = Exchange(exchange_addr).getEthToTokenOutputPrice(tokens_bought)
    token_reserve: uint256 = self.token.balanceOf(self)
    tokens_sold: uint256 = self.getOutputPrice(as_unitless_number(eth_bought), token_reserve, as_unitless_number(self.balance))
    # tokens sold is always > 0
    assert max_tokens_sold >= tokens_sold and max_eth_sold >= eth_bought
    assert self.token.transferFrom(buyer, self, tokens_sold)
    eth_sold: uint256(wei) = Exchange(exchange_addr).ethToTokenTransferOutput(tokens_bought, deadline, recipient, value=eth_bought)
    log.EthPurchase(buyer, tokens_sold, eth_bought)
    return tokens_sold
```

```python
# @notice Convert Tokens (self.token) to Tokens (token_addr).
# @dev User specifies maximum input and exact output.
# @param tokens_bought Amount of Tokens (token_addr) bought.
# @param max_tokens_sold Maximum Tokens (self.token) sold.
# @param max_eth_sold Maximum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param token_addr The address of the token being purchased.
# @return Amount of Tokens (self.token) sold.
@public
def tokenToTokenSwapOutput(tokens_bought: uint256, max_tokens_sold: uint256, max_eth_sold: uint256(wei), deadline: timestamp, token_addr: address) -> uint256:
    exchange_addr: address = self.factory.getExchange(token_addr)
    return self.tokenToTokenOutput(tokens_bought, max_tokens_sold, max_eth_sold, deadline, msg.sender, msg.sender, exchange_addr)
```

```python
# @notice Convert Tokens (self.token) to Tokens (token_addr) and transfers
#         Tokens (token_addr) to recipient.
# @dev User specifies maximum input and exact output.
# @param tokens_bought Amount of Tokens (token_addr) bought.
# @param max_tokens_sold Maximum Tokens (self.token) sold.
# @param max_eth_sold Maximum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output ETH.
# @param token_addr The address of the token being purchased.
# @return Amount of Tokens (self.token) sold.
@public
def tokenToTokenTransferOutput(tokens_bought: uint256, max_tokens_sold: uint256, max_eth_sold: uint256(wei), deadline: timestamp, recipient: address, token_addr: address) -> uint256:
    exchange_addr: address = self.factory.getExchange(token_addr)
    return self.tokenToTokenOutput(tokens_bought, max_tokens_sold, max_eth_sold, deadline, msg.sender, recipient, exchange_addr)
```

```python
# @notice Convert Tokens (self.token) to Tokens (exchange_addr.token).
# @dev Allows trades through contracts that were not deployed from the same factory.
# @dev User specifies exact input and minimum output.
# @param tokens_sold Amount of Tokens sold.
# @param min_tokens_bought Minimum Tokens (token_addr) purchased.
# @param min_eth_bought Minimum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param exchange_addr The address of the exchange for the token being purchased.
# @return Amount of Tokens (exchange_addr.token) bought.
@public
def tokenToExchangeSwapInput(tokens_sold: uint256, min_tokens_bought: uint256, min_eth_bought: uint256(wei), deadline: timestamp, exchange_addr: address) -> uint256:
    return self.tokenToTokenInput(tokens_sold, min_tokens_bought, min_eth_bought, deadline, msg.sender, msg.sender, exchange_addr)
```

```python
# @notice Convert Tokens (self.token) to Tokens (exchange_addr.token) and transfers
#         Tokens (exchange_addr.token) to recipient.
# @dev Allows trades through contracts that were not deployed from the same factory.
# @dev User specifies exact input and minimum output.
# @param tokens_sold Amount of Tokens sold.
# @param min_tokens_bought Minimum Tokens (token_addr) purchased.
# @param min_eth_bought Minimum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output ETH.
# @param exchange_addr The address of the exchange for the token being purchased.
# @return Amount of Tokens (exchange_addr.token) bought.
@public
def tokenToExchangeTransferInput(tokens_sold: uint256, min_tokens_bought: uint256, min_eth_bought: uint256(wei), deadline: timestamp, recipient: address, exchange_addr: address) -> uint256:
    assert recipient != self
    return self.tokenToTokenInput(tokens_sold, min_tokens_bought, min_eth_bought, deadline, msg.sender, recipient, exchange_addr)
```

```python
# @notice Convert Tokens (self.token) to Tokens (exchange_addr.token).
# @dev Allows trades through contracts that were not deployed from the same factory.
# @dev User specifies maximum input and exact output.
# @param tokens_bought Amount of Tokens (token_addr) bought.
# @param max_tokens_sold Maximum Tokens (self.token) sold.
# @param max_eth_sold Maximum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param exchange_addr The address of the exchange for the token being purchased.
# @return Amount of Tokens (self.token) sold.
@public
def tokenToExchangeSwapOutput(tokens_bought: uint256, max_tokens_sold: uint256, max_eth_sold: uint256(wei), deadline: timestamp, exchange_addr: address) -> uint256:
    return self.tokenToTokenOutput(tokens_bought, max_tokens_sold, max_eth_sold, deadline, msg.sender, msg.sender, exchange_addr)
```

```python
# @notice Convert Tokens (self.token) to Tokens (exchange_addr.token) and transfers
#         Tokens (exchange_addr.token) to recipient.
# @dev Allows trades through contracts that were not deployed from the same factory.
# @dev User specifies maximum input and exact output.
# @param tokens_bought Amount of Tokens (token_addr) bought.
# @param max_tokens_sold Maximum Tokens (self.token) sold.
# @param max_eth_sold Maximum ETH purchased as intermediary.
# @param deadline Time after which this transaction can no longer be executed.
# @param recipient The address that receives output ETH.
# @param token_addr The address of the token being purchased.
# @return Amount of Tokens (self.token) sold.
@public
def tokenToExchangeTransferOutput(tokens_bought: uint256, max_tokens_sold: uint256, max_eth_sold: uint256(wei), deadline: timestamp, recipient: address, exchange_addr: address) -> uint256:
    assert recipient != self
    return self.tokenToTokenOutput(tokens_bought, max_tokens_sold, max_eth_sold, deadline, msg.sender, recipient, exchange_addr)
```

```python
# @notice Public price function for ETH to Token trades with an exact input.
# @param eth_sold Amount of ETH sold.
# @return Amount of Tokens that can be bought with input ETH.
@public
@constant
def getEthToTokenInputPrice(eth_sold: uint256(wei)) -> uint256:
    assert eth_sold > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    return self.getInputPrice(as_unitless_number(eth_sold), as_unitless_number(self.balance), token_reserve)
```

```python
# @notice Public price function for ETH to Token trades with an exact output.
# @param tokens_bought Amount of Tokens bought.
# @return Amount of ETH needed to buy output Tokens.
@public
@constant
def getEthToTokenOutputPrice(tokens_bought: uint256) -> uint256(wei):
    assert tokens_bought > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    eth_sold: uint256 = self.getOutputPrice(tokens_bought, as_unitless_number(self.balance), token_reserve)
    return as_wei_value(eth_sold, 'wei')
```

```python
# @notice Public price function for Token to ETH trades with an exact input.
# @param tokens_sold Amount of Tokens sold.
# @return Amount of ETH that can be bought with input Tokens.
@public
@constant
def getTokenToEthInputPrice(tokens_sold: uint256) -> uint256(wei):
    assert tokens_sold > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    eth_bought: uint256 = self.getInputPrice(tokens_sold, token_reserve, as_unitless_number(self.balance))
    return as_wei_value(eth_bought, 'wei')
```

```python
# @notice Public price function for Token to ETH trades with an exact output.
# @param eth_bought Amount of output ETH.
# @return Amount of Tokens needed to buy output ETH.
@public
@constant
def getTokenToEthOutputPrice(eth_bought: uint256(wei)) -> uint256:
    assert eth_bought > 0
    token_reserve: uint256 = self.token.balanceOf(self)
    return self.getOutputPrice(as_unitless_number(eth_bought), token_reserve, as_unitless_number(self.balance))
```

```python
# @return Address of Token that is sold on this exchange.
@public
@constant
def tokenAddress() -> address:
    return self.token
```

```python
# @return Address of factory that created this exchange.
@public
@constant
def factoryAddress() -> address(Factory):
    return self.factory
```

```python
# ERC20 compatibility for exchange liquidity modified from
# https://github.com/ethereum/vyper/blob/master/examples/tokens/ERC20.vy
@public
@constant
def balanceOf(_owner : address) -> uint256:
    return self.balances[_owner]
```

```python
@public
def transfer(_to : address, _value : uint256) -> bool:
    self.balances[msg.sender] -= _value
    self.balances[_to] += _value
    log.Transfer(msg.sender, _to, _value)
    return True
```

```python
@public
def transferFrom(_from : address, _to : address, _value : uint256) -> bool:
    self.balances[_from] -= _value
    self.balances[_to] += _value
    self.allowances[_from][msg.sender] -= _value
    log.Transfer(_from, _to, _value)
    return True
```

```python
@public
def approve(_spender : address, _value : uint256) -> bool:
    self.allowances[msg.sender][_spender] = _value
    log.Approval(msg.sender, _spender, _value)
    return True
```

```python
@public
@constant
def allowance(_owner : address, _spender : address) -> uint256:
    return self.allowances[_owner][_spender]
```

[^erc20]: <https://eips.ethereum.org/EIPS/eip-20>
[^unittypes] [Vyper: Unit Types](https://vyper.readthedocs.io/en/v0.1.0-beta.4/types.html#unit-types)
[^strings] [Vyper: Strings](https://vyper.readthedocs.io/en/v0.1.0-beta.4/types.html#strings)