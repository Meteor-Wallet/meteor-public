# Meteor Wallet: How Local Account Data Is Protected

This document describes, in as much detail as is useful, how Meteor Wallet encrypts and
authenticates the wallet data it stores on a user's device. It is written for users who want to
understand what protects their keys, for security researchers reviewing our design, and for
users who are attempting an offline recovery of their own wallet data.

Everything here follows one principle: the design must remain secure even when the design is
public. Nothing in this document is a secret. The only secrets are the user's password, the
user's recovery phrase, and keys that live inside the device's hardware-backed keystore.

Scope: this document covers the **wallet unlock and account encryption** layer. It does not cover
network transport, dApp connections, or server-side systems. It also deliberately omits internal
storage key names, module layouts, and other implementation details that are not part of the
cryptographic design.

---

## 1. Summary

- Meteor Wallet is **self-custodial**. Private keys are generated on the device and never leave
  it in plaintext. Meteor has no copy of any user's password, keys, or recovery phrase, and has
  no ability to reset a password or recover a wallet on a user's behalf.
- Each account's signing material is stored as an **AES-256-GCM** encrypted blob. The encryption
  key is derived from the user's chosen unlock method (password or device biometrics).
- A small **authentication check token** is encrypted with the same key so the app can verify a
  password before touching account data.
- On iOS and Android, all of the above is additionally wrapped in an encrypted on-device store
  whose key is held in the platform keychain / keystore.
- The scheme has been stable since password authentication was introduced. App release
  **0.0.135** (iOS build 145) uses exactly the scheme described here.

---

## 2. Storage layers

Meteor stores a single local user record per installation. That record contains the account list
(public data such as account IDs and public keys), the encrypted signing payload for each
account, the encrypted authentication check token, and a small descriptor of which unlock method
is active. What protects that record depends on the platform.

### 2.1 iOS and Android (native app)

| Layer | What it is | Key location |
| --- | --- | --- |
| Outer | Encrypted key-value store (MMKV with encryption enabled) holding the whole local record | Random 32-byte key generated on first launch, stored in the iOS Keychain / Android Keystore with "accessible when unlocked" protection |
| Inner | Per-account AES-256-GCM payloads and the auth check token, as described in section 4 | Derived from the user's unlock method (section 3) |

The outer layer defends against casual inspection of the app sandbox and against other
processes on a compromised or jailbroken device. It is **not** a substitute for a password: the
outer key is by design readable by the app itself, and therefore by anyone who can fully
decrypt an encrypted device backup that includes keychain items. This is why the inner layer
exists and why the inner layer never depends on the outer one.

The outer store key is generated with the platform's cryptographically secure random source and
is never derived from user input. There is no way to regenerate it; if it is lost, the outer
store is unreadable and the wallet must be restored from the recovery phrase.

### 2.2 Web app and browser extension

The web app stores the local record in the browser's `localStorage`; the extension uses the
extension's private `chrome.storage.local` area. Neither adds an encryption layer of its own.
On these platforms the inner AES-256-GCM layer is the **only** protection for signing material,
which is why a password is strongly recommended there.

---

## 3. Unlock methods

A wallet has exactly one active unlock method at a time. Changing the method re-encrypts every
account payload and the auth check token under the new method. Each method's job is to produce a
**cipher string**, which is the input to the encryption scheme in section 4.

### 3.1 No unlock method set ("auto cipher")

Used when a user has not set a password. A random 32-byte value is generated on the device,
base64-encoded, and used directly as the cipher string. That value is stored in the local record
alongside the encrypted data, so on this setting the inner encryption provides **no protection
beyond the platform storage layer**. It exists so that all data follows the same encrypted
format regardless of the user's choice, and so that a password can be added later without
changing the data model.

### 3.2 Password

The user chooses a password of at least 8 characters. A strength meter is shown, but only the
length rule is enforced. The password is transformed into the cipher string as follows:

1. Encode the password as **UTF-8** bytes. There is **no normalization** of any kind: no
   trimming, no case folding, no Unicode NFC/NFD normalization, and no maximum length. The
   bytes are exactly what the text input produced.
2. Compute **SHA-256** over those bytes.
3. Encode the digest as **lowercase hexadecimal** (64 ASCII characters). This string is the
   cipher string.

The password itself is never stored anywhere, and neither is the SHA-256 digest (except
transiently in memory while the wallet is unlocked, and in the biometric case described next).

### 3.3 Device biometrics

Biometric unlock is only available on native platforms that report biometric capability.

- **Biometrics v2 (current):** the cipher string is the same SHA-256 hex string produced from the
  user's password (section 3.2). That string is stored in the platform keychain / keystore as an
  item that **requires user presence** (Face ID, Touch ID, or the device passcode fallback the OS
  provides) on every read. Because it is the same cipher, a user can always fall back to typing
  their password, and enabling or disabling biometrics does not re-encrypt any data.
