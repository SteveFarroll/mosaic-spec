# Record

<status>PAGE STATUS: draft</status>

A record is a datum within Mosaic. All datums are records.

## Notation

Byte slice notation `[m:n]` indicates the bytes including `m` up to and
including the byte `n-1` but not including the byte `n`. For example `[8:12]`
represents bytes 8, 9, 10 and 11.

## Maximum Size

The maximum size of a record is 1 mebibyte (1,048,576 bytes).

This is specified so that length fields inside of records can be of a defined
fixed number of bits and so that software can make reasonable decisions about
buffer sizes.

## Layout

Note that records are laid out in a way to provide 64-bit alignment.

```text
            1   2   3   4   4   5   6
    0   8   6   4   2   0   8   6   4
 0  +-------------------------------+
    | Signature 1/8                 |
    +-------------------------------+
    | Signature 2/8                 |
    +-------------------------------+
    | Signature 3/8                 |
    +-------------------------------+
    | Signature 4/8                 |
    +-------------------------------+
    | Signature 5/8                 |
    +-------------------------------+
    | Signature 6/8                 |
    +-------------------------------+
    | Signature 7/8                 |
    +-------------------------------+
    | Signature 8/8                 |
 64 +-------------------------------+
    | Hash 1/4                      |
    +-------------------------------+
    | Hash 2/4                      |
    +-------------------------------+
    | Hash 3/4                      |
    +-------------------------------+
    | Hash 4/4                      |
 96 +-------------------------------+
    | Signing public key, 1/4       |
    +-------------------------------+
    | Signing public key, 2/4       |
    +-------------------------------+
    | Signing public key, 3/4       |
    +-------------------------------+
    | Signing public key, 4/4       |
128 +-------------------------------+
    | Author public key, 1/4        |
    +-------------------------------+
    | Author public key, 2/4        |
    +-------------------------------+
    | Author public key, 3/4        |
    +-------------------------------+
    | Author public key, 4/4        |
160 +-------------------------------+
    | Timestamp             | Nonce |
168 +-------------------------------+
    | ... Nonce     | Kind          |
176 +-------------------------------+
    | RESV1 | Len_t | Len_p         |
184 +-------------------------------+
    | RESERVED2             | FLAGS |
192 +-------------------------------+
    | Tags...                       |
    |                   .. +PADDING |
  ? +-------------------------------+
    | Payload ...                   |
    |                   .. +PADDING |
  ? +-------------------------------+
```

## Fields

Records contain the following fields.

See the [layout](#layout) section for the binary layout of these fields within
a record.

### Signature

64 bytes at `[0:64]`

The signature field is the EdDSA ed25519ph signature using the 64-byte hash at
`[64:128]`. `ph` means "pre hashed" (we will hash the content with BLAKE3 and
tell EdDSA that it is a SHA-512 hash of the message). We also provide a context
to this algorithm of b"Mosaic";

This signature is made with the Signing private key and is represented in 64
bytes.

Rationale:

* EdDSA uses a SHA-512 hashing internal to the algorithm. But BLAKE3 is a faster
  especially for longer messages, and EdDSA works just fine with it.
* We provide a context so that users cannot be tricked by one application into
  signing content for a different application (in case users think they can use
  the same keypair for every application).

### Hash

64 bytes at `[64:96]`

The hashing function is BLAKE3, unkeyed, with 256-bits (32 bytes) of output.

Note that we extend this to 512-bits with `finalize_xof()` for the input of
the EdDSA algorithm, but we only store in the event the 256-bit prefix.

This input of the hash is all the data starting at byte 96, `[96:]`,
everything except the signature and hash itself.

### Signing Public Key

32 bytes at `[96:128]`

This is the public key of the signing keypair, which is usually a subkey under
the author's master keypair (but theoretically could be delegated in some other
fashion in the future). This is represented in 32 bytes (256 bits).

### Author Public Key

32 bytes at `[128:160]`

This is the identity of the author, expressed as a public key from their master
EdDSA ed25519 keypair, which is represented in 32 bytes (256 bits).

### Timestamp

6 bytes at `[160:166]`

This is a timestamp represented in 6 bytes (48 bits) according to
[timestamps](timestamps.md).

### Nonce

6 bytes at `[166:172]`

This is represented in 6 bytes (48 bits).

These are randomly generated (except in the case of record replacement, where
they are copied from the prior record).

The purpose of the nonce is to ensure that address-based references are unique.
See [References](reference.md).

### Kind

4 bytes at `[172:176]`

This is the [kind](kinds.md) of the record which is application specific, and
determines the nature of the payload, represented in 4 bytes (32 bits) as an
unsigned integer, little-endian.

### RESV1

2 bytes at `[176:178`

These are reserved and MUST be 0.

### Len_t

1 bytes at `[178:180]` representing the length of the tags section in bytes
as an unsigned integer in little-endian format.

This represents the exact length of the tags section, not counting padding
at the end to achieve 64-bit alignment.

The maximum tags section length is 65536 bytes.

### Len_p

4 bytes at `[180:184]` representing the length of the payload section in
bytes as an unsigned integer in little-endian format.

This represents the exact length of the payload section, not counting padding
at the end to achieve 64-bit alignment.

The maximum payload section length is 1_048_384 bytes.

### RESERVED2

6 bytes at `[184:190]`

This Reserved space MUST be set to 0.

### Flags

2 bytes at `[190:192]`

* 0x01 ZSTD - The payload is compressed with Zstd
* 0x02 FROM_AUTHOR - Servers SHOULD only accept the record from the author (requiring authentication)
* 0x04 TO_RECIPIENTS - Servers SHOULD only serve the record to people tagged (requiring authentication)
* 0x08 NO_BRIDGE - Bridges SHOULD NOT propogate the record to other networks (nostr, mastodon, etc)
* 0x10 EPHEMERAL - The record is ephemeral; Servers should serve it to current subscribers and not keep it.
* All other bits - RESERVED and MUST be 0

### Tags

Varying bytes at `[192:192+Len_t]`

These are searchable key-value tags.

Unlike nostr tags, all of thsese are searchable. If an application requires
unsearchable tags, these can be defined within that application's payload.

Each tag has a 2 byte (16 bit) type and a value that is at most 253 bytes long.

Tags are laid out as follows:

```text
+-------+---------+---------+
| type  | length  | value   |
+-------+---------+---------+
```

where the type is 2 bytes (16 bits) and the length is 1 byte (8 bits) and
represents the length of the value, and the value is at most 253 bytes long.

Tags only have one value.

The tags section is padded out to 64-bit alignment.

The maximum tags section length is 65536 bytes.

Tag types are documented at [Tag Type Registry](tag_types.md) and
the tags are defined at [Core Tags](core_tags.md).

Rationale:

* Tag values should not be too large as they need to be indexed by relays.
* Constraining the value to 253 bytes allows an entire TLV (with 16-bit
  type and 8-bit length) to fit within 256 bytes.

### Payload

Varying bytes at `[192+Len_t:192+Len_t+Len_p].`

Payload is opaque (at this layer of specification) application-specific data.

The payload section is padded out to 64-bit alignment.

The maximum payload section length is 1_048_384 bytes
