---
title: "Developing Bitcoin Intuition - Transactions: Part 1"
date: 2022-02-12T17:51:29-08:00
---

## Introduction 

Bitcoin is one of the most imaginative and mysterious inventions of our time. It brought a revolution against current financial institutions, introduced the idea of decentralized finance and inspired countless others from embarking on this journey. So becoming intimately familiar with the inner workings of Bitcoin is a prerequisite for blockchain engineering, developing smart contracts and defi in general. 

I became fascinated with this topic during the pandemic lockdown and dove deep into the rabbit hole. I wanted to develop a sense of intuition on Bitcoin, without being bogged down by all the prerequisite knowledge in finite math, public-key cryptography, and decentralized architecture. In a sense, I want to build a mental model of how Bitcoin works. Since I could not find this easily online, I decided to write my own, while also documenting my defi learning. 

In this series, called Developing Bitcoin Intuition, I would like to explore every facet of this elegant invention, explaining how it works plainly and efficiently. I will also demonstrate how it works using code, Python in this case, as it is the language I have the most familiarity. Its clear syntax and lack of excessive structure lend itself well to illustrating the many ideas found in Bitcoin. 

I will start from the ground up, explaining the most important concepts, starting with the **Bitcoin transaction**. I am going to show you how to construct bitcoin transactions from scratch, step by step. We will create actual transactions that will be visible to everyone on the Internet, so that you can verify my work.  

I hope you are excited. This is going to be a fun ride! 

## How this series is organized 

To develop intuition for Bitcoin and form our own mental model, we need to do more than merely scratching the surface, but at the same time, not to get bogged down by the math and technical details. To facilitate this process,  I've already created most of the math and supporting functions in Python. Where possible, I'll show snippets of code to help you understand important concepts. 

There will be some coding left to do, mostly declarative work to call the functions I've already created. This way, you can follow along and work step-by-steo to produce the end result. 

## Part 1 - keys and addresses

Imagine you walked into a bank with $100 you want to deposit. What do you need before you can do that? A bank account! 

This is the same in Bitcoin. Before you can transact in it, you need an account. This is known as a wallet, but it is really just a collection of private and public key pairs. If this is the first you've heard of public and private keys, I highly recommend that you read about [public-key encryption](https://www.cloudflare.com/en-ca/learning/ssl/how-does-public-key-encryption-work/) as it is one of the fundamental building blocks of not just Bitcoin, but Internet security. 

The **private key** is simply a string of randomly generated numbers. You do not want to disclose your private key as anyone with access to your private key can access the funds associated with it. In this sense, the private key is simultaneously your banking password and credit card number at the same time. To transact, you need to give the other party something to identify yourself and serve as your account number, and this is why the public key exists. 

The **public key** is derived from the private key using a one-way function and is used publicly as an address. People who want to transfer money to you will transfer to your public key. A one-way function is a function that takes a value and produces another value deterministically, but but reverse cannot be done. You can transform the private key into the a public key, but you cannot reverse engineer the public key into the private key. There are several one-way functions, including hash functions, RSA and ECC. Bitcoin uses ECC or Elliptical Curve Cryptography to achieve this. 


### Primer on ECC 

ECC is one of the most complex and least understood aspects of understanding Bitcoin. Understanding it, at least conceptually, is required if you want to have a good grasp of Internet security. I recommend starting with [this](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/) article for an introduction, and if your interest is piqued, read Chapters 1 to 4 of Jimmy Song's excellent book [Programming Bitcoin: Learn How to Program Bitcoin from Scratch](https://amzn.to/3i4QyKo) to learn how to implement this in Python. 


### Bitcoin curve

The "ec" in ECDSA stand for ellptical curve, defined by the equation `y^2 = x^3 + ax +b`. Bitcoin uses a particular curve called secp256k1, which is defined by `a = 0` and `b = 7`. This curve is set over a finite field to the order of `P = 2**256 - 2**32 - 977`, which is a massive number, bigger than the number of atoms in the perceivable universe. So the chance of a collision is basically zero. Finally, there is a generator point `G` which is also defined by Bitcoin. 

These are all constants used for every Bitcoin transaction, and we already defined them when creating the helper classes. Here they are if you are curious: 

```python
# Defining the secp256k1 curve constants
A = 0
B = 7
P = 2**256 - 2**32 - 977
N = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141
G = S256Point(
    0x79be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798,
    0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8)
```
Once we have defined the curve and constants, we can see precisely the relationship between the private key and the public key. 

### Private key and public key

The private key is a random number, and the public key is simply `private_key * G`. This is the private and public key combination that we will create shortly: 

```python 
from ecc import PrivateKey

secret = int.from_bytes(b'super secret identity one', 'big') # for reproducibility 
privateKey = PrivateKey(secret)
publicKey = privateKey.point # secret * G 

print("* Private key: ", privateKey.secret)
print("* Public key: ", (publicKey.x.num, publicKey.y.num)) 
```

```output
* Private key:  724746296646138075612064989570816355802000824461885300502117
* Public key:  (35490547311314112975969199385462927466356376524965552000974623035901126229990, 75829577894590863462191837680945451999817850420713104019785938471674831323880)
```

You may be surprised to see that the public key is actually not a single value, but rather a set of (x, y) coordinates on the secp256k1 curve. Intuitively this makes sense, as you are multiplying a randomly generated number by G, which is itself a coordinate. 

### The nany forms of public key

In its original form, the public key is very long and cumbersome to display. So Bitcoin uses several methods to shorten this value without losing its precision so that it is more easily transmitted across the Bitcoin Network and shared with people in transactions. 
Here are the various forms of the public key: 

