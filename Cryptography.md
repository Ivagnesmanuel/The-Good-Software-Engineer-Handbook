
# Cryptography

A quick-reference guide covering the cryptographic primitives that secure modern systems — symmetric and asymmetric ciphers, hash functions, MACs, signatures, key exchange, KDFs, and TLS as the canonical case study of how they compose.

---

## Table of Contents

1. Cryptography — Overview
2. Classical Ciphers (Intuition Only)
3. Symmetric Cryptography
4. Modes of Operation and AEAD
5. Asymmetric Cryptography
6. Hash Functions, MACs, and Passwords
7. Digital Signatures and PKI
8. Key Exchange and KDFs
9. TLS as a Case Study

---

## 1. Cryptography — Overview

Cryptography provides mathematical tools that allow systems to communicate securely even in the presence of attackers. It is the foundation of every secure protocol, but it is only one piece of the puzzle — a perfectly strong cipher is useless if keys leak, authentication is broken, or the software has bugs.

### The Four Security Goals

Every security mechanism targets one or more of these properties:

|Property|Meaning|Achieved by|
|---|---|---|
|Confidentiality|Only intended parties can read the data|Encryption (AES, ChaCha20)|
|Integrity|Data cannot be modified undetected|MACs, hashes, digital signatures|
|Authentication|You know who you are communicating with|Certificates, tokens, passwords|
|Non-repudiation|A sender cannot deny sending a message|Digital signatures (RSA, ECDSA)|

These four properties combine to form a **secure channel**:

```
Client                             Server

Plaintext
   |
Encrypt (key) + MAC (key)
   |
Ciphertext + Tag  ------------ ->  Ciphertext + Tag
	                                    |
	                               Verify MAC (key)
	                                    |
	                               Decrypt (key)
	                                    |
	                               Plaintext
```

### Why Cryptography Alone Is Not Enough

A system can use AES-256 and still be trivially broken if:

- Keys are stored in plaintext config files or committed to version control.
- Authentication is missing — anyone can connect and negotiate a session.
- The protocol allows downgrade to weaker ciphers (downgrade attacks).
- The implementation has side-channel vulnerabilities (timing attacks, padding oracles).
- Random number generation is weak or predictable.

Modern systems combine multiple layers:

```
Layer               What it provides                     Example
-----               --------------------                 -------
Encryption          Confidentiality                      AES-GCM, ChaCha20-Poly1305
Authentication      Identity verification                Certificates, mTLS, OAuth2
Key exchange        Shared secret without prior contact  ECDHE (Diffie-Hellman)
Certificates        Trust bootstrapping                  X.509, PKI, Let's Encrypt
Identity systems    Who can do what                      IAM, RBAC, ABAC
Network defenses    Perimeter and runtime protection     Firewalls, IDS/IPS, WAFs
```

### Kerckhoffs's Principle

A cryptosystem should be secure even if everything about the system, except the key, is public knowledge. In practice this means:
- Never rely on secret algorithms ("security through obscurity").
- Assume attackers know the algorithm, the protocol, the source code.
- The **key** is the only secret.

This is why AES, SHA-256, and TLS specifications are fully public. Any algorithm that requires secrecy of its design is considered broken by modern standards.

---

## 2. Classical Ciphers (Intuition Only)

Before modern cryptography, encryption relied on simple substitution and transposition techniques. These systems are **not secure today**, but they illustrate ideas that remain fundamental: key space, frequency analysis, confusion, and diffusion.

### Caesar Cipher

Each letter is shifted by a fixed number (the key).

```
Plaintext alphabet:   A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Shift +3:             D E F G H I J K L M N O P Q R S T U V W X Y Z A B C

Plaintext:   H E L L O
Ciphertext:  K H O O R
```

Decryption shifts back by the same amount.

**Why it fails:**
- Key space = 25 (only 25 possible shifts).
- Brute force takes seconds by hand, microseconds by machine.

### Substitution Cipher

Each letter maps to a different letter via a random permutation.

```
Plaintext alphabet:   A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Cipher alphabet:      Q W E R T Y U I O P A S D F G H J K L Z X C V B N M
```

Key space = 26! (approximately 4 x 10^26) — far too large for brute force.

**Why it fails anyway: frequency analysis.**

English text has predictable letter frequencies:

```
Letter  Frequency      Letter  Frequency
  E       12.7%          T       9.1%
  A        8.2%          O       7.5%
  I        7.0%          N       6.7%
  S        6.3%          H       6.1%
```

An attacker counts letter frequencies in the ciphertext and maps the most common ciphertext letter to E, the next to T, and so on. With a few hundred characters, the cipher breaks easily.

**Bigram and trigram analysis** accelerates the attack further — common pairs like TH, HE, IN, ER, and common triples like THE, AND, ING are identifiable patterns.

### Vigenere Cipher

Uses a repeating keyword to apply multiple different shifts, defeating simple single-letter frequency analysis.

```
Plaintext:  A T T A C K A T D A W N
Key:        L E M O N L E M O N L E
            -----------------------
Ciphertext: L X F O P V E F R N H R
```

Each column uses a different Caesar shift (determined by the key letter). This flattens single-letter frequencies.

**Why it fails: Kasiski examination and index of coincidence.**

If the key length is k, then every k-th letter is encrypted with the same shift. An attacker:
1. Determines the key length by finding repeating ciphertext patterns (Kasiski) or by measuring the index of coincidence for different assumed key lengths.
2. Splits the ciphertext into k groups, each encrypted with a single Caesar shift.
3. Applies frequency analysis to each group independently.

With enough ciphertext, this breaks completely.

### Lessons From Classical Ciphers

These failures motivate the requirements for modern cryptography:

|Requirement|Classical failure|Modern solution|
|---|---|---|
|Large key space|Caesar: only 25 keys|AES: 2^128 or 2^256 keys|
|Resistance to frequency analysis|Substitution: letter frequencies leak|Confusion and diffusion (Shannon)|
|No pattern preservation|ECB-like repetition|Chaining modes (CBC, CTR)|
|Strong randomness|Predictable keys|CSPRNGs (cryptographically secure PRNGs)|

**Shannon's principles (1949):**
- **Confusion:** each bit of the ciphertext should depend on multiple parts of the key. Makes the relationship between key and ciphertext complex. Achieved by substitution (S-boxes in AES).
- **Diffusion:** each bit of the plaintext should affect many bits of the ciphertext. Small changes in plaintext produce large changes in ciphertext (avalanche effect). Achieved by permutation and mixing (ShiftRows, MixColumns in AES).

A good cipher exhibits both. Classical ciphers have substitution (confusion) but poor or no diffusion.

---

## 3. Symmetric Cryptography

