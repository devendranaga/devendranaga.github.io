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

### Key Hierarchy

KEK is the key encryption key.
KSIK is Key store integrity key.

The salt is varied so the output keys are unique.

1. password is given via the command line argument.
2. Derive RKEK: `KEK = PBKDF(password, salt, key_len, hash);`.
3. Derive KSIK: `KSIK = PBKDF(password, salt, key_len, hash);`.

### Key storage format

```
|---------------------------|
|      Key store metadata   |
|---------------------------|
|      Wrapped Key data 1   |
|---------------------------|
|      Wrapped Key data 2   |
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

**Key store metadata**:

```c
typedef struct {
    uint32_t magic; // 'K' 'E' 'Y' 'S' in Hex
    uint32_t version;
    uint32_t creation_time_sec;
    uint32_t creation_time_usec;
} ss_key_metadata_t;
```

**Wrapped Symmetric key structure**:

```c
#define SS_KEY_USE_AES_GCM  0x00000001
#define SS_KEY_USE_AES_CMAC 0x00000002
#define SS_KEY_USE_AES_WRAP 0x00000004

typedef struct {
    uint32_t  key_type; // type of symmetric key
    uint32_t  usage; // usage of this key
    uint8_t   key[40]; // wrapped key data
    uint32_t  key_len; // 16, 24 or 32
} ss_wrapped_symm_key_t;

typedef struct {
    uint32_t key_len;
    uint32_t key[1];
} ss_wrapped_asymm_key_t;

#define SS_KEY_TYPE_RSA 1
#define SS_KEY_TYPE_EC  2

typedef struct {
    uint32_t key_type; // type of asymmetric key
    ss_wrapped_asymm_key_t pub_key;
    ss_wrapped_asymm_key_t priv_key;
} ss_wrapped_asymm_key_t;
```

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
