# Universal Offline Signatures

**THIS IS A DRAFT**

This document proposes a standard for QR code encoding that enables two-way communication between a _Hot Wallet_ and a _Cold Signer_ with access to private keys, for the purpose of securely signing and broadcasting transactions and data on current and future decentralized networks (including non-Ethereum networks). The goal is have a single, inter-operable standard, allowing users to use any combination of a Hot Wallet and a Cold Signer (defined below) without vendor locking.

## Design principles

+ **Concise** - a single QR code can represent up to 23624 (±4) bits of information. The more data that must be represented, the denser the code becomes, making it problematic to use with cheaper hardware, therefore packing as little data as possible per QR code is the pragmatic thing to do.
+ **Unambiguous** - there must be one, and only one, way for the payload to be signed to be interpreted by a correct implementation following this standard.
+ **Extensible** - it should be possible to add support for new networks and new cryptography on existing networks (should such need emerge) in the future, without breaking backwards compatibility.

## [QR code encoding](https://en.wikipedia.org/wiki/QR_code#Storage)

The common ways to encode binary data in a QR code would include:

+ Base64 ASCII representation with Binary QR encoding: ~33.3% overhead.
+ Hexadecimal representation with Alphanumeric QR encoding: 37.5% overhead.
+ Hexadecimal ASCII representation with Binary QR encoding: 100% overhead.
+ Native Binary QR encoding: *no overhead*.

For data density and simplicity **this standard will only use the native Binary QR encoding**.

_Note:_ Base64 ASCII representation with Alphanumeric QR encoding is impossible, as Alphanumeric QR code only permits 44 (5½ bits per character) out of the required 64 characters (6 bits per character).

## Nomenclature

Since this technology requires two separate devices/applications, to avoid confusion the following names will be used to differentiate the two:

+ **Hot Wallet** - an application running on an Internet connected device that has the ability to show and scan QR codes, produce and encode transactions or data to be signed, and broadcast them to appropriate network.
+ **Cold Signer** - a device or an application running on a dedicated device without Internet access that has the ability to show and scan QR codes, securely store private keys, decode transactions or data to be signed, and sign them.

For describing binary data this standard uses either a single byte index `[n]`, an open left-inclusive range `[n..]`, or a closed left-incluse right-exclusive range `[n..m]`. `[..n]` is a shorthand for `[0..n]`. Examples:

+ `[3]` is a single byte at index `3`.
+ `[0..5]` is 5 bytes at following indexes: `0`, `1`, `2`, `3`, and `4`.
+ `[..5]` is also 5 bytes at following indexes: `0`, `1`, `2`, `3`, and `4`.
+ `[7..7]` would be a zero-length range and contain no bytes.
+ `[10..]` would be all bytes starting from index `10` till the end.

For byte values this standard uses either a single hexadecimal value `AA`, or a range `AA...BB`, which is left and right inclusive:

+ `00` is a single ASCII `nul` byte.
+ `61...7A` is a range including all lowercase ASCII letters `a` to `z`.

Additionally we will define the following terms to mean:

+ **MUST** and **MUST NOT** - expected behavior, breaking which break compatibility with this standard.
+ **SHOULD** and **SHOULD NOT** - expected behavior although more fuzzily defined and breaking of which does not break compatibility with this standard.
+ **MAY** - behavior that is not part of this standard, but is allowed without breaking compatibility.

## Steps

Since this is a multi-step process, we will differentiate between the following types of QR codes:

