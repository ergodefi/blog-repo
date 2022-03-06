---
title: "Bitcoin Transactions - Part 4: Unlocking the UTXO"
date: 2022-03-03T16:49:26-08:00
---

Nothing individual aspect of Bitcoin is particularly groundbreaking or cutting edge. All of the tools and techniques that make up Bitcoin (i.e. ECDSA, public-key crytography, hash functions and so on) have been around since the late 90s. So it would seem that *there is no new thing under the sun*.

Yet, when you combining them in a specific way, you end up creating the first true cryptocurency. It makes me wonder what else is possible, but hiding in plain sight waiting for someone to piece together? 


## Answer to the Pop Quiz 
Here is the answer to the pop-quiz from [Part 3 pop quiz]({{< ref "3-bitcoin-pt3-script-sig-full-tx.md#pop-quiz" >}}). This exercise serves as a review of Parts 1 through 3. 

Here is the transaction we generated, encoded in hex format. I've separated the bytes so that it is more readable, but it is still just a mess of numbers! Let' decode this together. 

Transaction:`01 00 00 00 01 24 9c ff 3c b2 83 16 39 8c 5f 02 e3 43 31 ad d9 6a 5e 87 7f cf 9f 96 7c 06 57 52 6d 5c af 07 67 01 00 00 00 6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6 ff ff ff ff 02 f8 24 01 00 00 00 00 00 19 76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac f0 55 00 00 00 00 00 00 19 76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac 00 00 00 00`

Remember that every bitcoin transaction has 4 sections: version, inputs, outputs and locktime. They are encoded in this order and some of the value are stored as [little endian]({{< ref "3-bitcoin-pt3-script-sig-full-tx.md#endianness" >}}), so that they need to be swapped to obtain the real values. 

### 1. Version
Version:`01 00 00 00`

The first 4 bytes give us the *version* of the transaction, stored as little endian. Once you swap endian, you get `version 1`. This is the only version currently, but in the future we will likely get newer versions. This is Bitcoin's way of keeping the protocol flexible and future proof. 

### 2. Inputs 
Input is the largest part of the transaction. There could be 1 or more inputs to the transaction. Each input references the UTXO from a previous transaction and provides the script_sig or unlocking script, used to unlock the UTXO.

Input:`01 24 9c ff 3c b2 83 16 39 8c 5f 02 e3 43 31 ad d9 6a 5e 87 7f cf 9f 96 7c 06 57 52 6d 5c af 07 67 01 00 00 00 6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6 ff ff ff ff`

The next byte is the *input counter*. There is `01` input. 

