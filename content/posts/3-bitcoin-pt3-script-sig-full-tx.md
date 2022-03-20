---
title: "Bitcoin Transactions - Part 3: Signing and Generating Full Transaction "
date: 2022-02-23T16:49:26-08:00
---

In [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}), we learned about public and private keys, how they are the backbone of wallet security and generated our first identity for our wallet. We also transferred some Testnet bitcoins from a faucet application to our wallet. At the end, we have 0.0001 tBTC. 

In [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}), we generated a second identity and constructed our own transaction, a Pay to Public Key Hash (P2PKH) transaction to transfer the 0.0001 BTC from identity #1 to identity #2 and the change back to identity #1.

Today we are going to take this transaction apart and show you each element. In the process, we are going to learn how the Bitcoin's p2p network transmits transactions. We are going to gain a deeper understanding of some fundational computer science knowledge.  

There is no new code today. Instead, we are going to review some of the functions we have already created.  

Let's get started!  

## The many faces of a transaction

### Raw format 

In Part 2, we created the transaction and I told you the transaction looks like this:

```output
version: 1
tx_ins:
6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24:1
tx_outs:
75000:OP_DUP OP_HASH160 00f6739d5e8b4017a9eebe413249ed3949e65e24 OP_EQUALVERIFY OP_CHECKSIG
22000:OP_DUP OP_HASH160 363bb1ef1d8791bdbd7e7492ef91decc1eb7295d OP_EQUALVERIFY OP_CHECKSIG
locktime: 0
```

Actually, I lied. This is what the transaction looks like once we've cleaned up the data and made nice representations of each transaction object (i.e. TxIn, TxOut, Tx). 

If we strip away the layers, the raw transaction actually looks more like this:

```python
print(tx.serialize()) # print raw format
```
    b"\x01\x00\x00\x00\x01$\x9c\xff<\xb2\x83\x169\x8c_\x02\xe3C1\xad\xd9j^\x87\x7f\xcf\x9f\x96|\x06WRm\\\xaf\x07g\x01\x00\x00\x00jG0D\x02 `d\xf1\xcc\x90\x0e\x94p\x12\xb6\xa7%R]`\x87Gr:\xa1\x83\x91\xdd\x8a\xd6}\xdc\x94'\xa9\xd5@\x02 \t\xa2z\x0c\x9e\xdc\x15\xd1q4\x07\x12T3\xd5\x16\xad#2\x16\xe5\t\xd7}\xde34]\xc3\xafl\xbd\x01!\x02Nv\xf0\x1b\xc8\xad+\x0c\xa7u\xee\x0e9/R\xf5\xdd)\xe7y8\x8ce\x03\x04E\x92\xc5oi\xbf\xe6\xff\xff\xff\xff\x02\xf8$\x01\x00\x00\x00\x00\x00\x19v\xa9\x14\x00\xf6s\x9d^\x8b@\x17\xa9\xee\xbeA2I\xed9I\xe6^$\x88\xac\xf0U\x00\x00\x00\x00\x00\x00\x19v\xa9\x146;\xb1\xef\x1d\x87\x91\xbd\xbd~t\x92\xef\x91\xde\xcc\x1e\xb7)]\x88\xac\x00\x00\x00\x00"

Yikes! What is all this? This is the raw transaction, without any bells and whistles. This raw format is how all transaction data are stored and transmitted on the Bitcoin network. It is in the form of bytestring, essentially ASCII representation of the transaction data. In this form, the data is efficient for computer systems to process, but basically unredable for human. 

After we generate our transaction, it has to be transmitted to the Bitcoin network and propagated to the peers on the p2p network for it to be recognized and eventually mined into the blockchain. 

To prepare for this, Bitcoin converts the transaction into this raw format through a process known as **serialization**, turning objects and data into streams of bytes so that they can be stored on disk or sent over the network. When this data is needed, Bitcoin retrieves it, and through **deserialization**, makes the data readable once again. 

### Hexadecimal format 

The first step from gibberish to human-readable form is to encode the bytestring into a different, more readable format. Encoding is the process of converting data from one format to another, in this case, from bytestring to hexadecimal or hex.

