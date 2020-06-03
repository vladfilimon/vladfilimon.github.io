---
layout: post
title: ERC20 Short Address / Parameter Attack
---
> On: Security, Solidity, Smart Contracts, EVM

This underflow attack is a very interesting one as it relies on a couple of things, like lack of input validation (offchain), Ethereum not ~~implementing~~ enforcing checksum on addresses, and finally the way the EVM adds 0 for a missing byte, packs together the method call and the arguments.
To my understanding, this type of attack has never been observed in the wild. It is important to note that smart contracts cannot be attacked in this manner directly, but through 3rd party (offchain) software that interacts with the smart contracts.

A transitional [checksum system] (https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md) has eventually been developed for Ethereum, though it's not somemthing at protocol level, leaving it up to the wallets and exchanges if they choose to implement it or not.
However, it is different from the one most other blockchains use (starting with Bitcoin) which is to encode the string in base 58 with an added version number and checksum, as Ethereum's EIP-55 would only capitalize some characters in the string:
> Convert the address to hex, but if the ith digit is a letter (ie. it's one of abcdef) print it in uppercase if the 4*ith bit of the hash of the lowercase hexadecimal address is 1 otherwise print it in lowercase.

There is a very interesting read regarding the checksums and addesses [here](https://ethereum.stackexchange.com/questions/267/why-dont-ethereum-addresses-have-checksums/274#274)

In a contract-invocation transaction the function hash signature, alongside the arguments, are orderly encoded in the input field. The first four bytes specify callee function and the rest of the arguments are arranged in chunks of 32 bytes.
If the lengt of the encoded arguments happens to be less than expected, EVM will auto-pad extra zeros to the arguments untill the correct lenghth of 32 bytes is reached.
## Few words about calling a contract
 The input data payload for calling a contract is a hex-serialized string consisting of:
- First 4 bytes represent the method signature (Keccak hash of the function's prototype together with the argument types (as Solidity supports [function overloading](https://solidity.readthedocs.io/en/v0.5.10/contracts.html?highlight=function%20overloading#function-overloading). For the ERC20 token interface, the signature of the `transfer(address, uint256)` would be:
```
> web3.sha3("transfer(address,uint256)");  
"0xa9059cbb2ab09eb219583f4a59a5d0623ade346d962bcd4e46b11da047c9049b"  
```

   Thus the first 4 bytes `0xa9059cbb`, representing the signature, would tell EVM which method to invoke.
- Each argument supplied as a 32 bytes padded with zeros.

   Thus, for calling the transfer method in order to transfer an amount of 1 to an address 0x881f83D5317a12903472b89ccc54475e2a682d00, the input data payload should actually be:
```
a9059cbb (function selector) +
000000000000000000000000881f83D5317a12903472b89ccc54475e2a682d00 (first argument) +
0000000000000000000000000000000000000000000000008ac7230489e80000 (second argument)
```

## Input data payload
Imagine calling a method on a contract would like (newlines added for clarity):
``
   0x90b98a11  
   000000000000000000000000881f83D5317a12903472b89ccc54475e2a682d00
   0000000000000000000000000000000000000000000000000000000000000001
``
Where:
- 0x90b98a11 (first 4 bytes) is the method signature 
- 000000000000000000000000881f83D5317a12903472b89ccc54475e2a682d00 is the address (20 bytes) padded to 32 bytes
- 0000000000000000000000000000000000000000000000000000000000000001 represents the amount, exactly 1 WEI, unsigned integer padded to 32 bytes
## Causing an underflow
Removing the last byte of the address (00) would cause an underflow, resulting in the input data looking like:

``
0x90b98a11
000000000000000000000000881f83D5317a12903472b89ccc54475e2a682d??
                                                              ^^
                                          A byte is missing here
0000000000000000000000000000000000000000000000000000000000000001  
``

Given this underflowed input data, the EVM would just add whatever bytes are missing up untill it reaches 68 bytes. (4 bytes => the method signature + 32 bytes => address + 32 bytes => amount). After the method signature, the missing bytes get counted together with the starting 00 from the amout, now effectively making up to 32 bytes. Because now one byte has shifted to the left, and the data now only has 67 bytes, EVM will add one 0 byte so the lenght of the data would be 68 bytes, and execute the call;

``
    0x90b98a11  
    000000000000000000000000881f83D5317a12903472b89ccc54475e2a682d00
    00000000000000000000000000000000000000000000000000000000000001??  
``

At this point, the EVM would supply the ?? with zeros. But the amount ins an unsigned 256 integer. So, as one byte has shifted to the left, the amount that would actually get passed to the call would be 1 << 8 = 256 WEI.

## Why ERC20
Basically, if address would have been the final parameter, instead of amount, the exploit would not have worked. Any token that implements the [ERC20 interface](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) is, in theory, succeptible to this vulnerability, because of the order in which parameters are defined in the interface:
```
function transfer(address _to, uint256 _value) public returns (bool success)
```
If the order of the parameters would have been switched, the extra zeros would have been added to the address, in which case it would not have made any difference.

## How can the exploit be performed?
1. The attacker has to control an address ending with one or more trailing 0
2. The attacker has to find an exchange wallet or any other offchain software that does not perform appropriate validation on the address
2. Use the above address to send an amount of X to the exchange wallet
3. The attacker withdraws his X amount from the exchange wallet back to his address but leaves off the trailing 0.
4. The balance is checked, and the operation is allowed. However, when the call to the EVM is made to perform the actual transacton, the data payload would be underflown. EVM will add trailing 0 at the end, actually modifying the amount, and broadcast the transaction

## Fixes
The community consensus on this one is that it should be handled at the level of the library that reads the [ABI](https://solidity.readthedocs.io/en/develop/abi-spec.html) contract (web3.js, truffle-artifactor, etc).
Because of this, the fix has been [removed](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/261) from smart contract frameworks like OpenZeppelin and from many individual contracts.

However, this got fixed in Solidity v0.5.0 [see changelog](https://github.com/ethereum/solidity/blob/v0.5.0/Changelog.md) by this [pull request](https://github.com/ethereum/solidity/pull/4224)
> Code Generator: Revert at runtime if calldata is too short or points out of bounds. This is done inside the ABI decoder and therefore also applies to abi.decode().

Should you opt to fix this in the contract layer, the easiest way would be to check if the message has the correct length, quite possibly in a modifier as shown [here](https://ethereum.github.io/browser-solidity/#gist=f5c444b9e087d03438aa990cb91b9e3a&optimize=false&version=soljson-v0.6.8+commit.0bbfe453.js). The first workaround was suggested by Reddit user [ izqui9 ](https://www.reddit.com/r/ethereum/comments/63s917/worrysome_bug_exploit_with_erc20_token/dfwmhc3/), while Peter Vessens suggests a [similar solution](https://github.com/MonolithDAO/token/blob/master/audit/TokenSaleAudit.pdf). A problem with these fixes has been found in regards to [Multisig wallets](https://blog.coinfabrik.com/smart-contract-short-address-attack-mitigation-failure/). As the short address attack can only take place on buffer underflows, checking if the raw input data is equal *or greater* than the size of all the arguments (as each argument should be 32 bytes). The *or greater* part would allow an overflow, and should resolve the issues with the Multisig wallets, which send more bytes than expected)