The next 32 bytes is the *previous transaction id*, stored as little endian. Once we swap endian, we end up with `67 07 af 5c 6d 52 57 06 7c 96 9f cf 7f 87 5e 6a d9 ad 31 43 e3 02 5f 8c 39 16 83 b2 3c ff 9c 24`. We can verify that this is the [Testnet Faucet transaction](https://www.blockchain.com/btc-testnet/tx/6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24) that gave us 0.0001 tBTC. Notice that the transaction hash is the same. 

The next 4 bytes are the *input index*, telling us which output from the previous transaction we would like to spend. This is again stored in little endian. Swap endian and remove the trailing zeros and you get `1` which is the index for the **second** output (remember array index starts at 0). 

By referencing the the previous transaction id and index, the Bitcoin protocol can find that output and infer its value. You don't need to actually specificy the value of the input. 

#### script_sig

The rest of the input script, except the last 4 bytes, give us the *script_sig*. 

Script_sig:`6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6`

The script_sig is encoded differently from what we've seen so far, as (1) it is encoded in big endian, and (2) it makes uses of variable integers (varInts) extensively. Recall the script_sig is made up of the digital signature (r, s) values and the unhashed public key. The length of r and s values vary, so we need to tell the Bitcoin network how many bytes to expect. We do that using varInts. 

The first two bytes are both varInts. The first varInt gives us the length of the entire script_sig. The second varInt gives us the length of the signature portion. 

DER Sequence: `30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6`

From here, the next byte, `30` indicates the start of *DER encoding*, and this is followed by a varInt, `44` to indicate the length of the DER sequence, which works out to 68 bytes. Recall from [Part 3]({{< ref "3-bitcoin-pt3-script-sig-full-tx.md" >}}) that the signature (r, s) values are encoded using the DER sequence. 

The next byte, `02`, indicates that an integer will follow, the *r value* in this case. This is followed by  another varInt, `20`, so the next 32 bytes make up the r value. Finally, we get the r value itself. The encoding follows the same pattern for the *s value*. 

Once we decode both, we end up with (r, s) values: `(54944a0b195ffbb1d88458375a29c31fbc46fdb7cc5a9befa99d07b38ee00e10, 0da298a28c5f1f90d2e8efb625f817608ecc3d6b16da57f6cbfb46008aab8453)`

SIGHASH flag: `01`

The signature ends with 1 last byte for *SIGHASH flag*. This byte indicates which parts of the transaction are hashed to  generate the digital signature. In our transaction, `01` indicates SIGHASH_ALL, which means sign all inputs and outputs are signed. This is by far the most common flag. 


Finally, we have the public key. The next byte is a *varInt* of the length of the public key, `21`. The next 33 bytes are the public key: 

Pubkey:`024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6`

#### Sequence
The final 4 bytes are the *sequence*, which comes immediately after every input.  This feature is seldom used, and defaults to `ff ff ff ff`

If there is more than 1 input, each input will be encoded sequentially in this order, one after the other, beginning with the previous transaction id and ending with the sequence. Since there is just one input, the sequence indicates the end of all inputs and start of output. 

### 3. Outputs
Outputs are much more straightforward to decode as each output has just two elements: a value and a locking script. 

Outputs:`02 f8 24 01 00 00 00 00 00 19 76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac f0 55 00 00 00 00 00 00 19 76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac `

The first byte is the output counter. There are `02` outputs. Recall that we created 2 outputs to simulate the purchase of an item and getting the change back. 

The next 8 bytes are the value of the first output, stored as little endian. After swapping endian and removing the trailing zeros, you get `0124f8` which is hex for 75000 sats. 

The next byte, `19`, is the hex length of the script_pubkey or locking script. After converting to decimal, we know that the next 25 bytes are the locking script, which are: 

Locking script for first output: `76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac`

We can do the same for the second output, giving us a value of 22000 sats and a locking script of: 

Locking script for second output: `76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac`

### 4. Locktime 
The last 4 bytes are the locktime, which, like sequence, is a feature that is currently seldom used, and as such as set as their default value, `00 00 00 00`. And there you have it! The entire transaction decoded. 

## Unlocking the UTXO

This is where we learn how Bitcoin's locking and unlocking mechanism actaully works. While I could just create a python function to perform this task, it will be more instructive if we take a more manual approach and do this by hand, step by step. 

### Assembling the scripts 

First we must assemble the scripts. This is mostly a review, since we have collected both scripts.  

In [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}), we asked for 0.0001 BTC from the Testnet Faucet. This automatically created a transaction and the 0.0001 BTC we received is an UXTO. It came with its own locking script (the script_pubkey). Recall we stored the script in variable during the signing process: 

```python
print("Locking script:", prev_tx_script_pubkey.encode().hex())
```
`Locking script: 1976a914363bb1ef1d8791bdbd7e7492ef91decc1eb7295d88ac`

Then in [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}) and [Part 3]({{< ref "3-bitcoin-pt3-script-sig-full-tx.md" >}}), we created our own transaction and generated the the script_sig (or unlocking script): 

 Unlocking script: `6a473044022054944a0b195ffbb1d88458375a29c31fbc46fdb7cc5a9befa99d07b38ee00e1002200da298a28c5f1f90d2e8efb625f817608ecc3d6b16da57f6cbfb46008aab84530121024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6`


### Clean up the scripts 

Having assembled the correct scripts, we need to clean them up (i.e. removing unnecessary parts) before we can use them. 

For the **unlocking script**, recall that it is made up of the signature and the public key. We want to only extract the DER encoded signature block and the public key, discarding the varInt. 

\<signature>`3044022054944a0b195ffbb1d88458375a29c31fbc46fdb7cc5a9befa99d07b38ee00e1002200da298a28c5f1f90d2e8efb625f817608ecc3d6b16da57f6cbfb46008aab84530121024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6`

\<pubkey>`024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6`

For the **locking script**, recall that it is made up of varInts, op_codes and the public key hash. We must discard the varInts and extract the public key hash. I've also expressed the op_codes to make this more clear. We eventually end up with: 

Locking script: `OP_DUP OP_HASH160 363bb1ef1d8791bdbd7e7492ef91decc1eb7295d OP_EQUALVERIFY OP_CHECKSIG`


### Primer on the Script language

We are now ready to do the verification, but how does this work exactly? 

Bitcoin has its own programming language simply called Script. It is described as a *\"Forth-like, stack-based, reverse-Polish, Turing incomplete programming language."* Phew. That is a lot! Let's explain each term one by one:

