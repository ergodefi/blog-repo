---
title: "Bitcoin Transactions - Part 4: Unlocking the UTXO"
date: 2022-03-03T16:49:26-08:00
---

In our final part of this series on the Bitcoin transaction, we will observe how Bitcoin's locking and unlocking mechanism works. We are also going to transmit our transaction to the Bitcoin network. Finally, I will conclude with a challenge: *for you to steal the tBTC in my wallet!* 

If you stumbled upon this page, I recommend you start in order since each part builds upon the knowledge and code of the previous. 

In [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}), we learned about public and private keys, how they are the backbone of wallet security and generated our first identity for our wallet. We also transferred some Testnet bitcoins from a faucet application to our wallet. At the end, we have 0.0001 tBTC. 

In [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}), we generated a second identity and constructed our own transaction, a Pay to Public Key Hash (P2PKH) transaction to transfer the 0.0001 BTC from identity #1 to identity #2 and the change back to identity #1.

In [Part 3](), we encoded and decoded our transaction. Along the way, we learned how the Bitcoin network processes and transmits data as well as learn about some fundamental computer networking concepts including serialization, deserialization, hexadecimal and endianness. 


## Unlocking the UTXO

### Quick recap

Unlocking the UTXO requires both the unlocking script and the locking script. So the first step is we must assemble them. 

In [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}), we asked for 0.0001 BTC from the Testnet Faucet. This automatically created a transaction and the 0.0001 BTC we received is the UXTO. It came with its own locking script (the script_pubkey).












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






## Conclusion

Nothing individual aspect of Bitcoin is particularly groundbreaking or cutting edge. All of the tools and techniques that make up Bitcoin (i.e. ECDSA, public-key crytography, hash functions and so on) have been around since the late 90s. So it would seem that *there is no new thing under the sun*.

Yet, when you combining them in a specific way, you end up creating the first true cryptocurency. It makes me wonder what else is possible, but hiding in plain sight waiting for someone to piece together? 