Symmetric cryptography uses the **same key for encryption and decryption**. Both parties must possess the key before communication begins.

```
Plaintext + Key  ---> Encrypt ---> Ciphertext

Ciphertext + Key ---> Decrypt ---> Plaintext
```

Symmetric encryption is used for **bulk data encryption** because it is extremely fast — AES with hardware acceleration (AES-NI) processes data at tens of gigabytes per second on modern CPUs.

The central problem is **key distribution**: how do two parties agree on a shared key without an attacker intercepting it? This is solved by asymmetric cryptography or pre-shared keys (covered in sections 5 and 8).

### Block Ciphers vs Stream Ciphers

|Property|Block Cipher|Stream Cipher|
|---|---|---|
|Unit of encryption|Fixed-size block (e.g., 128 bits)|One byte/bit at a time|
|State|Stateless per block (mode adds state)|Maintains keystream state|
|Parallelization|Depends on mode (CTR: yes, CBC: no)|Inherently sequential keystream, but XOR is parallelizable|
|Examples|AES, 3DES (deprecated)|ChaCha20, RC4 (broken)|
|Primary use|Disk encryption, TLS, file encryption|TLS (ChaCha20), QUIC, mobile|

### Block Ciphers

Block ciphers encrypt fixed-size blocks of plaintext into equal-size blocks of ciphertext. The same key and plaintext always produce the same ciphertext (for a given block).

```
Plaintext block (128 bits)
        |
        v
   Block Cipher (key)
        |
        v
Ciphertext block (128 bits)
```

#### AES (Advanced Encryption Standard)

AES is the dominant symmetric cipher. Selected by NIST in 2001 after a public competition (Rijndael algorithm by Daemen and Rijmen).

**Core parameters:**

|Key size|Rounds|Security level|
|---|---|---|
|128 bits|10|Standard|
|192 bits|12|Higher margin|
|256 bits|14|Required for top-secret (NSA Suite B)|

Block size is always **128 bits** (16 bytes) regardless of key size.

**How AES works (one round):**

Each round applies four operations to a 4x4 byte matrix (the "state"):

```
State (4x4 bytes = 128 bits)

+----+----+----+----+
| b0 | b4 | b8 | b12|
| b1 | b5 | b9 | b13|        Byte layout of
| b2 | b6 | b10| b14|        the 128-bit block
| b3 | b7 | b11| b15|
+----+----+----+----+

Round operations:

1. SubBytes      - each byte replaced via S-box lookup (confusion)
2. ShiftRows     - rows shifted left by 0,1,2,3 positions (diffusion)
3. MixColumns    - column-wise matrix multiplication in GF(2^8) (diffusion)
4. AddRoundKey   - XOR with round key derived from key schedule
```

Step by step:

```
Plaintext
    |
AddRoundKey (initial)
    |
+---v----+
| Round  |  x 9/11/13 (for AES-128/192/256)
|  Sub   |
|  Shift |
|  Mix   |
|  AddRK |
+---v----+
    |
Final Round (no MixColumns)
    |
Ciphertext
```

**Why these steps matter:**
- **SubBytes (S-box):** A non-linear substitution. Each byte is independently replaced by looking up a table. This is the primary source of confusion — it destroys any linear relationship between input and output.
- **ShiftRows:** Cyclically shifts each row of the state by a different offset. Ensures that bytes from one column spread into all four columns, beginning the diffusion process.
- **MixColumns:** Treats each column as a polynomial over GF(2^8) and multiplies by a fixed matrix. This is the main diffusion step — each output byte depends on all four input bytes in the column.
- **AddRoundKey:** XORs the state with a round-specific key derived from the master key via the key schedule. Without this, the cipher would be a fixed (and reversible) permutation with no key dependence.

After approximately 4 rounds, a single-bit change in the plaintext affects every bit of the ciphertext (**full diffusion / avalanche effect**).

**AES-NI (hardware acceleration):**

Modern x86 CPUs include dedicated AES instructions:

```
AESENC     — one AES encryption round
AESENCLAST — final AES encryption round
AESDEC     — one AES decryption round
AESKEYGEN  — key expansion
```

With AES-NI, AES-GCM can process 5-10 GB/s on a single core. This makes hardware-accelerated AES faster than most software stream ciphers on x86. ARM has equivalent instructions (ARM CE — Cryptographic Extensions).

> **When AES-NI is unavailable** (older devices, some embedded systems, some ARM chips without CE), ChaCha20 is faster in pure software. This is why TLS servers often negotiate ChaCha20-Poly1305 with mobile clients.

#### Deprecated Block Ciphers

|Cipher|Block Size|Key Size|Status|Why deprecated|
|---|---|---|---|---|
|DES|64 bits|56 bits|Broken|Key too short — brute-forced in 1998|
|3DES|64 bits|168 bits (effective ~112)|Deprecated|64-bit block enables Sweet32 attack (birthday collision after 2^32 blocks)|
|Blowfish|64 bits|32-448 bits|Legacy|64-bit block, same birthday problem|

**Rule:** if the block size is 64 bits, do not use it. After encrypting 2^32 blocks (~32 GB), birthday collisions become likely, leaking information.

### Stream Ciphers

Stream ciphers generate a **keystream** from a key and a nonce (number used once), then XOR it with the plaintext.

```
Key + Nonce --> Keystream Generator --> Keystream

Plaintext:   10110001 01101010 11001100 ...
Keystream:   01101100 10010111 01010011 ...
             -------- -------- --------
Ciphertext:  11011101 11111101 10011111 ...
```

Decryption is identical: XOR the ciphertext with the same keystream.

#### ChaCha20

Designed by Daniel Bernstein. The modern standard stream cipher.

**Core design:**

- 256-bit key, 96-bit nonce, 32-bit counter.
- Operates on a 4x4 matrix of 32-bit words (512-bit state).
- 20 rounds of the "quarter-round" operation (add, XOR, rotate — all constant-time).
- Produces 512 bits (64 bytes) of keystream per block.

```
State matrix (initial):

+----------+----------+----------+----------+
| constant | constant | constant | constant |   "expand 32-byte k"
| key[0]   | key[1]   | key[2]   | key[3]   |
| key[4]   | key[5]   | key[6]   | key[7]   |
| counter  | nonce[0] | nonce[1] | nonce[2] |
+----------+----------+----------+----------+

After 20 rounds of quarter-round mixing:

State is added to the initial state (mod 2^32)
Result = 64 bytes of keystream
Increment counter, repeat for next block
```

**Why ChaCha20 matters:**

