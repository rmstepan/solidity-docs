# Gas optimization

Solidity cheatsheet is available in [solidity-cheatsheet](/README.md).

## Table of contents

- [Gas optimization](#gas-optimization)
  * [Table of contents](#table-of-contents)
  * [Motivation](#motivation)
  * [Optimizing variables](#optimizing-variables)
    + [Variable packing](#variable-packing)
    + [Data location](#data-location)
    + [Reference data types](#reference-data-types)
    + [Memory vs. Storage](#memory-vs-storage)
## Motivation

Gas optimization is a challenge that is unique to developing Ethereum smart contracts. Can be viewed as being similar to optimizing programs on memory-limited systems, altough the goal here is to reduce execution costs.

## Optimizing variables
### Variable packing
Solidity contracts have contiguous 32 byte (256 bit) slots used for storage. When we arrange variables so multiple fit in a single slot, it is called variable packing.
Let’s look at an example:
```solidity
uint128 a;
uint256 b;
uint128 c;
```
These variables are not packed. If `b` was packed with `a`, it would exceed the 32 byte limit so it is instead placed in a new storage slot. The same thing happens with `c` and `b`.
```solidity
uint128 a;
uint128 c;
uint256 b;
```
These variables are packed. Because packing `c` with `a` does not exceed the 32 byte limit, they are stored in the same slot.

Keep variable packing in mind when choosing data types — a smaller version of a data type is only useful if it helps pack the variable in a storage slot. If a uint128 does not pack, we might as well use a uint256.

### Data location
Variable packing only occurs in storage — memory and call data does not get packed. You will not save space trying to pack function arguments or local variables.

### Reference data types
Structs and arrays always begin in a new storage slot — however their contents can be packed normally. A `uint8` array will take up less space than an equal length `uint256` array.

It is more gas efficient to initialize a tightly packed struct with separate assignments instead of a single assignment. 
Separate assignments makes it easier for the optimizer to update all the variables at once.

Initialize structs like this:
```solidity
Point storage p = Point()
p.x = 0;
p.y = 0;
```
Instead of:
```solidity
Point storage p = Point(0, 0);
```
### Memory vs Storage
Performing operations on memory — or call data, which is similar to memory — is always cheaper than storage.

A common way to reduce the number of storage operations is manipulating a local memory variable before assigning it to a storage variable.

We see this often in loops:
```solidity
uint256 return = 5; // assume 2 decimal places
uint256 totalReturn;
function updateTotalReturn(uint256 timesteps) external {
    uint256 r = totalReturn || 1;
    for (uint256 i = 0; i < timesteps; i++) {
        r = r * return;
    }
    totalReturn = r;
}
```
In `calculateReturn`, we use the local memory variable `r` to store intermediate values and assign the final value to our storage variable `totalReturn`.
