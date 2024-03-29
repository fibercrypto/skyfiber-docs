Cipher is the core cryptography library for the Skycoin Project. It is designed to make cryptography easy. The library was designed to be the foundation of multiple applications and for usage beyond the coin implementation.

## Summary of Functionality

Cipher is based on secp256k1 and SHA256. It uses the same encryption algorithm as Bitcoin.

The package contains
- public/private key generation
- deterministic wallet and public/private key generation from a passphrase or seed
- serialization and deserialization of public keys to hex strings
- generation of Skycoin addresses from a pubkey
- generation of Bitcoin addresses from a pubkey [Skycoin and Bitcoin addresses are interconvertible]
- ability to sign hashes with private keys and verify signatures
- curve multiplication operation for encrypting data to a pub key.
- hashing operations (SHA256 and binary merkle-tree)
- File encryption with scrypt+chacha20poly1305

future functionality:
- Litecoin, Dogecoin and Ethereum address generation

## Background on Public Key Cryptography

Public key cryptography allows you to generate a private key and public key
- the private key is secret and only you know the private key
- the public key can be shared

Addresses in Bitcoin/Skycoin are derived from public keys. Every private key has a public key and every public key has an "address" that can receive coins. To spend the coins, you need to know the private key for the address.

There are two operations
- encryption: Anyone who knows your public key can encrypt data, so that only you can read it (or anyone who knows the secret private key).
- signatures: You can sign data with your private key and anyone can verify that only the person knowing the private key could have produced the signature. This is how Bitcoin/Skycoin transactions are authorized. The person who knows the private key for an address, "owns" the Bitcoin, in that they are able to authorize transactions by signing them.

## Skycoin private keys

Skycoin private keys are the same as Bitcoin private keys - a 256bit number on the secp256k1 curve. All 256-bit cryptographic numbers are serialized as big-endian.

## Skycoin public keys

Skycoin public keys are always 33 byte compressed public keys. The first byte is `0x03` if the `Y` point is odd, or `0x02` if the `Y` point is even.  The next 32 bytes are the big-endian `X` point.

## Skycoin Signatures

Skycoin uses compact recoverable signatures. The compactness defines its encoding and the recoverability is so the public key can be recovered from the signature itself for verification.  This is the only signature format used in Skycoin.