Hex is a base 16 numerical system where each hex value is represented by 4 bits. Since each byte is 8 bits, a byte can be represented by 2 hex. This property makes hex popular for storing data as 2 hex = 1 byte. You may have noticed during the coding walkthrough that we included `.hex()` in many of our outputs so far. This converts bytestrings into more readable hex format.

After encoding the raw transaction data into hex, it would look like this (note I added a space after every 2 hex to make each byte more distinguishable):



```python
print(tx.serialize().hex()) # this is our transaction
```
`01 00 00 00 01 24 9c ff 3c b2 83 16 39 8c 5f 02 e3 43 31 ad d9 6a 5e 87 7f cf 9f 96 7c 06 57 52 6d 5c af 07 67 01 00 00 00 6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6 ff ff ff ff 02 f8 24 01 00 00 00 00 00 19 76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac f0 55 00 00 00 00 00 00 19 76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac 00 00 00 00`

This is the same output from our sneak preview at the end of Part 2. 

## Decoding the hex transaction

### Endianness 

Before we can decode the hex transaction, we need to learn about the **endianness** concept. Don’t let the fancy word fool you. It just means the order the system stores and transmit byte data. It can either store the most significant digit first (big endian) or the least significant digit first (little endian).

The best way to understand endianness is through an example. Take a random number, say `1234`. When you read it out loud, you say “one thousand two hundred and thirty four”. You start with the most significant value, 1000, and end with the least significant value, 4. This is known as big endian (i.e. the big end). Little endian, in contrast, operates backward, working your way up from the least significant value to the most significant value.

In conversations, big endian makes sense. Saying one thousand automatically implies there are 4 digits. But when computers transmit data, it is done one byte at a time to be more precise, so it does not know in advance how many bytes it is receiving. Therefore, computers generally process data from the least significant value first, and this also applies to the Bitcoin protocol.

When we start to decipher the raw transaction, to get the true value of some of the information, we will have to convert little endian to big endian. This is as simple as reversing each byte (2 hexes). So `8a ab 84 53` in little endian is actually `53 84 ab 8a`.

However, one confusing aspect of the Bitcoin transaction is that both big endian and little endian are used, depending on what it is being encoded. But fear not. I will show you the general principles below. 

**Remember from [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}) that every transaction has 4 sections: version, inputs, outputs and locktime. They are encoded in this order and some of the value are stored as little endian so that they need to be swapped to obtain the real values.**

### 1. Version
**Version:** `01 00 00 00`

Version is 4 bytes, stored as little endian. Once you swap endian, you get Version `1`. 

### 2. Inputs 
**Input:** `01 24 9c ff 3c b2 83 16 39 8c 5f 02 e3 43 31 ad d9 6a 5e 87 7f cf 9f 96 7c 06 57 52 6d 5c af 07 67 01 00 00 00 6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6 ff ff ff ff`

Input is the largest part of the transaction, as you can have multiple inputs. Each input references the UTXO from a previous transaction and provides the script_sig (unlocking script). 

**Input Counter:** `01`

The first byte is the input counter, telling us there is 1 input. 

**Previous tx id:** `24 9c ff 3c b2 83 16 39 8c 5f 02 e3 43 31 ad d9 6a 5e 87 7f cf 9f 96 7c 06 57 52 6d 5c af 07 67`

The next 32 bytes is the previous transaction id, stored as little endian. Once we swap endian, we end up with `67 07 af 5c 6d 52 57 06 7c 96 9f cf 7f 87 5e 6a d9 ad 31 43 e3 02 5f 8c 39 16 83 b2 3c ff 9c 24`. We can verify that this is the same [Testnet Faucet transaction](https://www.blockchain.com/btc-testnet/tx/6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24) that gave us 0.0001 tBTC in [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}). 

**Input index:** `01 00 00 00`

The next 4 bytes are the input index, telling us which output from the previous transaction we would like to spend. This is again stored in little endian. Swap endian and remove the trailing zeros and you get `1` which is the index for the *second* output (because array index starts at 0). By referencing the the previous transaction id and index, the Bitcoin protocol can find that output and infer its value. You don't need to actually specificy the value of the input. Pretty neat huh? 

**Script_sig**: `6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6`

The rest of the input script, except the last 4 bytes, give us the script_sig. The script_sig is encoded differently from everything we've seen so far, as (1) it is encoded in big endian, and (2) it makes uses of variable integers (varInts) extensively. Recall the script_sig is made up of the digital signature (r, s) values and the public key. The length of r and s values vary, so we need to tell the Bitcoin network how many bytes to expect. We do that using varInts. 

* *VarInt (length of entire script_sig):* `6a` converts to 106 bytes or 212 hex

* *VarInt (length of signature only):*  `47` converts to 71 bytes or 142 hex

The first two bytes are both varInts. The first varInt gives us the length of the entire script_sig. The second varInt gives us the length of the signature portion, which as you may recall from [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}), is stored in a DER sequence. Let's break it down. 