- **Biometrics v1 (legacy):** in releases before September 2026, biometrics could be enabled
  on its own, without a password. An independent random 32-byte value was generated, stored in
  the keychain with the same user-presence requirement, and used directly as the cipher string.
  Wallets still on v1 keep working; new biometric setups use v2 and require a password.

Keychain items that require user presence are bound to the device's secure hardware and to the
biometric enrollment that existed when they were created. They cannot be read by an offline
tool, and they are not usable on any other device even if a backup carries a copy of the item.
The consequences differ by version:

- A wallet on **v2** always has a password behind the biometric, so it remains recoverable with
  that password (section 6) even if biometrics stop working.
- A wallet on **v1** with no password set can only be unlocked on the original device with the
  original biometric enrollment. If that is lost, the recovery phrase is the only way back in.

---

## 4. The encryption scheme

All encrypted records in the local user record share one format and one algorithm, regardless
of which unlock method produced the cipher string. The implementation is published in the
`@meteorwallet/utils` npm package (module `encryption`), built on the audited `@noble/hashes`
and `@noble/ciphers` libraries. App release 0.0.135 ships `@meteorwallet/utils` 1.31.0.

### 4.1 Record format

Every encrypted record is a JSON object:

```json
{
  "authType": "user_password_hash_salt",
  "v": "1",
  "salt": "<base64, 24 random bytes>",
  "encryptedData": "<base64, ciphertext followed by 16-byte GCM tag>"
}
```

- `authType` names the unlock method that produced the cipher string
  (`auto_cipher`, `user_password_hash_salt`, or `device_biometric`).
- `v` is the record format version. Only `"1"` exists.
- `salt` is 24 bytes drawn from the platform's cryptographically secure random source, freshly
  generated for **every** encryption operation. It doubles as the AES-GCM nonce.
- `encryptedData` is the raw AES-GCM output: ciphertext of the same length as the plaintext,
  immediately followed by the 16-byte authentication tag. There is no additional header,
  length prefix, or associated data.

### 4.2 Encrypting

Given a cipher string `C` (section 3) and a serializable value `D`:

1. `saltBytes` = 24 random bytes. `saltB64` = standard base64 encoding of `saltBytes`
   (with `=` padding; a 24-byte salt encodes to 32 characters with no padding).
2. `key` = **PBKDF2-HMAC-SHA256**(password = UTF-8 bytes of `C`, salt = **UTF-8 bytes of the
   base64 string `saltB64`** (not the decoded bytes), iterations = 32, output length = 32 bytes).
3. `plaintext` = UTF-8 bytes of `JSON.stringify(D)`. Note that a plain string value is therefore
   wrapped in double quotes, and an object is serialized with no whitespace.
4. `encryptedData` = base64( **AES-256-GCM**(key, nonce = `saltBytes` (24 bytes), plaintext,
   associated data = none) ) with the 16-byte tag appended to the ciphertext.

The 24-byte nonce is longer than GCM's conventional 12 bytes. Per the GCM specification, a
nonce that is not 96 bits is processed through GHASH to produce the initial counter block. Any
standards-compliant GCM implementation (including WebCrypto and OpenSSL) handles this
transparently when given the 24-byte IV directly.

### 4.3 Decrypting

The reverse: base64-decode `salt` and `encryptedData`, derive `key` exactly as in step 2 using the
base64 salt **string** as the PBKDF2 salt, then AES-256-GCM decrypt with the decoded salt bytes as
the nonce. A GCM tag failure means the cipher string (and therefore the password) is wrong.
On success, `JSON.parse` the UTF-8 plaintext.

### 4.4 What is encrypted

Two kinds of record exist in the local user record, both using the format above:

| Record | Plaintext `D` |
| --- | --- |
| Authentication check token | The fixed string `"auth_check_token"`. The plaintext bytes are the 18-byte JSON string `"auth_check_token"` including its quotes, so `encryptedData` decodes to 34 bytes. |
| Per-account signing payload | A JSON object with `version: "1"` and a `signers` array. Each signer holds the account's public key, the private key pair, and its derivation data (seed and HD path where the key was derived from a recovery phrase). Ledger-backed accounts carry only public data and a derivation path. |

The authentication check token exists so that the app can confirm a password with a single
small decryption before attempting to decrypt account data. Wallets created before the token
was introduced may have it recorded as `authType: "unset"`; those wallets prove the password
by decrypting an account payload instead, and the token is written on the next successful
unlock.

### 4.5 Design notes

- **Why hash the password first?** The SHA-256 step gives every unlock method a uniform,
  fixed-length cipher string, lets biometrics protect the same secret as the password without
  storing the password itself, and means the raw password never reaches the encryption layer.
- **Why is the salt also the nonce?** Both must be unique per encryption and both are generated
  fresh from a secure random source for each operation. Using one 24-byte random value for both
  roles keeps the record format minimal. Because a fresh salt is drawn per encryption, the
  derived key is also unique per record, so nonce reuse across records is not possible.
