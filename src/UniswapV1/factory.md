# Factory

Uniswap v1 Factory ([Github](https://github.com/Uniswap/v1-contracts/blob/master/contracts/uniswap_factory.vy))


```python
contract Exchange():
    def setup(token_addr: address): modifying
```

Prior to Vyper v0.2.1, which was a major breaking release ([v0.2.1 Release Notes](https://vyper.readthedocs.io/en/stable/release-notes.html#v0-2-1)), Vyper contracts used the keyword `contract` to declare interfaces for interacting with other contracts, instead of the now more familiar `interface`.

Factory uses the `setup` function of each deployed Exchange contract to configure the ERC20 asset. `setup` takes a single argument, which is the address of the ERC20 token. `setup` doesn't return any values, and it will modify the state of the Exchange instance.

```python
NewExchange: event({token: indexed(address), exchange: indexed(address)})
```

Factory emits the `NewExchange` event every time an Exchange is created; this is consistent with the practice of informing off-chain dApp counterparts about important state changes of the on-chain system. `NewExchange` records two pieces of information: the address of the ERC20 token in the pair (`token`) and the address of the Exchange contract deployed for the pair (`exchange`.)


```python
exchangeTemplate: public(address)
tokenCount: public(uint256)
token_to_exchange: address[address]
exchange_to_token: address[address]
id_to_token: address[uint256]
```

State variables are declared next. 

`exchangeTemplate` is a public variable that stores the address of the Exchange contract deployed during Factory initialization. This "on-chain template" is used by Factory to deploy Exchanges at run-time by deploying a minimal proxy using the code of the contract already deployed for its logic.

`tokenCount` is a public variable the stores the number of ERC20 tokens for which Exchanges have been deployed. 

`token_to_exchange` is a mapping to resolve ERC20 token addresses to Exchange addresses.

`exchange_to_token` is a mapping to resolve Exchange addresses to ERC20 token addresses.

`id_to_token` is a mapping to resolve ERC20 token identities (serial numbers as defined by the `tokenCount` counter) to ERC20 token addresses.

```python
@public
def initializeFactory(template: address):
    assert self.exchangeTemplate == ZERO_ADDRESS
    assert template != ZERO_ADDRESS
    self.exchangeTemplate = template
```

The initializer function is a once-only affair and it is expected to be run immediately after Factory is deployed. It takes the address of the Exchange template, which means the Exchange contract is already deployed at this point.

The two `assert` statements check that 
1. `exchangeTemplate` has not been initialized (thus preventing a second call to `initializeFactory`)
2. `template` is a non-zero address 

If both conditions hold, `self.exchangeTemplate = template` sets Factory's `exchangeTemplate` to the address of the deployed Exchange contract.


```python
@public
def createExchange(token: address) -> address:
    assert token != ZERO_ADDRESS
    assert self.exchangeTemplate != ZERO_ADDRESS
    assert self.token_to_exchange[token] == ZERO_ADDRESS
    exchange: address = create_with_code_of(self.exchangeTemplate)
    Exchange(exchange).setup(token)
    self.token_to_exchange[token] = exchange
    self.exchange_to_token[exchange] = token
    token_id: uint256 = self.tokenCount + 1
    self.tokenCount = token_id
    self.id_to_token[token_id] = token
    log.NewExchange(token, exchange)
    return exchange
```

`createExchange` takes the address of an ERC20 token as the argument and returns the address of the Exchange proxy which functions as the ETH/ERC20 Exchange for that token.

The three `assert` statements check that:
1. The `token` address is not zero
2. The template has been initialized
3. The Exchange for this particular ERC20 has not yet been created (see [Overview]()./index.md)

`exchange: address = create_with_code_of(self.exchangeTemplate)` deploys a [minimal proxy contract](https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/) pointing to the deployed Exchange contract. The name `create_with_code_of` was a misleading one; it didn't actually copy the code of a deployed contract. For this reason, the call was later renamed by the Vyper team to [`create_forwarder_to`](https://vyper.readthedocs.io/en/stable/built-in-functions.html?highlight=create_minimal_proxy#chain-interaction).

`Exchange(exchange).setup(token)` initializes the proxy state by calling the Exchange's `setup` function and passing the ERC20 token address as the argument. As the result of this approach, there will always only be a single Exchange deployment, which is only used for its logic, and multiple proxies, each holding state of an ETH/ERC20 pair.

Note that this is a common pattern in EVM dev: a contract's constructor will only be called once, on deployment, so proxies use initializer functions to emulate the work of a constructor. See [Exchange](./exchange.md) for the details.

`self.token_to_exchange[token] = exchange` stores the mapping from the ERC20 token address to the address of the Exchange proxy.

`self.exchange_to_token[exchange] = token` stores the mapping from the Exchange proxy address to the ERC20 token address. 

`token_id: uint256 = self.tokenCount + 1` stores the new token count in a memory variable, `token_id`. 

`self.tokenCount = token_id` replicates the update to storage variable `tokenCount`. The reason why storage wasn't updated immediately with a statement like `self.tokenCount = self.tokenCount + 1` becomes clear right away:

`self.id_to_token[token_id] = token`, for here we reuse the in-memory variable, avoiding reading from storage. Reading from storage (at the time of the writing) costs 100 gas, and reading from memory costs 3. (Verify this statement): Even with the overhead of setting up an in-memory variable, this saves gas.

`log.NewExchange(token, exchange)` emits a NewExchange event, logging the ERC20 token address and the Exchange proxy address.
    
`return exchange`
Finally, per function declaration, we return the address of the Exchange proxy to the caller. Note that because we both emit an event and return the address on-chain, we've covered all potential use cases where the caller might want to know the address after the transaction has been executed. We can get the address of the newly deployed Exchange proxy both off-chain and when calling `createExchange` from another smart contract.

Finally, Factory exposes three view functions:

```python
@public
@constant
def getExchange(token: address) -> address:
    return self.token_to_exchange[token]
```

Returns the Exchange proxy address based on the ERC20 token address. Will return the zero address if no such pairing exists.


```python
@public
@constant
def getToken(exchange: address) -> address:
    return self.exchange_to_token[exchange]
```

Returns the ERC20 token address based on the Exchange proxy address. Will return the zero address if no such pairing exists.

```python
@public
@constant
def getTokenWithId(token_id: uint256) -> address:
    return self.id_to_token[token_id]
```

Returns the ERC20 token address based on its identity (serial number).