Skycoin signatures are 65 bytes. The first 64 bytes is the big-endian serialized `R, S` point.  Y is always a "low S" point to prevent signature malleability attacks.  The last byte is a "recovery id", which allows a public key to be recovered from the signature given an sha256 hash digest (or any 32 byte array). See [BIP146](https://github.com/bitcoin/bips/blob/master/bip-0146.mediawiki) for more details on "low S" signatures.  The recovery id is some value `0 <= r <= 4`

Skycoin does not use DER or ASN.1 encoding.

## This is how the package is used

```go
package main

import (
    "github.com/skycoin/skycoin/src/cipher"
    "fmt"     
)

func main() {
    var pub cipher.PubKey 
    var sec cipher.SecKey
  
    pub, sec = cipher.GenerateKeyPair()
}
```

Now you can sign messages, encrypt things or generate addresses for receiving coins.

## Generating Addresses and Key Pairs

Generate a new public, private key pair

```go
pub, sec := cipher.GenerateKeyPair()
```

Generate a keypair from a pass phrase

```go
pub, sec := cipher.MustGenerateDeterministicKeyPair([]byte("bip39 seed")) 
```

Generate a skycoin address for a given public key

```go
pub, sec := cipher.GenerateDeterministicKeyPair([]byte("bip39 seed")) 
addr := cipher.AddressFromPubKey(pub)
```

Get address as a base58-encoded string

```go
addrStr := addr.String()
```

Print address, its pubkey and the secret key for the address
```go
log.Printf("address=%s, pubkey=%s, seckey=%s", addr.String(), pubkey.Hex(), seckey.Hex())
```

Load an address from a base58 string
```go
address := cipher.MustDecodeBase58Address("WyPXrQpAJ7bL6kXZ9ZB6c1p3yUMhBMF7u8")
```

Load a private key from hex string
```go
seckey := cipher.MustSecKeyFromHex("f399bd1b78792da9cc49b1157c73016450c949df565ce3ddbf2f9d65fd8f0dac")
```

Load a public key from a hex string
```go
pubkey := cipher.MustPubKeyFromHex("03e56ab0597167882813864bd71305660edc128d45ed41ff583b15a44e4e95233f")
```

## Deterministic Wallet Addresses

Read the [[Deterministic Keypair Generation Method]] for the details on the deterministic keypair generator.

To generate N addresses deterministically from a wallet passphrase or seed, do
```go
n := 16 //generate 16 addresses
seed = []byte("Secret Wallet Passphrase")
secKeys := cipher.MustGenerateDeterministicKeyPairs(seed)

//lets print out the addresses for out our private keys in deterministic wallet
for index, secKey in range secKeys {
  pubKey := cipher.PubKeyFromSecKey(secKey)
  addr := cipher.AddressFromPubKey(pubKey)

  fmt.Printf("num=%s, address=%s, public_key=%s, secret_key=%s\n", index, addr.String(), pubkey.Hex(), seckey.Hex())
}
```

## Sending an Encrypted Message to Someone

To send an encrypted message

1. Get their public key
2. Generate a throw-away ephemeral key with `pubkey2, seckey2 := cipher.GenerateKeyPair()`
3. Encrypt the message with AES using key, `cipher.ECDH(pubkey, seckey2)`, where pubkey is their key
4. Append `pubkey2` to front of encrypted message so they can decrypt it.
5. Deliver the message

To decrypt an encrypted message

1. Take the first 33 bytes off the front of the message to get `pubkey2`.
2. Compute `cipher.ECDH(pubkey2, seckey)` with your seckey and the pubkey they sent
3. Decrypt the rest of the message with AES, using `cipher.ECDH(pubkey2, seckey)` as the key
4. Read the message

A future version will have a simple to use encryption/decryption wrapper. There will also be a GUI and it will be easier to use than PGP.

It will also be possible to securely message private keys through Skywire.

## Signature Operations

Sign SHA256 hash of message with private key, return signature 

```go
signature := cipher.MustSignHash(hash, seckey)
```

Verify the signature for the hash, given a pubkey

```go
err := cipher.VerifyPubKeySignedHash(pubkey, sig, hash)
if err != nil {
    log.Println("Signature invalid:", err)
}
```

## Darkwallet Addresses

A darkwallet address allows someone with your public key to generate infinite unused addresses which you can receive coins to.

For a darkwallet address, you give someone your public key. They generate a throwaway ephemeral key which is published somewhere (in the blockchain). You both do an operation and get a shared secret. The secret is used to generate a private key which only you two know. The coins are sent to the address for this private key. Both you and the sender have access to the coins and so you then move the coins to a new place.

```go
// This is your private/public key pair. You give them the public key
pubkey1, seckey1 := cipher.GenerateDeterministicKeyPair([]byte("Password"))
// This is their key pair
pubkey2, seckey2 := cipher.GenerateKeyPair()

// They communicate pubkey2, by for instance using it as the first destination for coins in a transaction
// You know pubkey2, and seckey1/pubkey1
// They know your pubkey, pubkey1 and know their seckey seckey2/pubkey2

// Your computer 
// Use your private key and the pubkey they gave you
secret1 := cipher.ECDH(pubkey2, seckey1) 

// Their computer
// They use your pubkey and their private key
secret2 := cipher.ECDH(pubkey2,seckey1)

// secret1 and secret2 are equal!
// Now you may compute an address from the them
pubkey, seckey := cipher.GenerateDeterministicKeyPair(secret1)
address := cipher.AddressFromPubkey() // send coins here
```

To review:

- You send them your public key
- They generate a new public key and send you that
- You compute ECDH with their public key and your private key
- They compute ECDH with their private key and your public key
- The values computed are equal! You now have a shared secret.
- The shared secret is used to generate an address and only you two know the private key for this address

Dark Wallet In Practice

- You give someone your public key. They can now generate infinite numbers of unused addresses, to send you coins with.
- To send you coin they generate a new pubkey, send the coins they want to send you to the address of that pubkey
- Then they spend the coins from that address into the shared secret address (the dark wallet address)
- Then you look for transactions with one input, one output. You use the public key of input address for ephemeral key.
- You compute the ECDH for your pubkey against each of these transactions and see if it yields the private-key for the output
- If the private key works, you claim the coins by moving the coins into your wallet
- If no one claims the coins after a certain period, the person may spend the coins in the shared address, back into their wallet

This achieves dark wallet transactions in three transactions, but guarantees no information is leaked. To achieve it in two transactions, you can choose a convention such as

- Using the pubkey of the address for the first input to the transaction as the ephemeral key
- Or using the public key of the Nth input for the Nth output, as the potential ephemeral key

However for this you must ensure that the same public key is not reused as the ephemeral key for the same pubkey/dark wallet address or the same address will be generated and information will leak. One solution, is to salt the secret with a hash, such as salting the ECDH secret by the hash of the unspent output consumed.

Another consideration is that dark wallet transactions should be coinjoin compatible. One minimum leakage two-transaction dark wallet protocol is as follows

- You send public key
- They create transaction with N inputs M outputs. One or more of the outputs is to your darkwallet address.
- For each unspent output consumed in the transaction, the potential ephemeral pubkeys are the pubkey of the address spending the output.
- To generate the secret, the ECDH secret is hashed with the unspent output hash. This generates a unique address always. Even if the address the coins are spent from and destination dark-wallet address are jointly repeated.
- For each transaction with N inputs, you compute the N possible addresses from the ECDH operation plus hash and check the outputs for resulting dark addresses.
- Without your private key, no one can even determine if dark-wallet transactions even exist within the transaction.
- The order of the inputs and outputs in the transaction do not matter and therefore it is coinjoin compatible without the coinjoin server doing anything.
- If there are more dark-outputs than inputs, then we can hash an existing secret to generate a chain of secrets. Then one input can be split into five outputs to dark wallets.

For instance:

- A user may take 1 input and split it into 3 outputs to your dark wallet address and 2 change outputs (sending coins back to himself)
- The transaction may be executed in a coin-join transaction so the transactions are mixed in with transactions from other users
- You now have to move the coins from the 3 shared dark-wallet output addresses into a private wallet

For privacy, it is important to note the following. If you spend the 3 shared dark-wallet outputs into the same wallet in a single transaction, it says "All the inputs from this transaction belong to the same wallet/person". So it makes senses to claim the outputs by moving the outputs one at a time, over time, in separate coin-join transaction.

If you merge coins from two addresses into a single address, it also gives away that the two addresses belong to the same wallet. However, in a coin-join transaction there are inputs and outputs from multiple users and only the coin-join server can tell which input/output belong to who. So in a coin-join merge transaction with two users, where there are 4 inputs and 2 outputs it is not clear which of the two inputs belong to each user. 

No blockchain will ever provide absolute privacy or be as anonymous as cash, because the chain of all transactions is public record and can always be traced.  Governments who have access to exchange records will be easily able to determine where money is going and more importantly are able to determine which coins have been "cleaned" before they were moved onto an exchange. However, this is a significant improvement for user privacy.

We did an information theory and statistical analysis of coin transaction patterns. In Bitcoin, enough information is leaked that anyone can identity most wallets and possibly users from the public data. In a system with minimum information leakage, governments will still be able to trace transactions from the data created at exchanges interfacing the coins to the banking system. Most coins will only go through a few transaction before ending back at an exchange (or a merchant who must report the transaction data). However, people without the government data, wont be able to trace the transactions. 

In countries where coins are banned or where regulation drives people to underground exchanges, the reporting data wont exist and there will be absolute privacy. In countries with liberal policies, but who collect data at the exchange, they may not always be able to identify the full transaction path. However, they will be able to determine which users are suspiciously effective at protecting their transaction privacy and whose coins are suspiciously clean. Coinbase is already banning users for receiving coins that have gone through tumblers or have been used on websites associated with drugs.

Mandatory or default coin-join would improve privacy but create a situation, where any user could be arbitrarily targeted for raids or harassment because effectively all coins would become tainted with immoralities the recipient would otherwise take care to distance himself from. Therefore it makes sense to allow for some separation between coins used for legitimate commerce and coins which may have been tainted.

When a darknet site like Silk Road is raided, in the future instead of taking it down, they will seize the servers and continue operating the service. They will record the transactions and trace them back to Coinbase, BitPay shipping addresses and potentially do global SWAT raids on tens of thousands of people across hundreds of cities. The risk of allowing even a 30% false positive rate from coin mixing is significant.

Police are seizing Bitcoin and auctioning them off for money before trial. They do not have to charge the person to seize and sell the Bitcoin. Police have an incentive to go after the people with the most money to seize and people with the most activity. So even though privacy can be enhanced at a technical level, enabling that functionality as the default creates social problems that offset the advantages.

In the long-term, dark wallet addresses will not be necessary. Communication addresses over Skywire, will give an off-blockchain channel for communicating new addresses and receipts for each transaction.

## Encrypting Files

https://github.com/skycoin/skycoin/tree/master/src/cipher/encrypt

### Key derivation

Key derivation is performed with scrypt.

A user-provided password and random 32-byte salt is converted into an encryption key using scrypt.
The `r` parameter is defaulted to 8 and the `p` parameter is defaulted to 1.
The scrypt work factor `N` is defaulted to `1 << 20`, which is chosen
based upon modern desktop CPU speeds. 
As of 2017, key derivation takes 3-4 seconds on common desktop CPUs with this work factor.
If used in another context, such as a mobile device, the work factor should be reduced for usability,
at the risk of being easier to crack.
The default work factor will be increased as needed over time.

Note that if you encrypt your wallet and store the encrypted wallet somewhere,
eventually performance improvements in CPUs can make your encrypted file more
vulnerable to brute force attacks.  The work factor of `1 << 20` should be good until
at least 2020.

### Encryption

File encryption is performed with chacha20-poly1305. 

After the key is derived from the user password and the random salt, it used for encryption
with chacha20-poly1305. The nonce is randomly generated each time the file is encrypted.

## Hardware Devices

We believe that encryption/decryption and signature operations should be handled by a hardware device if possible. This ensures safety of coins and private keys, even if the computer is hacked or compromised.

In the long term, we plan to fund the development of such devices (a five dollar, 32 bit ARM processor the size of USB thumbstick). We are planning an interface to the encryption library which transparently handles signing operations from external key-storage devices.

## Example Application For Public Key Cryptography 

An example application, is that you want to save files so you can read them later but you dont want someone who steals your computer to be able to access them. For instance, you have a video camera that is producing 1 image per second and saving it to disc.

If symmetric encryption was used, the server would need to store the private key to encrypt the images. The person stealing the server would also be stealing the private key and would be able to view the images. 

- If the computer was shutdown and the private key was stored only in RAM, then the private key would be lost if the computer was unplugged. However, the private key would need to be re-entered every time the computer booted (which is a waste of time). This is how disc encryption works currently.
- However, a prepared attacker could hack the live system with a USB or firewire device and acquire the private key before seizing the server.

Using pub key encryption you would generate a public/private key deterministically from a pass phrase. Then you only store the public key on the server (on disc).
- Each image is encrypted with the public key, so that only the person with the private-key can read the images. 
- If the the server is stolen, nothing is recoverable without the private key. However, the server can still encrypt new files.
- The public key can safely be stored on disc, so the computer can be rebooted and still operate automatically.

To implement this, you would
- generate a public/private key pair
- write program that loads the public key from disc as a string and looks in a directory for images, reads in new images, encrypts them for the public key, renaming the images
- write program that takes in the private key to decrypt the files

This library is designed to make implementing a protocol like this very easy.