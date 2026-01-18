# Tool: Wycheproof

> Cryptographic test vectors designed to catch implementation vulnerabilities and edge cases.

## Overview

| Attribute | Value |
|-----------|-------|
| **Repository** | [C2SP/wycheproof](https://github.com/C2SP/wycheproof) (formerly google/wycheproof) |
| **Format** | JSON test vectors |
| **Category** | Cryptographic Testing |
| **Maintainer** | C2SP (Community, Google-originated) |

## Philosophy

**Core Insight**: Most cryptographic vulnerabilities come from **implementation errors**, not algorithm weaknesses. Testing with normal inputs catches bugs; testing with **adversarial inputs** catches vulnerabilities.

**The Name**: Wycheproof is a small mountain in Australia — the smallest mountain in the world. The idea: making your crypto implementation "Wycheproof" means even the smallest attack won't work.

**Design Goals**:
1. Test vectors that **real attackers would try**
2. Cover **known vulnerability classes**
3. Machine-readable format for **automation**
4. Document **why** each test exists

## Theoretical Foundations

### Vulnerability-Driven Testing

Traditional testing: "Does it work correctly with valid inputs?"
Wycheproof: "Does it **fail safely** with adversarial inputs?"

```
Traditional:
  Input: Valid ciphertext
  Expected: Correct plaintext
  
Wycheproof:
  Input: Ciphertext with flipped bit in tag
  Expected: Decryption MUST fail (not return garbage)
```

### Attack Coverage

Wycheproof vectors test for these attack classes:

| Attack Class | What It Tests |
|--------------|---------------|
| **Invalid curve attacks** | Points not on the curve |
| **Small subgroup attacks** | Points in small-order subgroups |
| **Signature malleability** | Multiple valid signatures for same message |
| **Bleichenbacher attacks** | RSA PKCS#1 v1.5 padding oracles |
| **Invalid padding** | Padding oracle vulnerabilities |
| **Nonce reuse** | Behavior when nonce repeats |
| **Edge cases** | Zero, max, boundary values |

### The Mathematics of Edge Cases

**Example: Invalid Curve Attack (ECDH)**

For elliptic curve `y² = x³ + ax + b`:
- Valid points satisfy the equation
- Invalid points don't
- Some implementations don't check!

```
Attacker provides point P not on curve.
Implementation computes k*P (scalar mult).
Result leaks information about private key k.

Wycheproof tests:
- Points with wrong x (not on curve)
- Points at infinity
- Points in twist curve
```

**Example: Small Subgroup Attack**

For curve with order `n = h * q` (h = cofactor, q = prime):
- Small subgroup has order h (often 4 or 8)
- Point in small subgroup: k*P cycles through h values
- Leaks k mod h

```
Wycheproof tests: Points of order 2, 4, 8
Expected: Reject or multiply by cofactor
```

## Test Vector Format

```json
{
  "algorithm": "AES-GCM",
  "generatorVersion": "0.9",
  "numberOfTests": 256,
  "testGroups": [
    {
      "ivSize": 96,
      "keySize": 128,
      "tagSize": 128,
      "type": "AeadTest",
      "tests": [
        {
          "tcId": 1,
          "comment": "basic encryption",
          "key": "5b9604fe14eadba931b0ccf34843dab9",
          "iv": "028318abc1824029138141a2",
          "aad": "",
          "msg": "001d0c231287c1182784554ca3a21908",
          "ct": "26073cc1d851beff176384dc9896d5ff",
          "tag": "0a3ea7a5487cb5f7d70fb6c58d038554",
          "result": "valid"
        },
        {
          "tcId": 75,
          "comment": "Flipped bit 0 in tag",
          "key": "000102030405060708090a0b0c0d0e0f",
          "iv": "505152535455565758595a5b",
          "aad": "",
          "msg": "202122232425262728292a2b2c2d2e2f",
          "ct": "eb156d081ed6b6b55f4612f021d87b39",
          "tag": "d9847dbc326a06e988c77ad3863e6083",  // Wrong tag!
          "result": "invalid"
        }
      ]
    }
  ]
}
```

### Result Values
- `"valid"` — Must succeed
- `"invalid"` — Must fail
- `"acceptable"` — Implementation-defined (both OK)

## Academic Papers

| Paper | Authors | Year | Related Vulnerability |
|-------|---------|------|----------------------|
| [Practical Invalid Curve Attacks](https://eprint.iacr.org/2015/824) | Jager et al. | 2015 | Invalid curve attacks |
| [Bleichenbacher Attack Revisited](https://robotattack.org/) | Böck et al. | 2017 | RSA padding oracle |
| [Return of Bleichenbacher's Oracle Threat](https://www.usenix.org/conference/usenixsecurity14/technical-sessions/presentation/meyer) | Meyer et al. | 2014 | TLS RSA attacks |
| [Twenty Years of Attacks on RSA](https://crypto.stanford.edu/~dabo/pubs/papers/RSA-survey.pdf) | Boneh | 1999 | RSA implementation pitfalls |
| [Mining Your Ps and Qs](https://factorable.net/paper.html) | Heninger et al. | 2012 | Weak key generation |

## Available Test Suites

| Suite | Algorithms | Tests |
|-------|------------|-------|
| **AEAD** | AES-GCM, ChaCha20-Poly1305, AES-CCM | Tag manipulation, nonce handling |
| **Signature** | ECDSA, EdDSA, RSA-PSS, RSA-PKCS1 | Malleability, invalid signatures |
| **ECDH** | P-256, P-384, P-521, X25519 | Invalid curves, small subgroups |
| **RSA** | PKCS#1 v1.5, OAEP | Padding oracles, invalid padding |
| **MAC** | HMAC, Poly1305 | Truncation, key handling |
| **Key Wrap** | AES-KW | Integrity, padding |
| **HKDF** | HKDF-SHA256, SHA512 | Edge cases |

## Usage

### Download Test Vectors
```bash
git clone https://github.com/C2SP/wycheproof
cd wycheproof/testvectors_v1
```

### Python Example
```python
import json

def test_aesgcm_implementation(your_aesgcm_decrypt):
    with open('aes_gcm_test.json') as f:
        data = json.load(f)
    
    for group in data['testGroups']:
        for test in group['tests']:
            key = bytes.fromhex(test['key'])
            iv = bytes.fromhex(test['iv'])
            ct = bytes.fromhex(test['ct'])
            tag = bytes.fromhex(test['tag'])
            aad = bytes.fromhex(test['aad'])
            expected = test['result']
            
            try:
                plaintext = your_aesgcm_decrypt(key, iv, ct, tag, aad)
                if expected == 'invalid':
                    print(f"FAIL: Test {test['tcId']} should have failed")
                    print(f"  Comment: {test['comment']}")
            except Exception:
                if expected == 'valid':
                    print(f"FAIL: Test {test['tcId']} should have succeeded")
                    print(f"  Comment: {test['comment']}")
```

### Go Example
```go
package crypto_test

import (
    "crypto/aes"
    "crypto/cipher"
    "encoding/hex"
    "encoding/json"
    "os"
    "testing"
)

type TestVector struct {
    TcId    int    `json:"tcId"`
    Comment string `json:"comment"`
    Key     string `json:"key"`
    Iv      string `json:"iv"`
    Aad     string `json:"aad"`
    Msg     string `json:"msg"`
    Ct      string `json:"ct"`
    Tag     string `json:"tag"`
    Result  string `json:"result"`
}

func TestAESGCM(t *testing.T) {
    // Load test vectors
    data, _ := os.ReadFile("aes_gcm_test.json")
    var suite struct {
        TestGroups []struct {
            Tests []TestVector `json:"tests"`
        } `json:"testGroups"`
    }
    json.Unmarshal(data, &suite)
    
    for _, group := range suite.TestGroups {
        for _, tc := range group.Tests {
            key, _ := hex.DecodeString(tc.Key)
            iv, _ := hex.DecodeString(tc.Iv)
            ct, _ := hex.DecodeString(tc.Ct + tc.Tag)
            aad, _ := hex.DecodeString(tc.Aad)
            
            block, _ := aes.NewCipher(key)
            gcm, _ := cipher.NewGCM(block)
            
            _, err := gcm.Open(nil, iv, ct, aad)
            
            if tc.Result == "valid" && err != nil {
                t.Errorf("Test %d failed: %s", tc.TcId, tc.Comment)
            }
            if tc.Result == "invalid" && err == nil {
                t.Errorf("Test %d should have failed: %s", tc.TcId, tc.Comment)
            }
        }
    }
}
```

## Key Vulnerabilities Caught by Wycheproof

| Bug | Impact | Tests |
|-----|--------|-------|
| **Missing curve validation** | Private key recovery | ecdh_*_test.json |
| **Accepting trivial signature** | Signature bypass | ecdsa_*_test.json |
| **DSA bias** | Key recovery | dsa_test.json |
| **GCM tag truncation** | Forgery | aes_gcm_test.json |
| **RSA padding oracle** | Decryption oracle | rsa_oaep_*_test.json |
| **ECDSA malleability** | Transaction replay | ecdsa_*_test.json |

## When to Use

- Testing any cryptographic implementation
- CI/CD for crypto libraries
- Validating third-party crypto code
- Security audits of crypto usage
- Before deploying new crypto library

## Integration Tips

1. **Run all relevant suites** — Don't cherry-pick
2. **Treat "invalid" failures as critical** — Security bugs
3. **Check "acceptable" handling** — Document your choice
4. **Update regularly** — New vectors added
5. **Automate in CI** — Catch regressions

## Limitations

- **Coverage** — Not exhaustive, focused on known attacks
- **Algorithm support** — May not cover niche algorithms
- **Implementation details** — Some vectors are language-specific

## Related Tools

- **Tink** — Uses Wycheproof internally
- **BoringSSL** — Also uses Wycheproof
- **cryptofuzz** — Fuzzing for crypto (complementary)

## References

- [Wycheproof GitHub](https://github.com/C2SP/wycheproof)
- [Project Wycheproof Wiki](https://github.com/google/wycheproof/wiki)
- [Test Vector Documentation](https://github.com/C2SP/wycheproof/blob/master/doc/files.md)