1) **Public Key (Uncompressed)** - In it's original uncompressed form, the public key it is a set of coordinates, (x, y) on the secp256k1 curve. Each coordinate is 256-bit in size, totalling 512 bits or 64 bytes. 

2) **Public Key (Compressed)** - It turns out that the y-coordinate can be calculated once you know the x-coordinate. So you just need to transmit and share the x-coordinate. The compressed format is 33 bytes (32 byte x-coordinate plus a prefix to indicate whether the y-coordinate is positive or negative). 

3) **Public Key Hash** - Hash functions are another type of one-way functions, taking a message and producing a hash (or fingerprint) of it. The hashing function used to create the public key hash is called HASH160, which is a combination of two hash functions: SHA256 and RIPEMD160. Hashing the compressed or uncompressed public key creates a 160-bit or 20 byte public key hash. Hashing provides 2 benefits. First, it introduces another layer of security into the Bitcoin transaction, so that it is protected by both ECDSA and HASH160. Second, it further shortens the length from 33 bytes to  20 bytes

4) **Bitcoin Address (Base58Check)** - The final form of the public key is the Bitcoin Address. This is simply the public key hash encoded using Base58Check encoding to make it more human readable. Base58Check uses both both uppercase and lowercase letters and numbers, but certain values are removed (i.e. zero and and uppercase i) to reduce transcription errors. 

### Time to code! 

To get started and follow along, you can clone my Github repo by typing the command below in terminal and then open it up in your editor of choice. My text editor of choice is [Visual Studio Code](https://code.visualstudio.com/blogs). 

```output
git clone -b 0-getting-started https://github.com/ergodefi/btc-deepdive.git
code btc-deepdive

```

*Note: If you are keen to learn how Bitcoin works in greater depth, you can review the code that have already been written in the various files in the repo: ecc.py, helper.py, op.py, script.py and tx.py. A huge shoutout to Jimmy Song and his excellent excellent book [Programming Bitcoin: Learn How to Program Bitcoin from Scratch](https://amzn.to/3i4QyKo). I based most of my code around this book as it is very well written.*

### Generate the keys 

In the code below, we will generator a private key, multiply it by `G` to generate the public key, perform `HASH160` to generate the public key hash, and finally encode it with `Base58Check` to create the Bitcoin adddress. You can see how each step creates a shorter key so that we end up with the much more manageable Bitcoin address. 

```python
from ecc import PrivateKey

secret = int.from_bytes(b'super secret identity one', 'big') # for reproducibility 
privateKey = PrivateKey(secret)
publicKey = privateKey.point # secret * G 

print("* Private key: ", privateKey.secret)
print("* Public key (Point): ", (publicKey.x.num, publicKey.y.num)) 
print("* Public key (SEC Compressed): ", publicKey.sec().hex())
print("* Public key (SEC Uncompressed): ", publicKey.sec(compressed=False).hex())
print("* Public key hash: ", publicKey.hash160().hex())  
print("* Bitcoin address ", publicKey.address(testnet=True))
```

```
Bitcoin Identity #1
* Private key:  724746296646138075612064989570816355802000824461885300502117
* Public key (Point):  (35490547311314112975969199385462927466356376524965552000974623035901126229990, 75829577894590863462191837680945451999817850420713104019785938471674831323880)
* Public key (SEC Compressed):  024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6
* Public key (SEC Uncompressed):  044e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6a7a605274e750d1d70a8548c96417d8036c4fb8b6d4296308505c2d8799a42e8
* Public key hash:  363bb1ef1d8791bdbd7e7492ef91decc1eb7295d
* Bitcoin address  mkTiJ6dtXpYJqAremsCZtEww2XFWw3f2WT
```

Note that normally the private key should be randomly generated, but since I want to make this exercise reproducible, I've used a static private key. Do not do this with your real bitcoin wallet! 

Now that we have our "bank account", let's deposit some money! 

### Obtain some bitcoins from Testnet 

Ordinarily, to transfer bitcoins into your wallet, you need to buy some bitcoins with an exchange. Fortunately, we can demonstrate this for free by using Testnet, a development environment where Bitcoin developers can build and test using testnet Bitcoins (tBTC). 

Head over to a [Testnet Faucet](https://bitcoinfaucet.uo1.net/) and simply enter the bitcoin address we just created, *mkTiJ6dtXpYJqAremsCZtEww2XFWw3f2WT* and an amount, *0.0001* bitcoin. 

![Testnet Faucet Transfer](/images/testnet_faucet_transfer.png)

After requesting the bitcoins, scroll down on the faucet page and you'll see that the transaction has gone through and a transaction id has been generated: *6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24*. This is the transaction id (txid) that will uniquely identify this transaction on the blockchain. 

![Testnet Faucet Pending](/images/testnet_faucet_pending.png)

Also notice that the transaction is pending. 

Bitcoin is a peer-to-peer (p2p) network, and transactions are validated through concensus on this p2p network. When a transaction is first created, it is unconfirmed by nodes on the network, so it is pending. Wait a few minutes and you'll see some confirmations. 

![Testnet Faucet Confirmed](/images/testnet_faucet_confirmed.png)

You can also head over to a block explorer website to see the transaction recorded on the blockchain (the Testnet blockchain that is). I used [blockchain.com](https://www.blockchain.com/) and searched for the public key hash generated earlier to find the [transaction](https://www.blockchain.com/btc-testnet/tx/6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24). 

![Block Explorer Transaction](/images/blockexplorer_faucet_tx.png)

From this transaction, you can see what happens when you asked for 0.0001 bitcoin from the Testnet Faucet. It creates a transaction which sends the fund to the bitcoin address you specified. In [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}), I will show you how to construct your own transaction to transfer some of the bitcoins to a second address while sending the rest back to this one. This will simulate a purchase and getting change back. Until next time! 

