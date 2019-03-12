# Universal Offline Signatures

**THIS IS A DRAFT**

This document proposes a standard for QR code encoding that enables two-way communication between a _Hot Wallet_ and a _Cold Signer_ with access to private keys, for the purpose of securely signing and broadcasting transactions and data on current and future decentralized networks (including non-Ethereum networks). The goal is have a single, inter-operable standard, allowing users to use any combination of a hot wallet and a cold signer without vendor locking.

## Design principles

+ **Concise** - a single QR code can represent up to 23624 (±4) bits of information. The more data that must be represented, the denser the code becomes, making it problematic to use with cheaper hardware, therefore packing as little data as possible per QR code is the pragmatic thing to do.
+ **Unambiguous** - there must be one, and only one, way for the payload to be signed to be interpreted by a correct implementation following this standard.
+ **Extensible** - it should be possible to add support for new networks and new cryptography on existing networks (should such need emerge) in the future, without breaking backwards compatibility.

## QR code encoding

The common ways to encode binary data in a QR code would include:

+ Hexadecimal representation with Alphanumeric QR encoding: 37.5% overhead.
+ Hexadecimal ASCII representation with Binary QR encoding: 100% overhead.
+ Base64 ASCII representation with Binary QR encoding: ~83.3% overhead.
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

## Steps

Since this is a multi-step process, we will differentiate between the following types of QR codes:

| Step | Name             | Direction  | Contains                            | [QR Encoding](https://en.wikipedia.org/wiki/QR_code#Storage) |
|------|------------------|------------|-------------------------------------|--------------------------------------------------------------|
| 0⁽¹⁾ | **Introduction** | Cold ⇒ Hot | Network identification and Address  | Binary (UTF-8)                                               |
| 1    | **Payload**      | Cold ⇐ Hot | Data to sign prefixed with metadata | Binary                                                       |
| 2    | **Signature**    | Cold ⇒ Hot | Signature for **Payload**           | Binary                                                       |

+ ⁽¹⁾ Step 0 is optional as it is only necessary if the Hot Wallet doesn't yet know the address which it must use in Step 1.

### *Introduction* Step

The goal of this step is for Cold Signer to inform the Hot Wallet about a single account it has access to. To make this useful outside of the scope of this specification, this standard proposes using URI format compatible with [EIP-681](https://eips.ethereum.org/EIPS/eip-681) and [EIP-831](https://eips.ethereum.org/EIPS/eip-831), with syntax:

```
introduction    = scheme ":" address
scheme          = STRING
address         = STRING
```

The `address` format depends on the `scheme`. For Ethereum and Ethereum-based networks `scheme` is always `"ethereum"`, while the address is `"0x"`-prefixed hexadecimal. A correct Introduction for address zero (`0x0000000000000000000000000000000000000000`) on Ethereum is therefore a string:

```
ethereum:0x0000000000000000000000000000000000000000
```

+ `scheme` **MUST** be valid ASCII, beginning with a letter and followed by any number of letters, numbers, the period `.` character, the plus `+` character, or the hyphen `-` character.
+ `address` **MUST** be valid UTF-8, appropriate for a given network.
+ Cold Signer **MUSTN'T** add any other information other than `scheme` and `address` to the string.
+ Hot Wallet **MAY** be able to read other information than required (such as is defined in EIP-681).
+ Hot Wallet **MAY** support any number of schemes/networks following this syntax.
+ For unsupported schemes/networks Hot Wallet **MUST** show the user an informative error, distinct from parsing failure, eg: `"Scheme {scheme} is not supported by {wallet name}"`.

### *Payload* Step.

Payload is always read left-to-right, using prefixing to determine how it needs to be read. The first prefix is single byte at index `0`:

| `[0]`     | `[1..]`                     |
|-----------|-----------------------------|
| `00`      | **Multipart Payload**       |
| `01`      | Ethereum Payload            |
| `02`      | Substrate Payload           |
| `03...7A` | Reserved for other networks |
| `7B`      | Legacy Ethereum Payload     |
| `7C...FF` | Reserved                    |

#### *Multipart Payload*

QR codes can only represent 2953 bytes, which is a harsh constraint as some transactions, such as contract deployment, may not fit into a single code. Multipart Payload is a way to represent a single Payload as a series of QR codes. Each frame of a Multipart Payload follows fits looks as follows:

| `[0]`  | `[1..3]` | `[3..5]`      | `[5..]`     |
|--------|----------|---------------|-------------|
| `00`   | `frame`  | `frame_count` | `part_data` |

+ `frame` is the number of current frame, represented as big-endian 16-bit unsigned integer.
+ `frame_count` is the total number of frames, represented as big-endian 16-bit unsigned integer.
+ `part_data` is data that should be stored by the Cold Signer, ordered by `frame` number, until all frames are scanned.
+ Hot Wallet **SHOULD** continuously loop through all the frames showing each frame for 2 seconds.
+ Cold Signer **MUST** be able to start scanning the Multipart Payload _at any frame_.
+ Cold Signer **MUSTN'T** expect the frames to come in any particular order.
+ Cold Signer **SHOULD** show a progress indicator of how many frames it has successfully scanned out of the total count.
+ `part_data` for frame `0` **MUSTN'T** begin with `nul` byte `00`.

Once all frames are combined, the `part_data` must be concatenated into a single binary blob, and then interpreted as a completely new, albeit larger, Payload, starting from the prefix table above.

### Ethereum Payload

Ethereum Payload follows the table:

| Action             | `[0]` | `[1]` | `[2..22]` | `[22..]`  |
|--------------------|-------|-------|-----------|-----------|
| Sign a transaction | `01`  | `01`  | `address` | `rlp`     |
| Sign a message     | `01`  | `02`  | `address` | `message` |
| Sign a hash        | `01`  | `03`  | `address` | `hash`    |