- **No S-boxes or table lookups** — immune to cache-timing side-channel attacks that can leak AES keys when AES-NI is absent.
- **Fast in software** — uses only add/XOR/rotate (ARX design), which are constant-time on all CPUs.
- **Used by:** TLS 1.3 (as ChaCha20-Poly1305), WireGuard, QUIC, Google's HTTPS traffic to Android devices, SSH.

#### Broken Stream Ciphers

- **RC4:** Severe biases in keystream output. Banned from TLS since RFC 7465 (2015). Never use RC4.

### Nonce Reuse: The Cardinal Sin of Symmetric Crypto

Encrypting two different plaintexts with the same key and nonce (stream cipher) or same key and IV (CTR mode) has the following consequence:

```
C1 = P1 XOR Keystream
C2 = P2 XOR Keystream

C1 XOR C2 = P1 XOR P2
```

The keystream cancels out. An attacker now has the XOR of the two plaintexts, which is often enough to recover both messages using crib-dragging (guessing one plaintext and XORing to reveal the other).

**Prevention:**
- Use a random nonce for every message (96-bit nonce gives negligible collision probability for up to ~2^32 messages).
- Or use a counter that never repeats.
- Or use a nonce-misuse-resistant scheme (AES-GCM-SIV) that degrades gracefully — repeated nonces only leak whether two plaintexts are identical, without revealing content.

---

## 4. Modes of Operation and AEAD

Block ciphers encrypt exactly one block at a time (128 bits for AES). To encrypt longer messages, a **mode of operation** defines how blocks are processed.

### ECB (Electronic Codebook) — Never Use This

Each block is encrypted independently.

```
Block1 --> Encrypt(Key) --> C1
Block2 --> Encrypt(Key) --> C2
Block3 --> Encrypt(Key) --> C3
```

**The problem:** identical plaintext blocks produce identical ciphertext blocks. Structure in the plaintext is preserved in the ciphertext. The classic demonstration is encrypting a bitmap image — the outline of the image is clearly visible in the ciphertext.

```
Plaintext blocks:    |  A  |  B  |  A  |  A  |  C  |  A  |
ECB ciphertext:      | x7f | 3b2 | x7f | x7f | 9a1 | x7f |
                       ^^^         ^^^   ^^^         ^^^
                       identical blocks remain identical
```

**Never use ECB.** It is only acceptable for encrypting a single block (e.g., encrypting a single AES key with another key).

### CBC (Cipher Block Chaining)

Each block is XORed with the previous ciphertext block before encryption.

```
C0 = IV (random, sent in cleartext)

C1 = Encrypt(P1 XOR IV,  Key)
C2 = Encrypt(P2 XOR C1,  Key)
C3 = Encrypt(P3 XOR C2,  Key)

Decryption:
P1 = Decrypt(C1, Key) XOR IV
P2 = Decrypt(C2, Key) XOR C1
P3 = Decrypt(C3, Key) XOR C2
```

```
Encrypt:
          IV
          |
P1 --XOR--+---> Encrypt --> C1
                             |
P2 --XOR---------------------+---> Encrypt --> C2
                                               |
P3 --XOR---------------------------------------+---> Encrypt --> C3

Decrypt:
C1 ---> Decrypt --XOR-- IV --> P1
C2 ---> Decrypt --XOR-- C1 --> P2
C3 ---> Decrypt --XOR-- C2 --> P3
```

