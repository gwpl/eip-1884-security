# Security considerations for EIP-1884


## Background

[EIP 1884](https://eips.ethereum.org/EIPS/eip-1884) is set to be implemented into the upcoming Ethereum 'Istanbul' hard fork. It 

- increases the cost of opcode `SLOAD` from `200` to `800` gas
- increases the cost of `BALANCE` and `EXTCODEHASH` from `400` to `700` gas
- adds a new opcode `SELFBALANCE` with cost `5`. 

The reasoning is that due to the increase in state size, and thus the added IO overhead for fetching tries from disk, the opcodes `SLOAD`, `BALANCE` and `EXTCODEHASH` have become disproportionally 'cheap', for the amount of work that a node has to perform. Having badly 'tuned' gas cost versus the underlying computational cost of an operation is a problem which can cause various problems, and pave the way for attacks such as the so called 'Shanghai attacks' as seen in late 2017. 

## Potential problems

In general, repricing opcodes can always break contracts that explicitly rely on assumptions of gas cost being constant. However, this has been considered bad practice for a long time, especially so since certain opcodes historically already _have_ been repriced in [Tangerine Whistle](https://eips.ethereum.org/EIPS/eip-150), where `SLOAD` was repriced from `50` to `200`. 

However, there is one case which could potentially become more problematic; `default` functions. 

### Default functions

A `default` function is a method of a contract that handles calls without any data -- they are there to handle transfers of ether that does not explicitly invoke any method at all. They are typically used to create an event using a `LOG` operation, so external systems can detect the event, and e.g. register that a transfer was made. 

A regular transfer of ether to a contract always gives the receiver at minimum `2300` as gas `stipend`. This number is meant to allow the recipient to issue an event, but is not sufficient to perform state changes (such as making another transfer, or updating a storage slot). 


### EIP 1884 and default functions

One potential problem of EIP-1884 is that `default` functions might start to fail on `2300` gas, e.g. for the following reasons:

* Limited wallets: A contract only allows payments if `balance(self)` is below a certain limit 
* Designated senders: A contract only allows payments from a set of pre-approved senders
* Disabled wallets: A contract only allows payments if a certain variable (slot) is set to `true`. 

Now, if a `default` function ceases to work with `2300` gas, this is not always a very serious problem. For example, if the caller is a so called `EOA` (Externally Owner Account - meaning end user), the caller can simply make sure to send a bit more than `21000` gas in the transaction. But the problem can arise if, for example

- The `target` has designated sender, 
- The `senders` are smart contracts, which are programmed to only ever use `transfer` with no extra gas. 

In that case, the flow of ether from the `senders` to the `target` would be broken in a way that is not 'fixable' unless other mechanisms can be used to handle the situation (e.g. replacing the senders). 

## Investigation

I reached out to the EthSecurity community to help assess this situation. Some notes:

- Contracts that does not have a `payable` default function would not be affected, 
- Contracts whose default function would not be executable today on `2300` gas would not be affected e.g. contracts that do SLOAD or transfer ether in `default` would already be 'broken'


Neville Grech, of [Contract Library](https://contract-library.com), performed an analysis of decompiled mainnet contracts. The analysis covers about 95% of all contracts on mainnet (500K unique bytecodes), and lists those that could potentially be affected. 
  - The list is available [here](https://contract-library.com/?w=FALLBACK_WILL_FAIL)


Hubert Ritzdorf, of [ChainSecurity](https://chainsecurity.com/), performed an analysis of recent transactions. The analysis is based on investigating actual transactions on mainnet, and seeing which of those would have failed if `SLOAD` had cost `800` instead of `200`. Partial results are [here](https://gist.github.com/ritzdorf/1c6bd72955391e831f8a397d3152b4e0). 

### Chain Security analysis

See [this gist](https://gist.github.com/ritzdorf/1c6bd72955391e831f8a397d3152b4e0), with the following comment:

> The first two occur very frequently, the others are less frequent. We listed the final one even though it would still work the EIP as we are not sure how these gas values are currently being determined for such "deep" transactions. We wanted to raise awareness of potential issues.

#### Kyber Network

```
    function() public payable {
        require(reserveType[msg.sender] != ReserveType.NONE);
        EtherReceival(msg.sender, msg.value);
    }
```

- KyberNetwork meets several of the criterias, 
 - Implements the "Designated senders" pattern, 
 - Called primarily through other contracts, which rely on `transfer` (this limited to `2300` gas)

We reached out to KyberNetwork, and although it is obviously a chore to do, this can be solved: 

 > technically the market maker can just deploy new reserve contract

#### CappedVault

```
    function total() public view returns(uint) {
        return getBalance() + withdrawn;
    }

    function () public payable {
        require(total() + msg.value <= limit);
    }
```
In this context, `withdrawn` is a storage `slot`, and so is `limit`. 

- CappedVault, with over `4K ether` and `70K` internal transactions, meet the criteria:
  - Implements the "Limited" pattern
  - Two `SLOAD` and one `BALANCE`

Implementation note: 

- This contract is programmed to 'break' exactly like this, in case the total of ether passed through the contract exceeds `33333 ether`. That is, regardless of how much `ether` is currently in the vault, it will cease to accept `ether` after `33K` has passed through it. 
  - **This indicates that there already _must_ be mechanisms to handle the case when `default` cease functioning. **
- The `limit` is a storage `slot`, but could have been implemented as a compile-time constant, reducing one `SLOAD`. 
- The `balance(self)` could, after Istanbul, be rewritten as `SELFBALANCE`

In essence, it currently uses:

 `200 (sload limit) +200 (sload withdrawn) +400 (balance) = 800 gas` 

into, post-EIP-1884: 

`5 (selfbalance) + 800 (sload withdrawn) = 805 gas`. 