| Step | Name                                   | Direction  | Contains                            | QR Encoding    |
|------|----------------------------------------|------------|-------------------------------------|----------------|
| 0⁽¹⁾ | [**Introduction**](#introduction-step) | Cold ⇒ Hot | Network identification and Address  | Binary (UTF-8) |
| 1    | [**Payload**](#payload-step)           | Cold ⇐ Hot | Data to sign prefixed with metadata | Binary         |
| 2    | [**Signature**](#isgnature-step)       | Cold ⇒ Hot | Signature for **Payload**           | Binary         |

+ ⁽¹⁾ Step 0 is optional as it is only necessary if the Hot Wallet doesn't yet know the address which it must use in Step 1.

---

### *Introduction* Step

The goal of this step is for Cold Signer to inform the Hot Wallet about a single account it has access to. To make this useful outside of the scope of this specification, this standard proposes using URI format compatible with [EIP-681](https://eips.ethereum.org/EIPS/eip-681) and [EIP-831](https://eips.ethereum.org/EIPS/eip-831), with syntax:

```
introduction    = scheme ":" address
scheme          = STRING
address         = STRING
```

The `address` format depends on the `scheme`.

+ `scheme` **MUST** be valid ASCII, beginning with a letter and followed by any number of letters, numbers, the period `.` character, the plus `+` character, or the hyphen `-` character.
+ `address` **MUST** be valid UTF-8, appropriate for a given network.
+ Cold Signer **MUST NOT** add any other information other than `scheme` and `address` to the string.
+ Hot Wallet **MAY** be able to read other information than required (such as is defined in EIP-681).
+ Hot Wallet **MAY** support any number of schemes/networks following this syntax.
+ For unsupported schemes/networks Hot Wallet **MUST** show the user an informative error, distinct from parsing failure, eg: `"Scheme {scheme} is not supported by {wallet name}"`.

#### Ethereum Introduction

+ `scheme` **MUST** be a string `ethereum`.
+ `address` **MUST** be a hexadecimal string representation of the address.
+ `address` **MUST** be prefixed with `0x`

A correct Introduction for address zero (`0x0000000000000000000000000000000000000000`) on Ethereum is therefore a string:

```
ethereum:0x0000000000000000000000000000000000000000
```

#### Substrate Introduction

+ `scheme` **MUST** be a string `substrate`.
+ `address` **MUST** be base58 representation of the address.

A correct Introduction for address `5GKhfyctwmW5LQdGaHTyU9qq2yDtggdJo719bj5ZUxnVGtmX` on a Substrate-based network is therefore a string:

```
substrate:5GKhfyctwmW5LQdGaHTyU9qq2yDtggdJo719bj5ZUxnVGtmX
```

---

### *Payload* Step.

Payload is always read left-to-right, using prefixing to determine how it needs to be read. The first prefix is single byte at index `0`:

| `[0]`     | `[1..]`                                             |
|-----------|-----------------------------------------------------|
| `00`      | [**Multipart Payload**](#multipart-payload)         |
| `01`      | [Ethereum Payload](#ethereum-payload)               |
| `02`      | [Substrate Payload](#substrate-payload)             |
| `03...7A` | Extension range for other networks                  |
| `7B`      | [Legacy Ethereum Payload](#legacy-ethereum-payload) |
| `7C...7F` | Extension range for other networks                  |
| `80...FF` | Reserved                                            |

#### *Multipart Payload*

QR codes can only represent 2953 bytes, which is a harsh constraint as some transactions, such as contract deployment, may not fit into a single code. Multipart Payload is a way to represent a single Payload as a series of QR codes. Each QR code in Multipart Payload, or _a frame_, looks as follows:

| `[0]`  | `[1..3]` | `[3..5]`      | `[5..]`     |
|--------|----------|---------------|-------------|
| `00`   | `frame`  | `frame_count` | `part_data` |

+ `frame` **MUST** the number of current frame, represented as big-endian 16-bit unsigned integer.
+ `frame_count` **MUST** the total number of frames, represented as big-endian 16-bit unsigned integer.
+ `part_data` **MUST** be stored by the Cold Signer, ordered by `frame` number, until all frames are scanned.
+ Hot Wallet **MUST** continuously loop through all the frames showing each frame for about 2 seconds.
+ Cold Signer **MUST** be able to start scanning the Multipart Payload _at any frame_.
+ Cold Signer **MUST NOT** expect the frames to come in any particular order.
+ Cold Signer **SHOULD** show a progress indicator of how many frames it has successfully scanned out of the total count.
+ `part_data` for frame `0` **MUST NOT** begin with byte `00` or byte `7B`.

Once all frames are combined, the `part_data` must be concatenated into a single binary blob, and then interpreted as a completely new albeit larger Payload, starting from the prefix table above.

#### Ethereum Payload

Ethereum Payload follows the table:

| Action             | `[0]` | `[1]` | `[2..22]` | `[22..]`  |
|--------------------|-------|-------|-----------|-----------|
| Sign a hash        | `01`  | `00`  | `address` | `hash`    |
| Sign a transaction | `01`  | `01`  | `address` | `rlp`     |
| Sign a message     | `01`  | `02`  | `address` | `message` |

+ `address` **MUST NOT** have any prefixes.
+ `address` **MUST** be exactly 20 bytes long.
+ `address` **MUST** be represented as a binary byte string, **NOT** hexadecimal.
+ `rlp` **MUST** be the [RLP](https://github.com/ethereum/wiki/wiki/RLP) encoded raw transaction with an empty signature being set in accordance with [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md): `v = CHAIN_ID`, `r = 0`, `s = 0`.
+ `message` **MUST** be a binary or UTF-8 encoded message to sign **WITHOUT** any prefixes ([EIP-191](https://eips.ethereum.org/EIPS/eip-191) or otherwise).
+ `hash` **MUST** be a valid `keccak256` hash of either a transaction or a correctly prefixed message.
+ Hot Wallet **SHOULD** always prefer sending either a full raw transaction or full message instead of a hash to sign, so that the user can verify that the the Cold Signer is signing what the Hot Wallet presented them with. Occasionally this might be completely impractical (the message or the transaction is megabytes long and not suitable for Multipart Payload).
+ Cold Signer **SHOULD** decode the transaction details from the RLP and display them to the user, so that they can verify that the transaction hasn't been altered by the Hot Wallet.
+ Cold Signer **SHOULD** attempt to decode the `message` as UTF-8 encoded human readable string by whatever heuristics it finds suitable and display it to the user, so that the user can verify that the message hasn't been altered by the Hot Wallet.
+ Cold Signer **SHOULD** warn the user that signing a hash is inherently insecure, because there is no easy way for the user to verify whether they are signing what they intended to sign.
+ Hot Wallet **SHOULD** have a way to show [Legacy Ethereum Payload](#legacy-ethereum-payload) at user request.
+ Cold Signer **SHOULD** display all account id values in SS58Check encoding.

TODO: Handle [EIP-712](https://eips.ethereum.org/EIPS/eip-712) typed data.

#### Substrate Payload

Substrate Payload follows the table:

| Action                       | `[0]` | `[1]`  | `[2]`  | `[1..1+L]`  | `[1+L..]`                  |
|------------------------------|-------|--------|--------|-------------|----------------------------|
| Sign a transaction           | `02`  |`crypto`|  `00`  | `accountid` | `payload`                  |
| Sign a transaction           | `02`  |`crypto`|  `01`  | `accountid` | `payload_hash`             |
| Sign an immortal transaction | `02`  |`crypto`|  `02`  | `accountid` | `immortal_payload`         |
| Sign a message               | `02`  |`crypto`|  `03`  | `accountid` | `message`                  |


+ `crypto` **MUST** be a recognised cryptographic algorithm. It implies the value of the `accountid` length, `L`. This **MUST** be one byte whose value is one of:
  - `0x00`: Ed25519 (`L = 32`)
  - `0x01`: Schnorr/Ristretto x25519 (`L = 32`)
+ `accountid` **MUST** be exactly `L` bytes long.
+ `accountid` **MUST** be represented as a binary byte string, **NOT** hexadecimal.
+ `payload` **MUST** be the SCALE encoding of the tuple of transaction items `(nonce, call, era_description, era_header)`.
+ `payload_hash` **MUST** be the Blake2s 32-byte hash of the SCALE encoding of the tuple of transaction items `(nonce, call, era_description, era_header)`.
+ `immortal_payload` **MUST** be the SCALE encoding of the tuple of transaction items `(nonce, call)`.
+ Hot Wallet **MUST** use type `00` for signing a standard transaction type if the length of the `payload` is 256 bytes or fewer.
+ Hot Wallet **SHOULD** always prefer using type `00` even if the length of the payload is greater than 256 bytes since this allows the full payload to be provided and decoded for the user. If doing that is completely impractical (the message or the transaction is megabytes long and not suitable for Multipart Payload), type `01` may be used alternatively.
+ Cold Signer **SHOULD** decode the transaction details from the SCALE encoding and display them to the user for verification before signing.
+ Cold Signer **SHOULD** attempt to decode the `message` as UTF-8 encoded human readable string by whatever heuristics it finds suitable and display it to the user for verification before signing.
+ Cold Signer **SHOULD** warn the user that signing a hash is inherently insecure, in the cash of type `01`.
+ Cold Signer **SHOULD** (at the user's discretion) sign the `message`, `immortal_payload`, or `payload` if `payload` is of length 256 bytes or fewer. If `payload` is longer than 256 bytes, then it **SHOULD** instead sign the Blake2s hash of `payload`.

#### Legacy Ethereum Payload

Byte `7B` is the ASCII byte representing open curly brace `{`, for that reason it's treated as a prefix for older, deprecated format. This Payload should be decoded in full as UTF-8 encoded JSON, following either of the two variants:

```json
{
  "action": "signTransaction",
  "data": {
    "account": ADDRESS,
    "rlp": RLP
  }
}
```

or

```json
{
  "action": "signData",
  "data":{
    "account": ADDRESS,
    "data": MESSAGE
  }
}
```

+ `ADDRESS` **MUST** be a hexadecimal string representation of the address, exactly 40 characters long.
+ `ADDRESS` **MUST NOT** include the `0x` prefix.
+ `RLP` **MUST** be a hexadecimal string representation of the [RLP](https://github.com/ethereum/wiki/wiki/RLP) encoded raw transaction with an empty signature being set in accordance with [EIP-155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md): `v = CHAIN_ID`, `r = 0`, `s = 0`.
+ `RLP` **MUST NOT** include the `0x` prefix.
+ `DATA` **MUST** be a hexadecimal string representation of a binary or UTF-8 encoded message to sign **WITHOUT** any prefixes ([EIP-191](https://eips.ethereum.org/EIPS/eip-191) or otherwise).
+ `DATA` **MUST NOT** include the `0x` prefix.
+ All _**SHOULD**s_ from [Ethereum Payload](#ethereum-payload) apply here as well.
+ Legacy Ethereum Payload does not support signing raw hashes.

---

### *Signature* Step

Signatures will vary on type of payload that is being signed.

#### Ethereum Signature

Ethereum signature must follow one of the two following formats:

| `[0]` | `[1..33]` | `[33..65]` | `[66]` |
|-------|-----------|------------|--------|
| `01`  | `r`       | `s`        | `v`    |

or

| `[0..64]` | `[64..128]` | `[128..130]` |
|-----------|-------------|--------------|
| `HEX_R`   | `HEX_S`     | `HEX_V`      |

+ Cold Signer **SHOULD** prefer the first format as it's more concise.
+ Hot Wallet **MUST** first check byte length and assume second format if length equals `130`.
+ Hot Wallet **MUST** support both formats.
+ `r` **MUST** be binary `r` value of the Secp256k1 signature for the signed Payload.
+ `s` **MUST** be binary `s` value of the Secp256k1 signature for the signed Payload.
+ `v` **MUST** be binary `v` value of the Secp256k1 signature for the signed Payload.
+ `HEX_R` **MUST** be a hexadecimal representation of `r` value of the Secp256k1 signature for the signed Payload.
+ `HEX_S` **MUST** be a hexadecimal representation of `s` value of the Secp256k1 signature for the signed Payload.
+ `HEX_V` **MUST** be a hexadecimal representation of `b` value of the Secp256k1 signature for the signed Payload.
+ `HEX_R`, `HEX_S`, and `HEX_V` **MUST NOT** be prefixed with `0x`.
+ `v` and `HEX_V` **MUST NOT** be combined with `CHAIN_ID`.
+ Hot Wallet **MUST** fold `CHAIN_ID` into the `v` value when constructing final transaction RLP.

Pseudocode for folding in `CHAIN_ID` into `v`:

```
if chainId > 0 {
    v += (chainId * 2 + 8) & 0xFF;
}
```

#### Substrate Signature

TODO

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