**Properties:**
- Identical plaintext blocks produce different ciphertext (good).
- Encryption is sequential (each block depends on the previous). Cannot be parallelized.
- Decryption can be parallelized (each block only depends on the corresponding ciphertext and the previous ciphertext).
- Requires padding (PKCS#7) to fill the last block — this has been a source of **padding oracle attacks** (see `Security Fundamentals.md`).

**IV must be unpredictable.** A predictable IV enables chosen-plaintext attacks (BEAST attack on TLS 1.0's CBC).

### CTR (Counter Mode)

Transforms a block cipher into a stream cipher. A counter (nonce + incrementing value) is encrypted to produce a keystream, which is XORed with plaintext.

```
Nonce || Counter=1 --> Encrypt(Key) --> Keystream1
Nonce || Counter=2 --> Encrypt(Key) --> Keystream2
Nonce || Counter=3 --> Encrypt(Key) --> Keystream3

C1 = P1 XOR Keystream1
C2 = P2 XOR Keystream2
C3 = P3 XOR Keystream3
```

```
  Nonce|1       Nonce|2       Nonce|3
     |             |             |
  Encrypt       Encrypt       Encrypt       (all independent -- parallelizable)
     |             |             |
P1--XOR-->C1   P2--XOR-->C2   P3--XOR-->C3
```

**Properties:**
- **Fully parallelizable** (encryption and decryption).
- No padding needed (just truncate the keystream to match plaintext length).
- Random access — can decrypt any block independently.
- **Nonce must never repeat** with the same key (same issue as stream ciphers).

### AEAD (Authenticated Encryption with Associated Data)

Encryption without integrity is dangerous. An attacker can flip bits in the ciphertext and cause predictable changes in the plaintext (bit-flipping attack on CTR/CBC without MAC). Early protocols (WEP, SSL 3.0) failed because they encrypted without authenticating.

AEAD modes provide both **confidentiality and integrity** in a single operation. They also authenticate **associated data** (AAD) — data that is not encrypted but must be tamper-proof (e.g., packet headers, metadata).

```
Inputs:
  - Key
  - Nonce (unique per message)
  - Plaintext (to be encrypted)
  - Associated Data (authenticated but not encrypted)

Outputs:
  - Ciphertext (same length as plaintext)
  - Authentication Tag (typically 128 bits)

Encrypt:  (Key, Nonce, Plaintext, AAD) --> (Ciphertext, Tag)
Decrypt:  (Key, Nonce, Ciphertext, Tag, AAD) --> Plaintext or REJECT
```

If even a single bit of the ciphertext, the tag, or the AAD has been modified, decryption returns an error. There is no partial decryption — it either succeeds or fails entirely.

#### AES-GCM (Galois/Counter Mode)

The most widely used AEAD mode. Combines CTR mode encryption with GHASH (a polynomial hash over GF(2^128)) for authentication.

```
AES-GCM internals:

Nonce|Counter --> AES --> Keystream XOR Plaintext --> Ciphertext
                                                         |
                                              GHASH(Ciphertext + AAD)
                                                         |
                                              XOR with AES(Nonce|0)
                                                         |
                                                  Authentication Tag
```

**Parameters:**
- Key: 128 or 256 bits.
- Nonce: 96 bits (12 bytes) is standard. Longer nonces are hashed down.
- Tag: 128 bits (16 bytes) is standard. Truncation possible but weakens security.

**Performance:** With AES-NI + PCLMULQDQ (carry-less multiply instruction for GHASH), AES-256-GCM processes data at hardware speed — typically 5-10 GB/s per core.

**Critical constraint: nonce reuse is catastrophic.** Reusing a nonce with the same key reveals the GHASH authentication key, allowing an attacker to forge tags and modify messages without detection. Additionally, CTR keystream reuse leaks plaintext XOR (same as any stream cipher nonce reuse).

#### ChaCha20-Poly1305

AEAD construction pairing ChaCha20 (encryption) with Poly1305 (MAC). Designed by Daniel Bernstein.

**Parameters:**
- Key: 256 bits.
- Nonce: 96 bits (RFC 8439) or 192 bits (XChaCha20 variant).
- Tag: 128 bits.

**How it works:**

```
1. Derive Poly1305 one-time key from ChaCha20(key, nonce, counter=0)
2. Encrypt plaintext with ChaCha20(key, nonce, counter=1...)
3. Compute Poly1305 tag over (AAD || ciphertext || lengths)
```

**Advantages over AES-GCM:**
- No timing side channels — pure ARX design, no table lookups.
- Faster on devices without AES hardware acceleration.
- Nonce reuse is still catastrophic, but XChaCha20 (192-bit nonce) makes random nonces safe even for very high message counts.

**Used by:** TLS 1.3, WireGuard, SSH (OpenSSH), NaCl/libsodium, Signal Protocol, Noise Protocol.

#### Choosing Between AES-GCM and ChaCha20-Poly1305

|Criterion|AES-GCM|ChaCha20-Poly1305|
|---|---|---|
|With AES-NI|Faster|Slightly slower|
|Without AES-NI|Slower, timing side channels|Faster, no side channels|
|Mobile devices|Mixed (depends on SoC)|Generally preferred|
|Server-side|Default in most TLS stacks|Preferred for mobile clients|
|Nonce size|96 bits (standard)|96 bits (or 192 with XChaCha20)|

In TLS 1.3, both are mandatory to implement. The server typically selects AES-GCM for desktop/server clients (which have AES-NI) and ChaCha20-Poly1305 for mobile clients.

#### AES-GCM-SIV (Nonce Misuse Resistant)

A variant that degrades gracefully if a nonce is accidentally reused. Instead of catastrophic failure (leaked auth key), nonce reuse only reveals whether two plaintexts are identical. Slightly slower than GCM. Used in Google's Tink library and environments where nonce uniqueness is hard to guarantee (e.g., distributed systems without coordination).

---

## 5. Asymmetric Cryptography

Asymmetric (public-key) cryptography uses **two mathematically linked keys**: a public key (shared freely) and a private key (kept secret).

```
Key Generation --> (Public Key, Private Key)

Encryption:   Encrypt(Plaintext, Public Key)  --> Ciphertext
Decryption:   Decrypt(Ciphertext, Private Key) --> Plaintext

Signing:      Sign(Hash, Private Key)   --> Signature
Verification: Verify(Hash, Signature, Public Key) --> Valid / Invalid
```

The fundamental property: **it is computationally infeasible to derive the private key from the public key** (under current mathematical assumptions).

Public-key cryptography is **much slower than symmetric** (100-1000x for RSA, 10-50x for ECC). It is therefore used primarily for:
- Exchanging symmetric keys (key agreement).
- Signing messages (authentication, non-repudiation).

Bulk data encryption always uses symmetric ciphers.

### RSA

Based on the difficulty of factoring large numbers. The product of two large primes is easy to compute, but factoring it back is computationally hard.

**Key generation (simplified):**

```
1. Choose two large primes p, q (each ~1024 bits for RSA-2048)
2. Compute n = p * q                    (modulus, 2048 bits)
3. Compute phi(n) = (p-1)(q-1)          (Euler's totient)
4. Choose e (public exponent, typically 65537)
5. Compute d = e^(-1) mod phi(n)        (private exponent)

Public key:  (n, e)
Private key: (n, d)
```

**Encryption:** C = M^e mod n
**Decryption:** M = C^d mod n

**Key sizes and security:**

|RSA key size|Equivalent symmetric bits|Status|
|---|---|---|
|1024|~80|Broken / deprecated|
|2048|~112|Minimum acceptable|
|3072|~128|Recommended|
|4096|~152|High security|

RSA-2048 is the current minimum for production use. NIST recommends RSA-3072+ for use beyond 2030.

**Padding is critical.** Raw "textbook RSA" (M^e mod n directly) is deterministic and vulnerable to many attacks. Always use:

- **OAEP (Optimal Asymmetric Encryption Padding)** for encryption (RSA-OAEP).
- **PSS (Probabilistic Signature Scheme)** for signing (RSA-PSS).
- Never PKCS#1 v1.5 encryption padding (Bleichenbacher attack, 1998 — still exploitable in some TLS implementations as ROBOT attack, 2017).

### Elliptic Curve Cryptography (ECC)

Based on the difficulty of the **elliptic curve discrete logarithm problem** (ECDLP). Given points P and Q = kP on an elliptic curve, finding k is computationally hard.

**Advantages over RSA:**

|Security level (bits)|RSA key size|ECC key size|
|---|---|---|
|128|3072 bits|256 bits|
|192|7680 bits|384 bits|
|256|15360 bits|512 bits|

ECC achieves equivalent security with much smaller keys, leading to:

- Faster computation (key generation, signing, verification).
- Smaller certificates and signatures.
- Less bandwidth (critical for TLS handshakes, IoT).

**Common curves:**

|Curve|Size|Standardized by|Used in|
|---|---|---|---|
|P-256 (secp256r1)|256 bits|NIST|TLS, X.509 certs, AWS, most platforms|
|P-384 (secp384r1)|384 bits|NIST|Government, higher security requirements|
|Curve25519|~128-bit security|Bernstein|WireGuard, Signal, SSH, TLS 1.3, NaCl|
|Ed25519|~128-bit security|Bernstein|SSH keys, signing, DNSSEC|

> **P-256 vs Curve25519:** P-256 is the NIST standard, widely deployed and required for government compliance. Curve25519 is designed to be simpler to implement correctly (fewer footguns — the API prevents many classes of implementation bugs). Both are secure. Modern protocols (WireGuard, Signal) prefer Curve25519. Legacy/compliance systems use P-256.

### Diffie-Hellman (Key Exchange)

Covered in detail in section 8.

### Comparison

|Property|RSA|ECC (P-256)|ECC (Curve25519)|
|---|---|---|---|
|Key generation|Slow (find large primes)|Fast|Very fast|
|Encryption/signing speed|Slow|Fast|Very fast|
|Key size for 128-bit security|3072 bits|256 bits|256 bits|
|Certificate size|Large|Small|Small|
|Quantum resistance|None|None|None|
|Primary use|Legacy, code signing, some CAs|TLS certs, key exchange (ECDHE)|Modern protocols (WireGuard, Signal)|

> **Post-quantum note:** All three (RSA, classic DH, ECC) are broken by a sufficiently large quantum computer running Shor's algorithm. Post-quantum cryptography (PQC) algorithms like ML-KEM (Kyber) and ML-DSA (Dilithium) are being standardized by NIST. TLS 1.3 already supports hybrid key exchange (classical ECDHE + ML-KEM). Transition is underway but will take years.

---

## 6. Hash Functions, MACs, and Passwords

### Cryptographic Hash Functions

A hash function maps arbitrary-length input to a fixed-length output (the digest). It is a one-way function — the input cannot be recovered from the output.

```
Input (any size) ---> Hash Function ---> Digest (fixed size)

SHA-256("hello")     = 2cf24dba5fb0a030e26e83b2ac5b9e29...  (256 bits)
SHA-256("hello.")    = 25c21fd2b1ef2d3e6b1ff58b5f6e5c34...  (256 bits)
```

A single-bit change in the input produces a completely different digest (**avalanche effect**).

### Security Properties

|Property|Definition|Attack it prevents|
|---|---|---|
|Preimage resistance|Given h, cannot find m such that H(m) = h|Reversing a hash to find the input|
|Second preimage resistance|Given m1, cannot find m2 != m1 where H(m1) = H(m2)|Finding a different input with the same hash|
|Collision resistance|Cannot find any (m1, m2) where H(m1) = H(m2)|Forging digital signatures, certificates|

**Birthday paradox:** For an n-bit hash, collisions can be found in approximately 2^(n/2) operations. This is why SHA-256 (128-bit collision resistance) is secure but SHA-1 (only 80-bit collision resistance) is not.

### Hash Functions in Practice

|Hash|Output size|Collision resistance|Status|
|---|---|---|---|
|MD5|128 bits|Broken (~2^18)|Broken. Never use for security. OK for non-security checksums.|
|SHA-1|160 bits|Broken (2^63)|Deprecated. SHAttered attack (2017) produced collisions. Chrome, Git transitioning away.|
|SHA-256|256 bits|128 bits|Secure. Current standard. Used in TLS, Bitcoin, code signing, certificate fingerprints.|
|SHA-384|384 bits|192 bits|Secure. Used when higher margin needed (government).|
|SHA-512|512 bits|256 bits|Secure. Faster than SHA-256 on 64-bit platforms (operates on 64-bit words).|
|SHA-3 (Keccak)|224-512 bits|Up to 256 bits|Secure. Different internal design (sponge construction). Backup if SHA-2 family is ever broken.|
|BLAKE2|256/512 bits|128/256 bits|Secure. Faster than SHA-256 in software. Used in Argon2, WireGuard, many modern systems.|
|BLAKE3|256 bits|128 bits|Secure. Parallelizable, extremely fast. Newer, gaining adoption.|

> **Rule of thumb:** Use SHA-256 unless there is a specific reason for something else. Where speed matters and a library is being chosen, BLAKE2b or BLAKE3 are excellent. Never use MD5 or SHA-1 for any security purpose.

### MAC (Message Authentication Code)

A MAC takes a key and a message and produces a tag that proves both **integrity and authenticity**. Only someone with the key can produce or verify the tag.

```
Tag = MAC(Key, Message)

Sender:   sends (Message, Tag)
Receiver: computes MAC(Key, Message), compares with received Tag
          Match    --> message authentic and unmodified
          Mismatch --> message tampered or wrong key
```

**HMAC (Hash-based MAC):**

The most common construction. Uses a hash function internally.

```
HMAC(K, M) = H((K XOR opad) || H((K XOR ipad) || M))

where ipad = 0x36 repeated, opad = 0x5c repeated
```

- HMAC-SHA256 is the standard. Secure as long as the underlying hash is.
- Used in TLS, API authentication (AWS Signature V4), JWT (HS256), SSH.

**CMAC (Cipher-based MAC):**

Uses a block cipher (AES) instead of a hash. AES-CMAC produces a 128-bit tag. Less common than HMAC but used in some protocols (e.g., IEEE 802.11i).

**Poly1305:**

A one-time MAC. Used in ChaCha20-Poly1305 and NaCl. Extremely fast. Must use a fresh key for every message (derived from the encryption key and nonce).

**MAC vs Hash:**

|Property|Hash (SHA-256)|MAC (HMAC-SHA256)|
|---|---|---|
|Key required|No|Yes|
|Provides integrity|Yes (if hash is distributed securely)|Yes|
|Provides authentication|No|Yes|
|Prevents forgery|No (anyone can compute a hash)|Yes (need the key)|

A hash alone only provides integrity if the hash itself is transmitted securely. A MAC provides both integrity and authenticity because only the key holder can produce a valid tag.

### Password Hashing

Passwords must **never** be stored in plaintext or encrypted (encryption is reversible — if the key leaks, all passwords leak). Instead, passwords are hashed with a **slow, salted, memory-hard function**.

**Why not just SHA-256?**

SHA-256 is designed to be fast. A modern GPU can compute billions of SHA-256 hashes per second, making brute-force attacks on password hashes trivial.

**What we want:** a hash function that is deliberately slow and expensive to compute.

**Salt:** A random value unique to each password. Stored alongside the hash. Prevents:

- **Rainbow tables** (precomputed hash lookups).
- **Identical passwords producing identical hashes** (if two users have the same password, their hashes are different because salts differ).

```
stored = slow_hash(password, salt, cost_parameters)

Verification:
  1. Retrieve stored hash and salt for user
  2. Compute slow_hash(entered_password, salt, cost_parameters)
  3. Compare with stored hash (using constant-time comparison)
```

**Modern password hashing algorithms:**

|Algorithm|Design|Key property|Status|
|---|---|---|---|
|bcrypt|Blowfish-based, adaptive cost|CPU-hard, tunable work factor|Mature, widely used|
|scrypt|Sequential memory-hard|CPU-hard + memory-hard (resists GPU/ASIC)|Good, used by some systems|
|Argon2|Winner of PHC (2015)|CPU-hard + memory-hard + parallelism-tunable|Modern standard, recommended|

**Argon2 variants:**

- **Argon2id:** Recommended for password hashing. Combines Argon2i (data-independent, resists side channels) and Argon2d (data-dependent, resists GPU attacks).
- Typical parameters: 64 MB memory, 3 iterations, 4 parallelism lanes. Tune to take ~0.5-1 second on the target server hardware.

**Pepper:** An additional secret value (not stored in the database) mixed with the password before hashing. If the database is stolen but the pepper (stored in HSM or environment variable) is not, the hashes cannot be attacked offline. Optional but adds defense in depth.

**Constant-time comparison:** When verifying passwords, always compare the full hash, not byte-by-byte with early exit. A timing side channel in comparison can reveal how many bytes of the hash are correct, enabling a byte-at-a-time attack.

```
WRONG:  if computed_hash == stored_hash     (may short-circuit)
RIGHT:  if hmac.compare_digest(a, b)        (Python)
        if crypto.timingSafeEqual(a, b)     (Node.js)
```

---

## 7. Digital Signatures and PKI

### Digital Signatures

Digital signatures provide three properties that MACs cannot:

- **Authentication:** proves who created the message.
- **Integrity:** proves the message was not modified.
- **Non-repudiation:** the signer cannot deny signing (unlike MACs, where both parties share the key and either could have produced the tag).

**Process:**

```
Signing:
  1. Hash the message            digest = SHA-256(message)
  2. Sign the hash               signature = Sign(digest, private_key)
  3. Send (message, signature)

Verification:
  1. Hash the message            digest = SHA-256(message)
  2. Verify the signature        result = Verify(digest, signature, public_key)
  3. If valid --> message is authentic and unmodified
```

```
Signer (has private key)                  Verifier (has public key)

Message                                   Message + Signature
   |                                         |            |
SHA-256                                   SHA-256      Verify(pub_key)
   |                                         |            |
Digest                                    Digest    Recovered digest
   |                                         |            |
Sign(priv_key)                            Compare --------+
   |                                         |
Signature                                 Valid / Invalid
```

**Why hash first?** RSA and ECDSA can only operate on small inputs. Hashing reduces an arbitrarily large message to a fixed-size digest. Also, signing the hash is much faster than signing the entire message.

### Common Signature Algorithms

|Algorithm|Based on|Key/Signature size|Speed|Status|
|---|---|---|---|---|
|RSA-PSS|Integer factoring|3072+ bits key, 3072 bits sig|Slow sign, fast verify|Mature, still used|
|ECDSA|Elliptic curve (P-256)|256 bits key, 512 bits sig|Fast|Standard (TLS, Bitcoin)|
|Ed25519|Elliptic curve (Curve25519)|256 bits key, 512 bits sig|Very fast|Modern standard (SSH, Signal)|
|Ed448|Elliptic curve (Curve448)|448 bits key, 896 bits sig|Fast|Higher security margin|

> **Ed25519 vs ECDSA:** Ed25519 is deterministic (no random nonce needed during signing, eliminating a class of implementation bugs where bad RNG leaks the private key — this famously broke PlayStation 3's ECDSA). Ed25519 is also faster and has a simpler, safer API. Prefer Ed25519 for new systems.

### Public Key Infrastructure (PKI)

PKI answers the question: **how can a public key be verified to belong to the entity it claims to represent?**

Without PKI, an attacker can substitute their own public key (man-in-the-middle). You would encrypt data with the attacker's key, thinking it is the server's key.

**Solution: certificates.** A trusted third party (Certificate Authority, CA) signs a binding between a public key and an identity (domain name, organization).

#### Certificate Chain of Trust

```
Root CA (self-signed, pre-installed in OS/browser trust store)
   |
   |  signs
   v
Intermediate CA (signed by root, used for day-to-day issuance)
   |
   |  signs
   v
Server Certificate (signed by intermediate, contains server's public key)
   |
   |  presented to
   v
Client (browser, TLS library)
```

Why intermediate CAs? Root CA private keys are kept offline in HSMs (Hardware Security Modules). If an intermediate CA is compromised, only its certificates are revoked — the root remains trusted.

#### X.509 Certificate Structure

```
Certificate:
  Version: v3
  Serial Number: unique identifier
  Signature Algorithm: e.g., SHA256withRSA, SHA256withECDSA
  Issuer: CN=Let's Encrypt Authority X3, O=Let's Encrypt
  Validity:
    Not Before: 2025-01-01
    Not After:  2025-04-01
  Subject: CN=example.com
  Subject Public Key Info:
    Algorithm: ECDSA P-256
    Public Key: 04:ab:cd:...
  Extensions:
    Subject Alternative Name (SAN): example.com, www.example.com
    Key Usage: Digital Signature
    Basic Constraints: CA:FALSE
    Authority Key Identifier: (links to issuer's key)
    CRL Distribution Points: http://crl.example.com/
    Authority Information Access:
      OCSP: http://ocsp.example.com/
      CA Issuers: http://certs.example.com/intermediate.crt
  Signature: (CA's signature over all the above)
```

**Key fields for engineers:**

- **SAN (Subject Alternative Name):** the domain(s) the cert is valid for. Modern browsers check SAN, not CN. Wildcard: `*.example.com` covers subdomains (one level only).
- **Validity period:** Let's Encrypt issues 90-day certs (encourages automation). Commercial CAs issue 1-year certs (maximum since 2020).
- **Key Usage / Extended Key Usage:** constrains what the key can be used for (server auth, client auth, code signing).

#### Certificate Validation (What the Client Checks)

```
1. Build the chain: Server cert -> Intermediate -> Root
2. Verify each signature in the chain
3. Check that the root is in the local trust store
4. Check certificate validity dates (not expired, not yet valid)
5. Check the SAN matches the hostname being connected to
6. Check revocation status (CRL or OCSP)
7. Check key usage constraints
```

If any check fails, the connection is rejected (browsers show a warning/error).

#### Certificate Revocation

Certificates need to be invalidated before expiry if the private key is compromised or the certificate was mis-issued.

|Method|How it works|Pros|Cons|
|---|---|---|---|
|CRL (Certificate Revocation List)|CA publishes a list of revoked serial numbers|Simple|Lists grow large, clients must download them|
|OCSP|Client queries CA's OCSP responder in real-time|Current status|Adds latency, privacy concern (CA sees what you visit)|
|OCSP Stapling|Server fetches OCSP response and attaches it to TLS handshake|No client-side query, private|Requires server support|
|CRLite (Firefox)|Compressed CRL distributed via browser updates|Fast, complete|Browser-specific|
|Short-lived certs|Issue certs valid for hours/days, no revocation needed|Simplest|Requires robust automation (ACME)|

In practice, OCSP stapling is the most common server-side approach. Let's Encrypt's 90-day certs and automated renewal (ACME protocol) reduce the window of exposure, making revocation less critical.

#### Let's Encrypt and ACME

Let's Encrypt is a free, automated CA that issues domain-validated (DV) certificates. It uses the ACME (Automatic Certificate Management Environment) protocol:

```
1. Client (certbot/other ACME client) requests certificate for example.com
2. CA issues a challenge: place a specific file at
   http://example.com/.well-known/acme-challenge/TOKEN
   (or add a DNS TXT record for DNS-01 challenge)
3. Client fulfills the challenge
4. CA verifies the challenge (proves domain control)
5. CA issues signed certificate
6. Client installs certificate and configures renewal cron
```

> **Best practice:** automate certificate renewal. Certbot, acme.sh, and cloud provider integrations (AWS ACM, Cloudflare) handle this. Never manually manage certificates in production.

---

## 8. Key Exchange and KDFs

### The Key Distribution Problem

Symmetric encryption requires both parties to share a secret key, but sharing a secret over an insecure channel is itself the problem: sending the key in plaintext allows an eavesdropper to intercept it.

Three solutions exist:

1. **Pre-shared key (PSK):** agree on a key out-of-band (in person, via secure courier). Does not scale.
2. **Asymmetric encryption:** encrypt the symmetric key with the receiver's public key. Requires PKI.
3. **Key exchange protocol:** both parties derive a shared secret without ever transmitting it. Diffie-Hellman.

### Diffie-Hellman Key Exchange

Allows two parties to agree on a shared secret over an insecure channel, without any prior shared secret.

**Mathematical basis (classic DH):**

```
Public parameters (known to everyone): prime p, generator g

Alice:                              Bob:
  Choose secret a                     Choose secret b
  Compute A = g^a mod p               Compute B = g^b mod p
  Send A to Bob    ------>            Send B to Alice  <------

Alice computes:                     Bob computes:
  S = B^a mod p                       S = A^b mod p
    = (g^b)^a mod p                     = (g^a)^b mod p
    = g^(ab) mod p                      = g^(ab) mod p

Both have the same shared secret S = g^(ab) mod p
```

An eavesdropper sees A = g^a mod p and B = g^b mod p, but computing g^(ab) mod p from these is the **Computational Diffie-Hellman (CDH) problem**, which is believed to be hard.

```
Alice                    Eve (eavesdropper)              Bob
  |                         |                              |
  |--- A = g^a mod p ------>|------- A ------------------>|
  |                         |                              |
  |<------ B ---------------|<------ B = g^b mod p --------|
  |                         |                              |
  S = g^(ab) mod p          Knows g, p, A, B              S = g^(ab) mod p
                            Cannot compute g^(ab) mod p
                            (discrete log is hard)
```

### ECDH (Elliptic Curve Diffie-Hellman)

Same concept, but over elliptic curve groups instead of modular arithmetic. Much smaller keys for equivalent security.

```
Public parameters: Curve (e.g., P-256 or X25519), base point G

Alice:                              Bob:
  Choose secret a                     Choose secret b
  Compute A = a * G (point mult)      Compute B = b * G
  Send A                              Send B

Shared secret: S = a * B = b * A = ab * G
```

ECDH with Curve25519 is called **X25519**. It is the default key exchange in TLS 1.3, WireGuard, Signal, and SSH.

### Ephemeral Diffie-Hellman and Forward Secrecy

**Static DH:** Both parties reuse their DH key pairs across sessions. If a private key is compromised, all past and future sessions are compromised.

**Ephemeral DH (DHE / ECDHE):** Both parties generate **fresh, temporary key pairs for every session**. After deriving the shared secret, the ephemeral private keys are discarded.

```
Session 1: Alice generates (a1, A1), Bob generates (b1, B1) --> S1
Session 2: Alice generates (a2, A2), Bob generates (b2, B2) --> S2
Session 3: Alice generates (a3, A3), Bob generates (b3, B3) --> S3

a1, b1 are deleted after session 1
a2, b2 are deleted after session 2
...
```

This provides **forward secrecy (also called perfect forward secrecy — PFS):** even if a long-term private key (e.g., the server's RSA/ECDSA signing key used in TLS) is compromised in the future, past session keys cannot be recovered because the ephemeral keys are gone.

**Forward secrecy is mandatory in TLS 1.3.** All key exchanges use ephemeral ECDHE.

> **Historical context:** TLS 1.2 allowed RSA key exchange (client encrypts a random value with the server's RSA public key). If the server's RSA key leaked, an attacker could decrypt all recorded past sessions. This is why the NSA's collection of encrypted traffic was concerning — one leaked key could unlock years of history. TLS 1.3 eliminated RSA key exchange entirely.

### DH Vulnerabilities

**Man-in-the-middle (MITM):** Raw DH does not authenticate the parties. An attacker can perform DH with Alice (pretending to be Bob) and separately with Bob (pretending to be Alice), relaying messages between them. **Solution:** authenticate the DH public values using digital signatures (this is what TLS does).

**Small subgroup attacks:** Maliciously crafted DH parameters can trick a party into computing a shared secret from a small set of values, making brute-force easy. **Solution:** validate DH parameters, use safe primes, or use ECDH with well-known curves.

**Logjam attack:** If the DH prime p is too small (512 or 1024 bits) or widely shared across servers, precomputation can break DH for all servers using that prime. **Solution:** use 2048-bit DH groups minimum, or preferably ECDH (which does not have this issue).

### KDF (Key Derivation Function)

A key exchange produces a single shared secret. But a protocol typically needs multiple keys: encryption key, MAC key, IVs, and separate keys for each direction.

A KDF derives multiple cryptographically independent keys from a single input.

**HKDF (HMAC-based Key Derivation Function):**

The standard KDF used in TLS 1.3, Signal, Noise Protocol, and most modern protocols.

Two phases:

```
1. Extract: PRK = HMAC(salt, input_key_material)
   Concentrates entropy from a potentially non-uniform input into a pseudorandom key (PRK).

2. Expand: output = HMAC(PRK, info || counter)
   Generates as many bytes of key material as needed.
   'info' is a context string that differentiates different derived keys.

Example (TLS 1.3):
	Shared secret from ECDHE
       |
	HKDF-Extract (with salt)
       |
    PRK (handshake secret)
       |
  +----+----+----+
  |    |    |    |
HKDF-Expand with different labels:

  "client_key"     --> client encryption key
  "server_key"     --> server encryption key
  "client_iv"      --> client IV
  "server_iv"      --> server IV
```

The info/label parameter ensures that even though all keys derive from the same PRK, they are cryptographically independent. Knowing one derived key reveals nothing about the others.

---

## 9. TLS as a Case Study

TLS (Transport Layer Security) secures the majority of Internet traffic. It combines nearly every primitive discussed so far: key exchange, symmetric encryption, MACs, digital signatures, certificates, and key derivation.

### What TLS Provides

|Property|How TLS achieves it|
|---|---|
|Confidentiality|Symmetric encryption (AES-GCM, ChaCha20-Poly1305)|
|Integrity|AEAD tags (built into the encryption mode)|
|Authentication|Server certificate (signed by CA), verified by client|
|Forward secrecy|Ephemeral key exchange (ECDHE)|

### TLS Versions

|Version|Status|Handshake RTTs|Notes|
|---|---|---|---|
|SSL 3.0|Broken (POODLE)|2|Do not use|
|TLS 1.0|Deprecated|2|Vulnerable (BEAST). Disabled in modern browsers|
|TLS 1.1|Deprecated|2|No significant improvements over 1.0|
|TLS 1.2|Supported|2|Still widely used. Many cipher suites (some weak)|
|TLS 1.3|Current standard|1 (0-RTT optional)|Simplified, faster, removed all weak ciphers|

### TLS 1.3 Handshake — Full Detail

```
Client                                              Server

1. ClientHello  ----------------------------------->
   - Protocol version: TLS 1.3
   - Random (32 bytes)
   - Cipher suites: [TLS_AES_256_GCM_SHA384,
                      TLS_CHACHA20_POLY1305_SHA256,
                      TLS_AES_128_GCM_SHA256]
   - Key shares: [X25519 public key,
                   P-256 public key (fallback)]
   - Signature algorithms: [Ed25519, ECDSA-P256-SHA256, RSA-PSS-SHA256]
   - SNI: example.com
   - ALPN: [h2, http/1.1]

                                              2. ServerHello
                              <-----------------------------------
                                 - Random (32 bytes)
                                 - Chosen cipher: TLS_AES_256_GCM_SHA384
                                 - Key share: X25519 public key

                              --- Everything below is encrypted ---

                              3. EncryptedExtensions
                              <-----------------------------------
                                 - ALPN: h2

                              4. Certificate
                              <-----------------------------------
                                 - Server's X.509 cert chain

                              5. CertificateVerify
                              <-----------------------------------
                                 - Signature over handshake transcript
                                   (proves server owns the private key)

                              6. Finished
                              <-----------------------------------
                                 - HMAC over entire handshake transcript

7. Finished  ----------------------------------->
   - HMAC over entire handshake transcript

8. Application Data  <=========================>  Application Data
   (encrypted with derived session keys)
```

**Key differences from TLS 1.2:**

- **1 RTT instead of 2:** Key exchange and cipher suite negotiation happen in the first flight. The server responds with its key share immediately.
- **Server's certificate is encrypted:** An eavesdropper cannot see which certificate the server presents (privacy improvement).
- **No RSA key exchange:** Only ephemeral ECDHE. Forward secrecy is mandatory.
- **No weak ciphers:** Only AEAD modes (AES-GCM, ChaCha20-Poly1305). No CBC, no RC4, no 3DES, no static RSA.
- **Simplified cipher suite naming:** TLS 1.3 cipher suites only specify the AEAD and hash. Key exchange and signature are negotiated separately.

### Key Derivation in TLS 1.3

TLS 1.3 uses a structured HKDF-based key schedule:

```
ECDHE shared secret
        |
   HKDF-Extract
        |
   Handshake Secret
        |
   +----+----+
   |         |
Derive     Derive
   |         |
client_     server_
handshake_  handshake_
traffic_    traffic_
secret      secret
   |         |
(encrypt handshake messages)

After handshake:
   Handshake Secret
        |
   HKDF-Extract (with zero salt)
        |
   Master Secret
        |
   +----+----+
   |         |
client_      server_
application_ application_
traffic_     traffic_
secret       secret
   |         |
(encrypt application data)
```

Each direction (client-to-server, server-to-client) has independent keys and IVs. The key schedule ensures that handshake keys and application keys are cryptographically independent.

### 0-RTT Resumption

If the client has connected before, it cached a **PSK (Pre-Shared Key)** from the previous session. It can send encrypted application data in the very first message:

```
Client                              Server

ClientHello + Key share
+ PSK identity
+ Early data (0-RTT) ------------>

                          <--------  ServerHello + Key share
                                     EncryptedExtensions
                                     Finished

Finished  ----------------------->

Application data  <=============>  Application data
```

**0-RTT data is not replay-protected.** An attacker who captures the ClientHello + 0-RTT data can replay it, causing the server to process the request again. Only use 0-RTT for **idempotent operations** (GET requests, not POST/PUT/DELETE). Many servers disable 0-RTT entirely.

### TLS Record Protocol

After the handshake, application data is transmitted in **TLS records**:

```
TLS Record:
+------+--------+--------+---------------------------+------+
| Type | Legacy | Length |     Encrypted payload     | Tag  |
| (1B) | (2B)   | (2B)  |     (variable)            |(16B) |
+------+--------+--------+---------------------------+------+

Type:    23 (Application Data) — always 23 in TLS 1.3 (real type inside encrypted payload)
Legacy:  0x0303 (TLS 1.2 — for middlebox compatibility)
Length:  payload + tag length
Tag:     AEAD authentication tag (16 bytes for AES-GCM / ChaCha20-Poly1305)
```

The actual content type (handshake, alert, application data) is hidden inside the encrypted payload, providing traffic analysis resistance.

**Maximum record size:** 16,384 bytes of plaintext (2^14). Larger application data is split across multiple records.

### Practical TLS for Engineers

**Testing TLS configuration:**

```bash
# Check what cipher suite and protocol version a server uses
$ openssl s_client -connect example.com:443 -tls1_3

# Show full certificate chain
$ openssl s_client -connect example.com:443 -showcerts

# Check certificate expiry
$ echo | openssl s_client -connect example.com:443 2>/dev/null \
  | openssl x509 -noout -dates

# Test specific cipher suite
$ openssl s_client -connect example.com:443 \
  -cipher ECDHE-RSA-AES256-GCM-SHA384

# SSL Labs scan (web-based)
# https://www.ssllabs.com/ssltest/
```

**Common TLS misconfigurations:**

|Issue|Risk|Fix|
|---|---|---|
|Self-signed cert in production|Browsers reject, no trust chain|Use Let's Encrypt or commercial CA|
|Expired certificate|Connection refused|Automate renewal (certbot, ACM)|
|Supporting TLS 1.0/1.1|Vulnerable to downgrade, POODLE, BEAST|Disable TLS <1.2, prefer 1.3|
|Missing intermediate cert|Chain validation fails on some clients|Include full chain in server config|
|Weak cipher suites (RC4, 3DES, CBC)|Exploitable|Allow only AEAD ciphers|
|No HSTS header|SSL-stripping MITM possible|Add `Strict-Transport-Security` header|
|No OCSP stapling|Slow revocation check or no check|Enable OCSP stapling in server config|