- **Where the security comes from.** The key-derivation step is deliberately light because the
  scheme's strength against offline guessing comes primarily from the entropy of the password
  the user chooses. The best defense a user has is a long, unique password and a securely stored
  recovery phrase. We may raise the cost of the derivation in a future record format version;
  any such change will carry a new `v` value so that existing records remain readable.

---

## 5. Test vector

This vector was produced with the exact libraries shipped in the app and verified three ways:
with the app's own decrypt routine, with an independent reimplementation over `@noble`, and with
WebCrypto (PBKDF2 + AES-GCM with a 24-byte IV). The password is a well-known dummy value.

```
password (UTF-8)          : correct horse battery staple
cipher string (SHA-256)   : c4bbcb1fbec99d65bf59d85c8cb62ee2db963f0fe106f483d9afa73bd4e39a8a
salt bytes (hex)          : 0102030405060708090a0b0c0d0e0f101112131415161718
salt (base64, as stored)  : AQIDBAUGBwgJCgsMDQ4PEBESExQVFhcY
PBKDF2 salt input         : UTF-8 bytes of "AQIDBAUGBwgJCgsMDQ4PEBESExQVFhcY" (32 bytes)
PBKDF2 params             : HMAC-SHA256, 32 iterations, 32-byte output
derived AES key (hex)     : 63ebc9b19b57a59db8570d8a066f351fda7f51992e8a0b74fcab4bc4527155ba
plaintext (UTF-8)         : "auth_check_token"          (18 bytes, quotes included)
AES-256-GCM nonce         : the 24 salt bytes above
encryptedData (base64)    : E1w1frkEMjZWM5US4bsc/s3FXlQ/+WMXVH297xgeYIe1OA==
                            (34 bytes: 18 ciphertext + 16 tag)
```

A tool that reproduces `encryptedData` from the inputs above, or decrypts it back to the
plaintext, implements the scheme correctly. A real wallet's records use a random salt, so
`salt` and `encryptedData` will differ, but the procedure is identical.

---

## 6. Recovery: what is and is not possible

**Meteor cannot recover a forgotten password or a lost recovery phrase.** There is no server-side
copy of either, no escrow, and no backdoor. The two supported ways to regain access to an account
are:

1. Enter the correct password on the device that holds the wallet.
2. Restore the account on any device from its recovery phrase (or private key).

If both the password and the recovery phrase are lost, the only remaining option is for the
user to guess their own password offline against their own data. This is a legitimate use of
the information in this document, and the following notes are intended to make it as
efficient and as safe as possible.

### 6.1 Offline password check procedure

Given the `salt` and `encryptedData` of a record and a candidate password:

```
cipher   = hex( SHA256( utf8(candidate) ) )
key      = PBKDF2_HMAC_SHA256( password = utf8(cipher), salt = utf8(saltBase64), c = 32, dkLen = 32 )
plain    = AES_256_GCM_DECRYPT( key, nonce = base64decode(saltBase64), data = base64decode(encryptedData) )
```

- If the GCM tag check fails, the candidate is wrong.
- If it succeeds on the authentication check token, `plain` is exactly the 18 bytes
  `"auth_check_token"` and the candidate is the password.
- Prefer testing against the authentication check token: it is the smallest record and its
  expected plaintext is known. If that token is recorded as `"unset"`, test against one
  account's signing payload instead; success yields a JSON object beginning with `{"version":"1"`.
- Once the password is known, decrypt each account's signing payload the same way. The
  resulting JSON contains the private key material and, where applicable, the seed, which can
  be imported into Meteor or any compatible wallet.

### 6.2 Practical guidance for candidate passwords

- Test the **exact bytes** typed. Leading or trailing spaces, capitalization, and Unicode
  variants of the same visible character all produce different keys.
- Mobile keyboards commonly auto-capitalize the first letter, insert a space after
  punctuation, or substitute typographic quotes and dashes. Include those variants.
- Minimum length is 8 characters. Anything shorter was never accepted by the app.
- Passwords are compared by decrypting, not by lookup, so there is no stored hash to compare
  against and no lockout counter to worry about in an offline check.

### 6.3 Handling recovered material

Anyone performing a recovery should treat the decrypted output as equivalent to the recovery
phrase itself: do it on a trusted, offline machine, never paste keys or phrases into websites or
messages, and move funds to a fresh wallet with a new, backed-up recovery phrase as soon as
access is regained. Meteor support will never ask for a password, private key, recovery
phrase, or encrypted payload.

---

## 7. What this document intentionally leaves out

To avoid handing an attacker a map of the app without weakening any user's security, we do not
publish: the names of storage keys and keychain service identifiers, internal module and file
layout, the exact list of non-sensitive fields in the local record, and operational details of
biometric prompts. None of these are security controls; the security controls are the ones
described above, and they are designed to hold up with full knowledge of the design.

Security researchers who believe they have found a weakness in any of the above are asked to
contact Meteor through our official support channels rather than disclosing publicly, so that
users can be protected first.
