---
title: "Bitcoin Transactions - Part 1: keys and addresses"
date: 2022-02-12T17:51:29-08:00
---


When you buy or sell bitcoin, the transaction  will be recorded and stored permanently on the Bitcoin network. The bitcoin transaction, and the cryptography behind it, is one of the most important and elegant part of Bitcoin. It also forms the foundation for other cryptocurrencies and blockchain platforms, including Ethereum and Solana. So becoming intimately familiar with the inner workings of the bitcoin tranaction is a pre-requisite to getting into the defi space as an engineer or developer. 

In this multi-part series, I am going to do a deep dive into this topic. You will see that with a little bit of know-how and some computer programming knowledge, you can recreate many parts of the Bitcoin protocol and see exactly how the bitcoin transaction works. 

Most guides do not dive deeply into the code, so you end up only with superficial knowledge of the Bitcoin protocol. To get that next level hands-on experience, we will use the Bitcoin Testnet to produce actual transactions that you can verify on Block Explorer and do it all using Python.   

I hope you are excited, because this is going to be a fun ride! 

## Part 1 - keys and addresses 

To create a bitcoin transaction, we first need accounts to hold the funds. In Bitcoin, this is commonly known as a wallet, but it is really just a collection of private and public key pairs. Using your randomly generated private key, you generate a public key that serves as your account number. Anyone who wishes to transfer you funds can do so by sending money to this account. You then verify that you have possession of the private key through a digital signature.

In Part 1, we cover how to generate the private and public keys, and in Part 2 we will discuss the digital signature in more detail. 

A private key is simply a string of randomly generated numbers. Anyone who has possession of the private key has access to funds associated with it. Therefore, it is also known as a "secret key" because you need to store this number privately. The private key is used to generate the public key, in a one-way (or trapdoor) function. This means you can generate the public key with the private key, but the reverse cannot be done. In Bitcoin, the trap door function used is the Elliptic Curve Digital Signature Algorithm (ECDSA). 

In contrast with the private key, the public key is the address that you can freely give out to people who want to transact with you. A transaction sent to a public key is known as a Pay-to-Public-Key (P2PK) transaction. The first bitcoin transaction was a P2PK transaction. This type of transaction is not common anymore as it has been replaced by another form of transaction, the Pay-to-Public-Key-Hash (P2PKH). 

Instead of giving out the public key, which is a massively long number, you give out a hash of the public key, a much smaller number. A hashing function is another type of trap door function, which takes the public key and creates the public key hash. The hashing process adds another layer of security. To date, P2PKH is the most common transaction on the Bitcoin Protocol. We are going to generate a P2PKH transaction from scratch to show you how it all works. 

In 2012 another type of transaction was introduced, known as Pay-to-Script-Hash (P2SH). Instead of sending funds directly to a recipient's public key or public key hash, a P2SH transaction sends funds to a script that has custom logic and conditions that allow for more complex transactions. An example of P2SH is a multisig transaction, which requires multiple people's digital signatures to unlock the funds. we will cover both P2PKH and P2SH in this series. 

## Coding time! 

In the coding portion below, I will generate a private key, use ECDSA to generate a public key and the hashing function to generate a public key hash. I will then transfer some bitcoins to this address so that we have some funds to play with and craft our own P2PKH transaction. 

Let's start! 

### Setting up the helper file

There is a lot of math in Bitcoin, but you don't need to fully understand them to learn how Bitcoin works. To make sure we can learn Bitcoin without being bogged down by the math, I created a helper.py file that has all of the needed algorithms and functions. 

To get started and follow along, you can clone my Github repo by typing the command below in terminal and then open it up in your editor of choice. I use Visual Studio Code. 

```
git clone -b 0-getting-started https://github.com/ergodefi/btc-deepdive.git
code 0-getting-started
```

Next, we are going to import the helper file and its useful library of data structures and functions. 



```python
from helper import Curve, Point, Generator, ec_addition, double_and_add 
```

We can now finally start slinging some codes! 



