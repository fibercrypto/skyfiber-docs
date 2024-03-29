## How to Validate a Skycoin Address

**Golang:**
- Use the `cipher.DecodeBase58Address` from `package cipher`: https://godoc.org/github.com/skycoin/skycoin/src/cipher#DecodeBase58Address

**Python:**
- Use [pyskycoin](https://github.com/skycoin/pyskycoin). The function is `skycoin.cipher__DecodeBase58Address`.

**Javascript/other languages:**
- You can make a `POST` request to https://node.skycoin.com/api/v2/address/verify (or use any Skycoin API node of your choice). See https://github.com/skycoin/skycoin/blob/develop/src/api/README.md#verify-an-address for the API docs
- Make language bindings to [libskycoin](https://github.com/skycoin/libskycoin)
- Construct a validator in the native language (requires these libraries: sha256, ripemd160, base58).

## Differences between Skycoin and Bitcoin addresses

Skycoin addresses are similar version 1 Bitcoin addresses, with these differences:

- The Skycoin address hash is calculated as `ripemd160(sha256(sha256(pubkey)))` whereas the Bitcoin address hash is `ripemd160(sha256(pubkey))`
- The Skycoin address version byte is placed last, whereas the Bitcoin address version byte is placed first
- The Skycoin address checksum is calculated as `sha256(address+version)`, whereas the Bitcoin address checksum is calculated as `sha256(sha256(version+address))`

## Technical background

*The following is adapted from https://en.bitcoin.it/wiki/Technical_background_of_version_1_Bitcoin_addresses*

A Skycoin address is a 160-bit hash of the public portion of a public/private [ECDSA](http://en.wikipedia.org/wiki/Elliptic_Curve_DSA) keypair. Using [public-key cryptography](http://en.wikipedia.org/wiki/Public-key_cryptography), you can "sign" data with your private key and anyone who knows your public key can verify that the signature is valid.

A new keypair is generated for each receiving address, deterministically from the wallet's seed.
The public key, private key, address and the seed needed to generate them are stored in the wallet data file (saved in `~/.skycoin/wallets` by default).  The seed and private keys are encrypted if the wallet is encrypted, but public data is left visible. If you lose your seed and the wallet file or you forget the password to your encrypted wallet, then you cannot recover your coins.  Best practice is to retain the seed.  It is ok to lose the wallet file as long as you have the seed.

Skycoin allows you to create as many addresses as you want.

Skycoin addresses contain a built-in check code (checksum), so it's generally not possible to send Skycoins to a mistyped address. However, it is possible that the address is mistyped but still valid. If no one owns it, any coins sent to that address will be lost forever.

Hash values and the checksum data are converted to an alpha-numeric representation using a custom scheme: the Base58Check encoding scheme. Under Base58Check, addresses can contain all alphanumeric characters except 0, O, I, and l. Due to the base58 encoding scheme, addresses can be 25-34 characters in length, but most addresses are 33 or 34 characters long.  The underlying byte representation of an address is still a fixed length of 25 bytes (20 bytes for the ripemd160 hash, 1 byte for the version, 4 bytes for the checksum).

## How to create a Skycoin address

0. Having a private ECDSA key

    `7e4c9fbbbbb21be96fd496653b81036bab6e599a10557c790d1aa4d29c461ab9`

1. Take the corresponding public key generated with it

    `03a4e90c34b9f359364b01f2136f1ff850d648d56da2f4b4aed6cbb6fede346831`

2. Perform SHA-256 hashing on the public key

    `e6958adc2f9f3260b269d14842bde2e3da9dad93d310c9dab547a547e60fe885`

3. Perform SHA-256 hashing on the result of the previous SHA-256 hash

    `a8015a9b10fd68c55539a4fee83e9976b0efe808c6666780f9d30ae8c80be1fd`

4. Perform RIPEMD-160 hashing on the result of SHA-256

    `d690c4ca95e00522b34d8c5d4f618a0255f370df`

5. Add version byte at the end of the RIPEMD-160 hash (version is `0x00` in this example)

    `d690c4ca95e00522b34d8c5d4f618a0255f370df00`

6. Perform SHA-256 hash on the extended RIPEMD-160 result

    `d643b43ea9a97c0840c32e95a1d967b8fa3f5345207b72e5cc3714622f777dd5`

7. Take the first 4 bytes of the previous SHA-256 hash. This is the address checksum

    `d643b43e`

8. Add the 4 checksum bytes from stage 7 at the end of extended RIPEMD-160 hash from stage 4. This is the 25-byte binary Bitcoin Address.

    `d690c4ca95e00522b34d8c5d4f618a0255f370df00d643b43e`

9. Convert the result from a byte string into a base58 string

    `2VLXYmAWYnX5MMCfRwAvt46YyEk2X7TvrS9`

*These values were generated by this script: https://gist.github.com/gz-c/7f9018a8742133280685671bfcff80b6*
