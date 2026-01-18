# Tool: Tink

> Multi-language, cross-platform cryptographic library designed to be secure by default and hard to misuse.

## Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [tink-crypto/tink](https://github.com/tink-crypto/tink) |
| **Languages** | Java, C++, Go, Python, Objective-C |
| **Category** | Cryptography Library |
| **Maintainer** | Google |

## Philosophy

**Core Insight**: Cryptographic APIs are dangerous. Even expert developers make mistakes. The solution isn't better documentation — it's APIs that are **impossible to misuse**.

**Design Principles**:
1. **Secure by default** — No insecure options, safe defaults
2. **Hard to misuse** — Type system prevents errors
3. **Easy to use correctly** — Simple API for common cases
4. **Agile** — Easy key rotation and algorithm migration

**The Tink Bet**: If you can't call a crypto API wrong, you won't call it wrong.

## Theoretical Foundations

### Primitive-Based Design
Tink organizes cryptography by **what you want to do**, not by algorithm:

| Primitive | Purpose | Example Algorithms |
|-----------|---------|-------------------|
| **AEAD** | Authenticated Encryption | AES-GCM, ChaCha20-Poly1305 |
| **MAC** | Message Authentication | HMAC-SHA256 |
| **Digital Signature** | Sign/Verify | ECDSA, Ed25519, RSA-PSS |
| **Hybrid Encryption** | Encrypt to public key | ECIES |
| **Deterministic AEAD** | Same plaintext → same ciphertext | AES-SIV |
| **Streaming AEAD** | Encrypt large data streams | AES-GCM-HKDF |

### Keyset Abstraction
Keys are never exposed raw. They're wrapped in **Keysets**:

```
Keyset = {
    primary_key_id: 12345,
    keys: [
        {key_id: 12345, status: ENABLED, key_data: ...},  // Primary
        {key_id: 12344, status: ENABLED, key_data: ...},  // Old, still valid
        {key_id: 12343, status: DISABLED, key_data: ...}  // Retired
    ]
}
```

**Benefits**:
- Key rotation built-in
- Multiple key versions coexist
- Clear primary for new operations
- Disabled keys can still decrypt old data

### Ciphertext Format
Tink ciphertexts are **self-describing**:

```
[5-byte prefix][actual ciphertext]

Prefix: [1 byte version][4 byte key_id]

Benefits:
- Know which key to use for decryption
- Support key rotation seamlessly
- Detect wrong key immediately
```

## Academic Foundations