*DER Sequence:* `30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53` 

* `30` - signals the start of a DER encoding 
* `44` - varInt to indicate length of the DER sequence (0x44 = 68 bytes or 134 hex)
* `02` - indicates that an integer will follow. This is the **r value**
* `20` - varInt for the length of r (0x20 = 32 bytes or 64 hex) 
* The next 32 bytes are the r value
* `02` - indicates that another integer will follow. This is the **s value**
* `20` - varInt for the length of s (also 32 bytes or 64 hex)
* The next 32 bytes are the s-value

Once we decode both, we end up with (r, s) values: `(54944a0b195ffbb1d88458375a29c31fbc46fdb7cc5a9befa99d07b38ee00e10, 0da298a28c5f1f90d2e8efb625f817608ecc3d6b16da57f6cbfb46008aab8453)`

*SIGHASH flag:* `01`

The next byte after the DER sequence is the SIGHASH flag. `1` signals SIGHASH_ALL, meaning sign all inputs and outputs. 

*VarInt (Public Key length):* `21` converts to 33 bytes or 66 hex, meaning next 33 bytes are the public key. 

*Public Key:* `024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6`

Finally, we get the public key, concluding the script_sig. We can verify this is the correct public key by comparing it against the SEC compressed public key for identity #1. They are the same. 

*Sequence:* `ff ff ff ff`

The final 4 bytes are the *sequence*, which comes immediately after every input.  As we mentioned earlier, this feature is rarely used, and defaults to `ff ff ff ff`. 

If there is more than 1 input, each input will be encoded sequentially in this order, one after the other, beginning with the previous transaction id and ending with the sequence. Since there is just one input, the sequence indicates the end of all inputs and start of output. 

### 3. Outputs
Outputs are much more straightforward to decode as each output has just two elements: a value and a locking script. 

**Outputs:** `02 f8 24 01 00 00 00 00 00 19 76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac f0 55 00 00 00 00 00 00 19 76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac `

*Output counter:* `02`

The first byte is the output counter, indicating there are 2 outputs. Similar to inputs, outputs will come one after another, so the next series of bytes will give us the first output. 

*Output #1 value:* `f8 24 01 00 00 00 00 00`

The next 8 bytes are the value of the first output, stored as little endian. After swapping endian and removing the trailing zeros, you get `0124f8` which is hex for 75000. Output 1 therefore contains 75000 satoshis. 

*Output #1 varInt (length of script_pubkey):* `19` converts to 25 byte or 50 hex 

*Output #1 script_pubkey:*: `76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac`

In [Part 4](), I will show you how the script_pubkey works. For now, just know that each byte in the script_pubkey has a specific function or value, and when fully decoded, you end up with: 

`OP_DUP OP_HASH160 00f6739d5e8b4017a9eebe413249ed3949e65e24 OP_EQUALVERIFY OP_CHECKSIG`

With the script_pubkey, We just decoded output #1. 

As an exercise, see if you can decode output #2: 

*Output #2:* `55 00 00 00 00 00 00 19 76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac`

You should end up with:

* Value: 22000 satoshis 
* Script_pubkey: `76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac`


### 4. Locktime 

**Locktime:** `00 00 00 00`

The last 4 bytes are the locktime. It is currently not being used, so it always defaults to 0. 

And there you have it! The entire transaction decoded. 

## Conclusion

Being able to encode and decode a Bitcoin transaction is fundamental to building a mental model.  In the next and final chapter on the Bitcoin transaction,  we are going to complete our understanding by learning how the locking and unlocking scripts work together, introduce the Bitcoin scripting language known as Script, and learn how to transmit and propagate our transaction. 