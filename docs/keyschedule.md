# Key Schedule Record

<status>PAGE STATUS: early draft</status>

A key schedule record lists subkey information and revocation information for a
master key.

A key schedule record has kind 0x1.

A key schedule record MUST be considered invalid if it does not conform to this
specification.

A key schedule record MUST be considered invalid if the signing key and the
master key are not identical.

## Tags

Every subkey listed in a key schedule record must have an associated `Subkey`
tag listing the subkey.

## Payload

`The payload contains a sequence of 40-byte subkey records laid out as follows:

```
            1   2   3   4   4   5   6
    0   8   6   4   2   0   8   6   4
 0  +-------------------------------+
    | Subkey    1/4                 |
 8  +-------------------------------+
    | Subkey    2/4                 |
 16 +-------------------------------+
    | Subkey    3/4                 |
 24 +-------------------------------+
    | Subkey    4/4                 |
 32 +-------------------------------+
    |MARKER|RES|REVOC TIMESTAMP     |
    +-------------------------------+
```

`Marker` `[32:33]` is 1-byte and is one of the following

* 0x0 - ACTIVE_SIGNING_KEY - A Mosaic ed25519 signing key (subkey) in current use
* 0x1 - ACTIVE_ENCRYPTION_KEY - A Mosaic X25519 encryption key in current use
    * These keys are used for receiving encrypted data only, not for signing.
    * Generally the secretkey for encryption is distributed to every device
      that needs the ability to view encrypted data. Being a separate subkey
      from the signing keys, it limits the damage from compromise.
* 0x40 - REVOKED_ALL - All records signed by the key are to be considered
  invalid.
* 0x41 - REVOKED_PAST - Records signed by the key that were received prior to
  the revocation timestamp (based on when it was received by software and NOT
  based on the date inside of the record) are still considered valid; however,
  records either received after the revocation timestamp, or with a timestamp
  after the revocation timestamp, are considered invalid.
* 0x4F - OUT_OF_USE - Key is no longer in use (but nothing is revoked). This
  may be used for signing keys or encryption keys.
* 0x80 - ACTIVE_NOSTR_KEY - A nostr secp256k1 subkey
    * This helps support dual-stack software that works with both nostr and
      Mosaic.

`Res` `[33:34]` is 1-byte and is reserved. It MUST be 0.

`Timestamp` `[34:40]` is 6-bytes and is in the format described in
[timestamps](timestamps.md). Timestamp is required for REVOKED ALL and
REVOKED PAST.  Timestamp is suggested for OUT_OF_USE. Timestamp SHOULD be
zeroed in all other cases.