| Paper/Resource | Authors | Contribution |
|----------------|---------|--------------|
| [Security and Usability of Crypto APIs](https://www.ieee-security.org/TC/SP2017/papers/161.pdf) | Acar et al. | Why crypto APIs fail |
| [Authenticated Encryption: Relations among Notions](https://link.springer.com/chapter/10.1007/3-540-44448-3_41) | Bellare & Namprempre | AEAD theory |
| [The Security of ChaCha20-Poly1305](https://eprint.iacr.org/2014/613) | Procter | ChaCha20-Poly1305 analysis |
| [Tink Documentation](https://developers.google.com/tink) | Google | Design rationale |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           TINK ARCHITECTURE                              │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        USER CODE                                 │   │
│  │                                                                  │   │
│  │    KeysetHandle handle = ...;                                    │   │
│  │    Aead aead = handle.getPrimitive(Aead.class);                 │   │
│  │    byte[] ct = aead.encrypt(pt, aad);                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      PRIMITIVE LAYER                             │   │
│  │                                                                  │   │
│  │  ┌──────┐  ┌─────┐  ┌───────────┐  ┌────────┐  ┌─────────┐    │   │
│  │  │ AEAD │  │ MAC │  │ Signature │  │ Hybrid │  │Streaming│    │   │
│  │  └──────┘  └─────┘  └───────────┘  └────────┘  └─────────┘    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                       KEYSET LAYER                               │   │
│  │                                                                  │   │
│  │  KeysetHandle: Manages keys, enforces access control            │   │
│  │  KeysetManager: Key rotation, generation                         │   │
│  │  KeysetReader/Writer: Serialization (JSON, Binary)              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     KEY MANAGER LAYER                            │   │
│  │                                                                  │   │
│  │  AesGcmKeyManager, ChaCha20Poly1305KeyManager,                  │   │
│  │  EcdsaSignKeyManager, Ed25519KeyManager, ...                    │   │
│  │                                                                  │   │
│  │  Each KeyManager:                                                │   │
│  │  - Knows one key type                                           │   │
│  │  - Generates keys                                                │   │
│  │  - Creates primitives                                           │   │
│  │  - Validates keys                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     REGISTRY (Singleton)                         │   │
│  │                                                                  │   │
│  │  Maps: key_type_url → KeyManager                                │   │
│  │  Ensures: Only registered, approved algorithms usable           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## What Tink Prevents

| Mistake | How Tink Prevents It |
|---------|---------------------|
| ECB mode | Not available |
| CBC without MAC | Not available (AEAD only) |
| Nonce reuse | Nonces generated internally |
| Weak algorithms | Not registered by default |
| Key as string | Keys are opaque objects |
| Missing authentication | AEAD is default |
| Wrong key size | Validated at generation |
| Partial encryption | All-or-nothing API |

## Usage

### Java: AEAD Encryption
```java
import com.google.crypto.tink.*;
import com.google.crypto.tink.aead.*;

// One-time setup
AeadConfig.register();

// Generate a new keyset
KeysetHandle keysetHandle = KeysetHandle.generateNew(
    PredefinedAeadParameters.AES256_GCM);

// Get the primitive
Aead aead = keysetHandle.getPrimitive(Aead.class);

// Encrypt
byte[] plaintext = "secret message".getBytes(UTF_8);
byte[] associatedData = "context".getBytes(UTF_8);
byte[] ciphertext = aead.encrypt(plaintext, associatedData);

// Decrypt
byte[] decrypted = aead.decrypt(ciphertext, associatedData);
```

### Python: Digital Signatures
```python
import tink
from tink import signature

# Register
signature.register()

# Generate key
private_keyset_handle = tink.new_keyset_handle(
    signature.signature_key_templates.ECDSA_P256)

# Sign
signer = private_keyset_handle.primitive(signature.PublicKeySign)
sig = signer.sign(b"message to sign")

# Get public key and verify
public_keyset_handle = private_keyset_handle.public_keyset_handle()
verifier = public_keyset_handle.primitive(signature.PublicKeyVerify)
verifier.verify(sig, b"message to sign")  # Raises on failure
```

### Go: Key Rotation
```go
import (
    "github.com/tink-crypto/tink-go/v2/aead"
    "github.com/tink-crypto/tink-go/v2/keyset"
)

// Create manager with initial key
manager := keyset.NewManager()
keyID, _ := manager.Add(aead.AES256GCMKeyTemplate())
manager.SetPrimary(keyID)

// Later: Rotate to new key
newKeyID, _ := manager.Add(aead.AES256GCMKeyTemplate())
manager.SetPrimary(newKeyID)
// Old key still works for decryption

// Get handle
handle, _ := manager.Handle()
a, _ := aead.New(handle)
```

### Storing Keys
```java
// Store keyset (encrypted with a master key)
KeysetHandle masterKey = KeysetHandle.generateNew(
    PredefinedAeadParameters.AES256_GCM);
Aead masterAead = masterKey.getPrimitive(Aead.class);

// Write encrypted keyset
TinkJsonProtoKeysetWriter writer = new TinkJsonProtoKeysetWriter(
    new FileOutputStream("keyset.json"));
keysetHandle.write(writer, masterAead);

// Read encrypted keyset
KeysetHandle loaded = KeysetHandle.read(
    TinkJsonProtoKeysetReader.create(new FileInputStream("keyset.json")),
    masterAead);
```

## Key Templates (Pre-configured Secure Defaults)

| Template | Algorithm | Security Level |
|----------|-----------|----------------|
| `AES128_GCM` | AES-128-GCM | 128-bit |
| `AES256_GCM` | AES-256-GCM | 256-bit |
| `CHACHA20_POLY1305` | ChaCha20-Poly1305 | 256-bit |
| `ECDSA_P256` | ECDSA with P-256 | ~128-bit |
| `ED25519` | Ed25519 | ~128-bit |
| `ECIES_P256_HKDF_HMAC_SHA256_AES128_GCM` | Hybrid ECIES | 128-bit |

## When to Use

- Any application needing encryption, signatures, or MACs
- When developers aren't cryptography experts
- When key rotation is a requirement
- Multi-language projects needing consistent crypto
- When you want to prevent crypto misuse by design

## Limitations

- **Learning curve** — Different paradigm from raw crypto
- **Flexibility** — Can't use unsupported algorithms
- **Performance** — Slight overhead vs raw primitives
- **Dependency** — Adds library dependency

## Related Tools

- **Wycheproof** — Test vectors for crypto implementations
- **BoringSSL** — Underlying crypto (for C++ Tink)
- **libsodium** — Alternative secure crypto library

## References

- [Tink Documentation](https://developers.google.com/tink)
- [Tink GitHub](https://github.com/tink-crypto/tink)
- [Tink Security Usability Design Doc](https://github.com/tink-crypto/tink/blob/master/docs/SECURITY-USABILITY.md)
- [Google Security Blog: Tink Posts](https://security.googleblog.com/search/label/tink)
