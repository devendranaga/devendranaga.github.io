---
layout: post
title:  "Secure Storage on Linux"
date:   2025-08-25 07:04:00
categories: cryptography
---

# Version
1. Initial version - half done with the usecase, key storage, key format, interface definitions.

## Goals

1. The system must be able to generate and load keys from an external entity securely.
2. The system must let the users to perform cryptographic operations.
3. An external entity must never be able to read keys.

## Assumptions

1. The communication link between the user and the secure storage service is protected.

## Usecases

1. Key generation for RSA / EC with the secure storage.
2. Use of RSA / EC keys for Key exchange / agreement schemes.
3. Return public key back.
4. Unwrap the secret key with the stored private key and write into the key storage.
5. Generate symmetric keys via a secure random number generator.
6. Perform cryptographic operations such as encrypt / decrypt / authenticate and verify.

## Design

### Startup

The startup would involve the secure storage service initializing the command line and enter into console mode.
The user will add in the password via the console and the daemon starts up in the background.

The daemon startup would involve,

1. Generating the `KEK` and `KSIK` based on the user provided password as explained in the below section.
2. Integrity check on the key storage using the KSIK. Both Main and Backup.
3. If either storage integrity is failed, the secure storage service will attempt to restore the corrupted storage from the backup.
4. If the backup is also corrupted, the service will log error and halt.

### Key Hierarchy

KEK is the key encryption key. Wraps the key store wrapping key.
KSIK is Key store integrity key. Performs integrity check and generates integrity over the key store.

The salt is varied so the output keys are unique.

1. Derive RKEK: `KEK = PBKDF(password, salt, key_len, hash);`.
2. Derive KSIK: `KSIK = PBKDF(password, salt, key_len, hash);`.
3. Generate a unique random master key. Wrap this key with the KEK.
4. Store the unique random master key in two key slots within the key storage.
5. The unique random master key will then be used as a wrapping key for other keys within the secure storage.

### Key storage format

```
|---------------------------|
|      Key store metadata   |
|---------------------------|
|      Wrapped Key data 1   | <-- master wrapping key
|---------------------------|
|      Wrapped Key data 2   | <-- master wrapping key replica
|---------------------------|
|             .             |
|             .             |
|---------------------------|
|      Wrapped Key data N   |
|---------------------------|
|      Storage MAC          |
|---------------------------|
```

Two types of key storage are required. Main and Backup.

Backup storage replicates the Main storage. Main storage contains a list of wrapped keys and key metadata.
Storage MAC would protect the integrity of the entire keys. If any wrapped key has been altered indirectly, the secure storage can flag this and restore it from the backup. If the backup integrity is not verified, the Secure Storage flags this event over syslog.

Storage MAC is calculated over the key store metadata and the entire wrapped key data to ensure the integrity of the key store.

Keys and wrapped and unwrapped using `AES_KeyWrapPadding` and `AES_KeyUnwrapPadding` algorithm. The IV is kept as a constant as per the standard with 0xA6 repeated 8 times padded with 4 bytes of 0s.

Every key wrap operation involve zeroing out the key buffer after the operation is complete.

**Key store metadata**:

A sample of how the key metadata looks like as follows.

```c
typedef struct {
    uint32_t magic; // 'K' 'E' 'Y' 'S' in Hex
    uint32_t version;
    uint32_t creation_time_sec;
    uint32_t creation_time_usec;
    uint8_t  salt[32];
    uint32_t kdf_iterations;
    uint32_t n_wrapped_keys;
} ss_key_metadata_t;
```

**Wrapped Symmetric key structure**:

Wrapped key data is either a `ss_wrapped_symm_key_t` or `ss_wrapped_asymm_key_t`.
The `key_type` will help knowing the type of the wrapped key. Based on that the rest of the data structure can be known during the load time.

```c
#define SS_KEY_USE_AES_GCM               0x00000001
#define SS_KEY_USE_AES_CMAC              0x00000002
#define SS_KEY_USE_AES_WRAP              0x00000004
#define SS_KEY_USE_RSA_PUB_ENCRYPT       0x00000008
#define SS_KEY_USE_RSA_PRIV_DECRYPT      0x00000010
#define SS_KEY_USE_RSA_PRIV_SIGN         0x00000020
#define SS_KEY_USE_RSA_PUB_VERIFY        0x00000040

typedef struct {
    uint32_t  key_type; // type of symmetric key
    uint8_t   key_id[32]; // unique id for this wrapped key
    uint32_t  usage; // usage of this key
    uint32_t  key_len; // 16, 24 or 32
    uint8_t   key[1]; // wrapped key data
} ss_wrapped_symm_key_t;

typedef struct {
    uint32_t key_len;
    uint32_t key[1];
} ss_wrapped_asymm_key_t;

#define SS_KEY_TYPE_RSA 1
#define SS_KEY_TYPE_EC  2

typedef struct {
    uint32_t key_type; // type of asymmetric key
    uint8_t  key_id[32]; // unique id for this wrapped key
    ss_wrapped_asymm_key_t pub_key;
    ss_wrapped_asymm_key_t priv_key;
} ss_wrapped_asymm_key_t;
```

The `key_id` is uniquely generated for each store request. It is generated via the random number generator such as `/dev/unrandom`.

### Interface format

```c
typedef enum {
    SS_KEY_OP_STORE_KEY_RSA_DECRYPT,
    SS_KEY_OP_GEN_RSA_KEY,
    SS_KEY_OP_GEN_SYMM_KEY,
    SS_KEY_OP_LOAD_KEY,
} ss_key_op_t;

typedef struct {
} ss_key_intf_t;
```