### Creating the bitcoin curve

First, we need to build parts of the ECDSA algorithm, which is the trapdoor function used to generate the public key from the private key. 

We can now start by creating the "Bitcoin curve" also known as the secp256k1, a specific type of ellitpical curve set over a finite field. We can define some constants at this point, including the generator point (G) and the point at infinity. 

Don't worry if you are confused at this point. There are [lots](https://www.coindesk.com/markets/2014/10/19/the-math-behind-the-bitcoin-protocol/) of great articles written to help you understand ellptical curve cryptography and how it works. 



```python
# secp256k1 ellptical curve constants - y^2 = x^3 + 7 (mod p)
bitcoin_curve = Curve(
    p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F,
    a = 0x0, 
    b = 0x7, 
)

# generator point 
G = Point(
    bitcoin_curve, 
    x = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798, 
    y = 0x483ada7726a3c4655da4fbfc0e1108a8fd17b448a68554199c47d08ffb10d4b8,
)

bitcoin_gen = Generator(
    G = G,
    n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141, 
)

# point at infinity 
INF = Point(None, None, None) 
```

### Elliptical math: point addition and multiplication 

Because we are working with points on a finite field, the concept of addition and multiplication are also different. When addition or multiplication takes the value outside of the finite field, the value "wraps around" so that it would fall into this range. 

If you have ever played classic games incluidng Asteriod or Donkey Kong, when you go out of bounds on one side of the screen, you appear on the other side. It is the same concept here. 

Therefore, we have to define point addition and point multiplication differently. I have defined those functions in the helper file, and you can take a look at how they work if you're interested. 

Here, we simply have to change the default addition and multiplication functions with point addition and point multiplication. 

```python
Point.__add__ = ec_addition  # point addition
Point.__rmul__ = double_and_add # point multiplication
```

### Generating the private key

Now we are ready to generate our identity (i.e. private key).

```python
# identity 1
secret_key = int.from_bytes(b'super secret identity one', 'big') # for reproducibility 
assert 1 <= secret_key < bitcoin_gen.n 

```

The generation of a private key is done automatically by your Bitcoin wallet application, and it should be doneusing a source of entropy to ensure complete randomness. Basically, it is extremely important that your private key is random, since that is the only guarantee you have that your pricate key is secure. If the random generator of your wallet is falwed or if your private key is ever exposed to someone else, it is no longer secure. Anyone with access to your private key can transfer all the funds associated with the key. So never share your private key and make sure your wallet application has a flawless random generator. 

But for our educational purpose, we can create a static private key so that you can reproduce each step. Furthermore, we are using Testnet, so there's no real bitcoin changing hands. 

### Generating the public key 

Once we have generated the private key, we simply perform elliptical curve multiplication with the generator point G  to find the public key.

```python
public_key = secret_key * G
public_key
```
    Point(curve=Curve(p=115792089237316195423570985008687907853269984665640564039457584007908834671663, a=0, b=7), x=35490547311314112975969199385462927466356376524965552000974623035901126229990, y=75829577894590863462191837680945451999817850420713104019785938471674831323880)

You may be surprised to know that the public key is actually not a single value, but rather (x, y) coordinates. Intuitively this makes sense, since you are multiplying a randomly generated number by G, which is itself a coordinate on the secp256k1 curve. 

At this point, the "public_key" variable is a Point object. But as we'll see shortly, the Public Key takes on many forms and serves many purposes in a bitcoin transaction, so it's kind of a big deal and deserves its own object. 


```python
from helper import PublicKey
```

Now, let's see the various forms of the public key. 

1) **Public Key (Uncompressed)** - In it's original form, the public key it is a set of coordinates, (x, y) on the secp256k1 curve. Each of the coordinate is 256-bit in size, totalling 512 bits. 

2) **Public Key (Compresse)** - it turns out that you can calculate y given x, so there is no point in storing the y value for a transaction. Since most transactions include the public key as part of its verification process, halving the size of the public key is a significant space reduction, from 512 to 256 bits 

