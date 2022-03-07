---
layout: post
title:  "Poly1305-aes in openssl 3.0"
date:   2022-03-07 23:26:00
categories: technology,cryptography, c, yocto,
---

Poly1305 is an Authentication scheme, similar to those of HMAC and CMAC. However, this scheme differs to that of both in the following respects.

1. requires the change of key for every new message to be authenticated
2. requires a 32 bit random unpredictable key (prefer generation via chacha20 or other forms of key generation schemes)
3. AES is used just like in the CBC mode for CMAC. (HMAC however uses hash algorithms)

OpenSSL 3.0 has the support for doing poly1305-aes.

The new API interfaces are bit daunting for the first time, but here they are:

| S.NO | API Name | Description |
|------|----------|-------------|
| 1 | EVP_MAC_fetch | fetch a MAC generation algorithm |
| 2 | EVP_MAC_CTX_new | create a new EVP_MAC context |
| 3 | EVP_MAC_init | initialize the new MAC with the key and params (such as cipher, hash etc) |
| 4 | EVP_MAC_update | streaming mode |
| 5 | EVP_MAC_final | finalize the MAC and copy it to the target buffer |
| 6 | EVP_MAC_CTX_free | free the context |
| 7 | EVP_MAC_free | free the returned MAC context |

Below are the steps:

1. Get Mac Context -> `mac = EVP_MAC_fetch(NULL, "poly1305", NULL);`
2. Create new MAC Context -> `EVP_MAC_CTX_new(mac);`
3. Initialize MAC -> `EVP_MAC_init(ctx, key, sizeof(key), params)`
   1. setting parameters:
       for poly1305 nothing needs to be set except the end pointer
       ```c
           OSSL_PARAM params[1];
           size_t params_n = 0;

           params[params_n] = OSSL_PARAM_construct_end();
       ```
4. Update / Stream buffers to the MAC context -> `EVP_MAC_update(ctx, buf, sizeof(buf));`
5. Once streaming complete, get the result -> `EVP_MAC_final(ctx, mac, &mac_len, sizeof(mac));`

The example program is shown below.

To validate the results, i took the test vector from [here](https://github.com/madmax440/cryptol/blob/master/examples/MiniLock/prim/Poly1305.md)

```bash
test vector
===========
Poly1305TestKey = join (parseHexString
    ( "85:d6:be:78:57:55:6d:33:7f:44:52:fe:42:d5:06:a8:01:"
    # "03:80:8a:fb:0d:b2:fd:4a:bf:f6:af:41:49:f5:1b."
    ) )
    
Poly1305TestMessage = "Cryptographic Forum Research Group"

Poly1305TestTag = "a8:06:1d:c1:30:51:36:c6:c2:2b:8b:af:0c:01:27:a9."
```

The program below yields:

```bash
Result: A8:06:1D:C1:30:51:36:C6:C2:2B:8B:AF:0C:01:27:A9:
```

<script src="https://gist.github.com/madmax440/0d3fc84c63cd37b2f52cdb4cf90e3818.js"></script>
