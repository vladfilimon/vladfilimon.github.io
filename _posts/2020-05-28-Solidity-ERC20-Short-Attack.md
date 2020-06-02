---
layout: post
title: ERC20 Short Attack
---
> On: Security, Solidity, Smart Contracts, EVM

This underflow attack is a very interesting one as it relies on a couple of things, like lack of input validation, Ethereum not ~~implementing~~ enforcing checksum on addresses, and finally the way the EVM adds 0 for a missing byte, packs together the method call and the arguments.
This type of attack has never been observed in the wild, but is theoretically possible nonetheless. It is important to note that smart contracts cannot be attacked in this manner, but 3rd party (offchain) software that interacts with the smart contracts.

A transitional [checksum system] (https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md) has eventually been developed for Ethereum, though it's not somemthing at protocol level, leaving it up to the wallets and exchanges if they choose to implement it or not.
However, it is different from the one most other blockchains use (starting with Bitcoin) which is to encode the string in base 58 with an added version number and checksum, as Ethereum's EIP-55 would only capitalize some characters in the string:
> Convert the address to hex, but if the ith digit is a letter (ie. it's one of abcdef) print it in uppercase if the 4*ith bit of the hash of the lowercase hexadecimal address is 1 otherwise print it in lowercase.

There is a very interesting read regarding the checksums and addesses [here](https://ethereum.stackexchange.com/questions/267/why-dont-ethereum-addresses-have-checksums/274#274)

In a contract-invocation transaction the function hash signature, alongside the arguments, are orderly encoded in the input field. The first four bytes specify callee function and the rest of the arguments are arranged in chunks of 32 bytes.
If the lengt of the encoded arguments happens to be less than expected, EVM will auto-pad extra zeros to the arguments untill the correct lenghth of 32 bytes is reached.

## How can the exploit be performed?
1. The attacker has to control an address ending with a trailing 0.
2. Use the above address to send an amount of X to an exchange wallet.
3. The attacker withdraws his X amount from the exchange wallet back to his address but leaves off the last 0

Looks like this got fixed in Solidity v0.5.0 [see changelog](https://github.com/ethereum/solidity/blob/v0.5.0/Changelog.md)
> Code Generator: Revert at runtime if calldata is too short or points out of bounds. This is done inside the ABI decoder and therefore also applies to abi.decode().