3) **Public Key Hash** - hashing the public key (compressed or uncompressed) using a combination of two hashing functions, SHA256 and RIPEMD160, will create the public key hash. In a P2PKH transaction, this is what you give out to the sender. It introduces another layer of security by introducing another trapdoor function and also further reduce the size of the transaction to 160 bits. 

4) **Bitcoin Address (Base58Check)** - the final form of the public key is the Bitcoin Address. It is actually just another way to encode the public key hash to make it more human readable by using the Base58Check encoding. In this format, both uppercase and lowercase letters are used alongside numbers, but certain values are removed (i.e. zero and uppercase i) to reduce transcription errors. You won't see the base58check encoded address format on the Bitcoin Protocol. Its purpose is purely for human readability. 


```python
public_key_compressed = PublicKey.from_point(public_key).encode(compressed=True, hash160=False).hex()
public_key_hash = PublicKey.from_point(public_key).encode(compressed=True, hash160=True).hex()
bitcoin_address = PublicKey.from_point(public_key).address(net='test', compressed=True)

print("Bitcoin Identity #1")
print("* Secret (Private) Key: ", secret_key)
print("* Public key (uncompressed): ", (public_key.x, public_key.y))
print("* Public key (compressed): ", public_key_compressed) 
print("* Public key hash: ", public_key_hash) 
print("* Bitcoin address (base58check): ", bitcoin_address)

```
    Bitcoin Identity #1
    * Secret (Private) Key:  724746296646138075612064989570816355802000824461885300502117
    * Public key (uncompressed):  (35490547311314112975969199385462927466356376524965552000974623035901126229990, 75829577894590863462191837680945451999817850420713104019785938471674831323880)
    * Public key (compressed):  024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6
    * Public key hash:  363bb1ef1d8791bdbd7e7492ef91decc1eb7295d
    * Bitcoin address (base58check):  mkTiJ6dtXpYJqAremsCZtEww2XFWw3f2WT


Now let's take a look at the outputs from our code above. Notice how much longer the uncompressed public key is? Also notice how much shorter the bitcoin address is when encoded in base58check? 

### Getting some bitcoins! 

Now that we have our identity and account in the form of private key and public key, the final step is to receive some bitcoins!

Ordinarily, this would involve you purchasing some bitcoin for cash, but in the magical world of Testnet, we can just ask for some! 

Head over to a Testnet Faucet, such as this [one](https://bitcoinfaucet.uo1.net/) and simply enter the bitcoin address we just created, *mkTiJ6dtXpYJqAremsCZtEww2XFWw3f2WT* and an amount, *0.0001* bitcoin. 


![Testnet Faucet Transfer](/images/testnet_faucet_transfer.png)

After requesting the bitcoins, scroll down on the faucet page and you'll see that the transaction has gone through and a transaction id has been generated: *6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24*. This is the transaction id (txid) that will uniquely identify this transaction on the blockchain. 

![Testnet Faucet Pending](/images/testnet_faucet_pending.png)

Also notice that the transaction is pending. 

Bitcoin is a peer-to-peer (p2p) network, and transactions are validated through concensus on this p2p network. When a transaction is first created, it is unconfirmed by any node on the network so it is pending. However, wait a few minutes and you'll see some confirmations. 



![Testnet Faucet Confirmed](/images/testnet_faucet_confirmed.png)

You can also head over to a block explorer website to see the transaction recorded on the blockchain (the Testnet blockchain that is). I used blockchain.com and searched for the public key hash generated earlier to find the [transaction](https://www.blockchain.com/btc-testnet/tx/6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24). 


![Block Explorer Transaction](/images/blockexplorer_faucet_tx.png)

From this transaction, you can see what happens when you asked for 0.0001 bitcoin from the Testnet Faucet. It creates a transaction which sends the fund to the bitcoin address you specified. In the next part, I will show you how to generate your own bitcoin transaction to transfer some of the fund to a second address while sending the rest back to this one. This will simulate a purchase and getting change back. Stay tuned! 
