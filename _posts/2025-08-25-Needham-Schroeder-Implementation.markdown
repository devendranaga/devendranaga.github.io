---
layout: post
title:  "Learning / PoC implementation of Needham-Schroeder Protocol"
date:   2025-08-25 07:04:00
categories: cryptography
---

Recently i have come cross the Needham-Schroeder protocol. So thought of implementing it in C++ following the public key version of the protocol over this weekend.

The protocol details can be found from wikipedia [here](https://en.wikipedia.org/wiki/Needham%E2%80%93Schroeder_protocol).

As described above, the protocol is intended for communication over an insecure network. Few of the current systems use it as their backbone for communication.

I  have put a diagram of the public key based implementation of the protocol below. It took me 2 hours to implement and validate the version of the protocol.

TL;DR, this is not a nice implementation of Needham-Schoreder protocol. But it is only used for learning and understanding the protocol itself. Look for the github link on the implementation code.

A full fledged implementation of the protocol can be done with a proper statemachine with the assumptions of few preconditions. This is for future works.


Here's a brief detail about the protocol:

Problem:
========
1. A and B wants to communicate with each other over an insecure channel.

Preconditions:
==============
1. A and B knows each of their identities beforehand (somehow, either pre-provisioned via application etc). In case of embedded systems this can simply be a unique idenitier that is programmed during manufacturing. In other cases, A might know B via some means (DNS query for example).
2. A and B both knows S, server public keys and they have shared with the server their public keys as  well.

Protocol:
=========
1. Assume KPA - A's public key, KSA - A's secret key, KPB - B's public key, KSB - B's secret key, KPS - S's public key, KSS - S's secret key
1. A requests S about B public key (A -> S: {A, B}).
2. S responds with a message by appending B's public key and B's identity, signed by S's private key KSS. (S -> A: {KPB, B}KSS)
3. A then verifies S's message with S's public key. A then creates an encrypted request to B with A's Nonce NA and its identity A with B's public key KPB. (A -> B: {NA, A}KPB)
4. B upon receiving, understands A wants to communicate further, creates a request to S. (B -> S: {B, A})
5. S upon receiving B's request, prepares a message containing A's public key KPA, identity A, signed by its private key KSS. (S-> B: {KPA, A}KSS)
6. B verifies S's message with S's public key. B then decrypts A's message with its private KSB. There by parsing KPA and A. B then sends an encrypted message to A with its nonce NB appending to A's nonce NA, Encryption uses A's public key KPA. (B -> A: {NA, NB}KPA)
7. A decrypts B's message with its private KSA. Verifies its nonce NA. This makes A trust B that B can understand its nonce NA.
8. A formulates message with B's nonce NB, encrypting with B's public key KPB. (A -> B: {NB}KPB)
9. B decrypts the message with its private key, KSB. Verifies its nonce NB. This makes B trust A that A can understand its nonce NB.

Since A and B now know each of their nonces, they can create session keys with KDF functions.

For example a KDF  can be realised as following:

1. input : ID of A << n OR ID of B
2. salt  : SHA_256(NA << n OR NB)
3. info  : CA's email address or something commonly known for both the participants or empty

```bash
output_keys (1..255keys) = HKDF(input, salt, info);
```

Below is the diagram that shows the sequence of steps involved.



![NS_Protocol](https://raw.githubusercontent.com/devendranaga/devendranaga.github.io/blob/master/_posts/NS_protocol.jpg)

The C++ implementation of the protocol below simulates A, B and S as functions within a single program. In realitiy they are tasks or services in each machine, device running and communicating over the insecured channel.

For Public key implementations, we use RSA for signing and encryption. We sign with Private key and verify with Public keys. We encrypt with Public key and decrypt with Private Key. We use RSA key length of 2048 for learning purposes. For realworld uses 3072 key lengths are better.

HKDF derivation of the final keys is not shown in the implementation.



An example output of the implementation is as follow:

```bash
main:	 setup communication

step1:
======
alice:	 initiate communication
alice:	 send uid 0x12345678 bob uid 0x12345688 to server A->S: A, B


step2:
======
server:	 prepare response
server:	 read bob's public key
server:	 copy bob's key
server:	 send response to A, S -> A {KPB, B}KSS


step3:
=====
alice:	 send first message to bob
alice:	 server signature verified.. extract bob's key
alice:	 set nonce and alice uid
alice:	 encrypted ok 256 A -> B: {NA, A}KPB


step4:
=====
bob:	 received alice request
bob:	 set bob_uid 0x12345688 alice_uid 0x12345678 B->S: {B, A}
bob:	 request server about alice identity


step5:
=====
server:	 load alice public key
server:	 copy alice public key
server:	 copy alice uid
server:	 signed ok S -> B: {KPA, A}KSS
server:	 respond to bob with KPA, A signed with server priv key


step6:
=====
bob:	 verified server message ok
bob:	 try to verify alice len 20
bob:	 encrypt {NA,NB}KPA
bob:	 sends {NA, NB}KPA to alice


step7:
=====
alice:	 decrypt bob's msg ok
alice:	 **** bob is verified ok **** 
alice:	 b_nonce verified ok
alice:	 encrypt ok A -> B {NB}KPB
alice:	 sends {NB}KPB to bob


step8:
=====
bob:	 decrypt alice msg ok
bob:	 **** alice is verified ok ****
main:	 mutual verification complete

```

A C++ implementation of the protocol is below.

<script src="https://gist.github.com/madmax440/61964cf4fc9a2c676b3430b1ba43a0f3.js"></script>


<script src="https://gist.github.com/madmax440/0b3a81c82d2c2bb37be70ab42c09f62d.js"></script>


<script src="https://gist.github.com/madmax440/c063a0f1165fb85fa17aa43bf64f2912.js"></script>