* **Forth-like** means Script resembles [Forth](https://en.wikipedia.org/wiki/Forth_(programming_language)), which is an ancient old programming language that you likely have never heard of. One example of this resemblance is Script's use of op_codes, which Forth introduced.  

* **Stack-based** means Script evaluates commands on the *stack* data structure. The stack is a data structure much like a stack of plates. To remove the bottom plate, you have to first remove all the plates above it. Hence, a stack data structure has the characteristic of last in first out (LIFO). You'll see why this is important in the next section when we evaluate the locking and unlocking scripts. 

* **Reverse-Polish** is a different way of expressing mathematical notations, where the operators follow the operand. Normally, we would express addition as `3 + 4`, but with the Reverse Polish Notation, we would express the same expression as `3 4 +` instead. Reverse Polish Notation was invented to save computer memory and improve calculation speed in the 50s. 

* **Turing incomplete** means Script is not a complete programming language, like C or Python. Script can perform certain operations butnot other more complex ones, such as looping and conditional statements. This was an intentional design decision to prevent abuse. Only simple scripts can be executed. 

### Evaluating the scripts 
With the primer on Script, let's now see the unlocking and locking scripts are evaluated, step-by-step. 

First, we assemble the elements in the order of execution. Bitcoin protocol specifies we load the unlocking script then locking script, like so: 
```
<signature>
<pubkey>
OP_DUP 
OP_HASH160 
<pubkey_hash>
OP_EQUALVERIFY 
OP_CHECKSIG
```

Bitcoin protocol will execute each element, one at a time. If it is a value, it is pushed onto the stack. If it is an op_code, the op_code runs, consuming one or more elements on the stack. This is repeated until all of the elements are executed. At that point, whatever is left on the stack will determine whether whether the transaction is validate and the UTXO is unlocked. 

Ready? Let's give it a try! 


#### 1. \<Signature> 
This is a value, so push it onto the stack. 

#### 2. \<pubkey> 
Another value, push onto the stack! 

#### 3. OP_DUP  

Our first op_code is `OP_DUP`. This pushes a copy of the topmost stack item on to the stack. Since the topmost item is the  \<pubkey>, it is copied and pushed onto the stack. 

#### 4. OP_HASH160 

`OP_HASH160` consumes the topmost item on the stack, , the \<pubkey>, performs hash160 on it, and then push the resulting hash onto the stack. 

Note that hash160 is not a hash function, but a combination of SHA256 and then RIPEMD160 such that `RIPEMD160(SHA256(<pubkey>)) = <pubkey_hash>`. 

In the *helper.py*, we've already implemented both RIPEMD160 and SHA256, so we can observe `OP_HASH160` in action. 

```python
from helper import ripemd160, sha256

pubkey = '024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6' 
pubkey_bytes = bytes.fromhex(pubkey) # convert pubkey to bytes before hashing 
pubkey_hash = ripemd160(sha256(pubkey_bytes)) # hash160 is ripemd160(sha256())
print("Pubkey_hash: ", pubkey_hash.hex())
```
`Pubkey_hash:  363bb1ef1d8791bdbd7e7492ef91decc1eb7295d`

#### 5.  \<pubkey_hash> 
Another value, push it onto the stack! Astute observers may remark this value, `363bb1ef1d8791bdbd7e7492ef91decc1eb7295d`, is the same as the \<pubkey_hash> we just calculated from hash150. Is this a coincidence? I think not!


#### 6. OP_EQUALVERIFY

This op_code is comprised of two op_codes: `OP_EQUAL` then `OP_VERIFY`. 

`OP_EQUAL` consumes the top two items on the stack, compares them, and pushes 1 onto the stack if they are the same, or 0 if different. `OP_VERIFY` then consumes this top item (i.e. 1 or 0). If it is a 1, continue with the script. If 0, terminate the script. 

This step basically checks whether the hash160 of the \<pubkey> from the unlocking script is equal to the \<pubkey_hash> of the locking script. 

We can see that they are equal, so when `OP_EQUAL` is executed, both items are consumed, and a 1 is pushed onto the stack. Then, `OP_VERIFY` consumes the 1 and lets us continue with the script. 

The verification of the public key hash is why this type of transaction is called Pay to Public Key Hash (P2PKH). 

In addition to the verification of the digital signature (`OP_CHECKSIG`, that is the the next step), P2PKH transactions add an extra layer of protection, leveraging the hash functions. Therefore, a P2PKH transaction is protected by ECDSA, RIPEMD160 and SHA256. 


#### 7. OP_CHECKSIG 

At this point, we have the \<signature> and \<pubkey> on the stack, and one more op_code to run, OP_CHECKSIG. 

`OP_CHECKSIG` consumes the top two items on the stack: (1) the digital signature and (2) public key. It checks whether the signature was generated by the correct private key, without revealing the private key. If so, it pushes 1 onto the stack. Otherwise, it pushes 0. 

Since this is the last element for execution and it consumes the 2 items that are, so whatever is left on the stack should be either 1 or 0. If 1 is the remaining value on the stack, it means the script was successful, and the UTXO is unlocked. Conversely, a 0 would mean invalid script and the UTXO will stay put. 

Pretty neat huh? 

Okay, but how does `OP_CHECKSIG` actually work? 

Not surprisingly, the answer is MATH, or ECDSA verification math to be more precise. This is a big section, so we'll do it next time. We will also finalize our transaction and propate it to the Testnet Bitcoin Network! Stay tuned. 
