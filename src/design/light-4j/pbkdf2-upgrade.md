# PBKDF2 Upgrade and CRYPT Format Compatibility

This document describes the proposed upgrade path for PBKDF2 usage in
`light-4j`, with special focus on encrypted configuration values that use the
`CRYPT` prefix.

The immediate operational issue is compatibility. The new `light-encryptor`
emits only the four-part AES-GCM format, while existing deployed configuration
may still contain the legacy three-part AES-CBC format. The `light-4j`
decryptor should read both formats so services can upgrade safely without
requiring every encrypted configuration value to be regenerated at the same
time.

## Background

There are two separate PBKDF2 use cases in `light-4j`:

* `AESSaltDecryptor` derives an AES key from a configured master password for
  decrypting encrypted configuration values.
* `HashUtil` generates and validates PBKDF2 password or API-key hashes.

These two paths should not be upgraded in exactly the same way. OWASP password
storage guidance applies directly to password and API-key hashes, where PBKDF2
is the password hashing function. The `CRYPT` path uses PBKDF2 as a key
derivation function before AES encryption. It has its own compatibility
constraints because the encrypted value format historically did not include an
algorithm version or iteration count.

Current OWASP guidance recommends `PBKDF2-HMAC-SHA256` with 600,000 iterations
for password storage. See the OWASP Password Storage Cheat Sheet:

https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html

## Current CRYPT Formats

### Legacy Three-Part Format

The legacy format is:

```text
CRYPT:<hex-salt>:<hex-ciphertext>
```

The legacy implementation uses:

* `PBKDF2WithHmacSHA256`
* 65,536 iterations
* 256-bit AES key
* `AES/CBC/PKCS5Padding`
* a static zero IV

This format is still present in older deployed service configuration and older
test fixtures. It should be treated as deprecated but readable.

The main security weakness is the static zero IV in CBC mode. With a fixed IV,
the same plaintext encrypted with the same derived key produces the same first
ciphertext block. That can reveal equality or common prefixes across encrypted
configuration values. For this reason, legacy CBC support must be read-only and
the runtime should warn when it decrypts a legacy value.

### Current Four-Part Format

The current `light-encryptor` output format is:

```text
CRYPT:<hex-salt>:<hex-iv>:<hex-ciphertext-and-tag>
```

The current implementation uses:

* `PBKDF2WithHmacSHA256`
* 65,536 iterations
* 256-bit AES key
* `AES/GCM/NoPadding`
* random 16-byte salt
* random 12-byte GCM IV
* 128-bit GCM authentication tag

This is the only format that `light-encryptor` should emit going forward.

## Goals

* Allow `light-4j` runtime decryptors to read both legacy three-part `CRYPT`
  values and current four-part `CRYPT` values.
* Keep `light-encryptor` simple by generating only the current four-part
  AES-GCM format.
* Avoid breaking existing checked-in or deployed encrypted configuration during
  framework upgrades.
* Make the future PBKDF2 cost upgrade explicit instead of overloading an
  unversioned format.
* Preserve clear error messages that identify invalid `CRYPT` formatting.

## Non-Goals

* Do not make `light-encryptor` generate legacy three-part values.
* Do not silently re-encrypt configuration files at runtime.
* Do not change the meaning of existing four-part `CRYPT:salt:iv:hash` values
  without adding a versioned format.
* Do not apply OWASP password-storage iteration guidance blindly to every
  PBKDF2 use without measuring startup and request-time cost.

## Design

### 1. Add Format Dispatch in AESSaltDecryptor

`AESSaltDecryptor.decrypt(String input)` should parse the token count and route
to the correct decryptor:

```text
parts.length == 3 -> decryptLegacyCbc(parts)
parts.length == 4 -> decryptGcm(parts)
otherwise         -> invalid CRYPT format
```

Use `split(":", -1)` so malformed values with empty fields are still detected
as malformed fields instead of being accidentally normalized by `String.split`.

Suggested method shape:

```java
public String decrypt(String input) {
    if (input == null || !input.startsWith(CRYPT_PREFIX + ":")) {
        throw new RuntimeException("Unable to decrypt, input string does not start with 'CRYPT:'.");
    }

    String[] parts = input.split(":", -1);
    return switch (parts.length) {
        case 3 -> decryptLegacyCbc(parts);
        case 4 -> decryptGcm(parts);
        default -> throw invalidCryptFormat();
    };
}
```

The error text should mention both accepted formats:

```text
CRYPT:salt:hash
CRYPT:salt:iv:hash
```

The implementation should validate that required fields are non-empty before
hex decoding. A malformed value should fail as a format error, while a bad
master password, corrupted ciphertext, or failed GCM authentication tag should
fail as a cryptographic error. The public exception message should remain
generic enough that it does not disclose secret material, but the internal log
message should let an operator distinguish invalid structure from failed
decryption.

### 2. Keep Legacy CBC Decryption Read-Only

The legacy path should preserve the previous behavior exactly:

* parse salt from `parts[1]`
* parse ciphertext from `parts[2]`
* derive the AES key with `PBKDF2WithHmacSHA256`, 65,536 iterations, 256-bit key
* decrypt with `AES/CBC/PKCS5Padding`
* use the legacy static zero IV

This path exists only to keep existing configuration readable. It should not be
used by `light-encryptor` for new values.

When this path is used, the decryptor should log a throttled warning, ideally
once per JVM or once per unique legacy value:

```text
Legacy three-part CRYPT value decrypted. Regenerate this secret with the current
light-encryptor to migrate to AES-GCM.
```

The warning must not include the encrypted value, plaintext secret, derived key,
master password, salt, or ciphertext.

### 3. Keep Four-Part GCM as the Default Format

The four-part path should preserve the current behavior:

* parse salt from `parts[1]`
* parse IV from `parts[2]`
* parse ciphertext plus tag from `parts[3]`
* derive the AES key with `PBKDF2WithHmacSHA256`, 65,536 iterations, 256-bit key
* decrypt with `AES/GCM/NoPadding`
* use a 128-bit authentication tag

`light-encryptor` should keep emitting only this format.

### 4. Cache Derived Keys by KDF Parameters

The existing cache key is based on salt. That is enough while all supported
formats use the same PBKDF2 algorithm, iteration count, and key size, but it is
not a good foundation for the next upgrade.

Use a cache key that includes the KDF parameters:

```text
<kdf-algorithm>:<iterations>:<key-size>:<salt-hex>
```

This prevents a future format from accidentally reusing a key derived with old
parameters.

The cache key does not need to include the master password as long as the master
password is fixed for the JVM lifetime, which matches the current
`AutoAESSaltDecryptor` environment-variable model. If a future implementation
allows hot reload of the master password without a JVM restart, the cache must
be cleared on password change or the password identity must become part of the
cache key.

### 5. Avoid Shared Cipher Instances

`Cipher` instances are mutable and should not be shared across concurrent
requests. Each decrypt operation should create and initialize a local `Cipher`.
The cache should store only derived `SecretKeySpec` values.

### 6. Keep Error Handling Actionable

The decryptor should separate three categories internally:

* format errors, such as missing fields, empty fields, invalid hex, or an
  unsupported number of parts
* key derivation errors, such as an unavailable PBKDF2 algorithm
* cryptographic failures, such as an incorrect master password, corrupted
  ciphertext, or AES-GCM tag failure

All categories should avoid logging sensitive values. GCM tag failures should
not be downgraded or ignored; they are the authenticated-encryption signal that
the value cannot be trusted.

## PBKDF2 Iteration Upgrade

### Password and API-Key Hashes

For `HashUtil`, new hashes should move to a versioned format that records the
algorithm and iteration count. For example:

```text
pbkdf2-sha256:600000:<hex-salt>:<hex-hash>
```

Validation should support both:

```text
<iterations>:<hex-salt>:<hex-hash>
pbkdf2-sha256:<iterations>:<hex-salt>:<hex-hash>
```

The legacy three-part password-hash format should continue to validate with its
original algorithm, `PBKDF2WithHmacSHA1`, because the stored value does not
record the algorithm. New hashes should use `PBKDF2WithHmacSHA256` and 600,000
iterations.

Hash comparison should use a constant-time byte-array comparison such as
`MessageDigest.isEqual(...)`. This avoids early-exit comparison behavior when
validating passwords or API keys.

For API keys, request-time cost must be measured before rollout because
`ApiKeyHandler` can call `HashUtil.validatePassword(...)` during request
authentication when hashed API keys are enabled.

`HashUtil` should also expose a helper such as `needsRehash(String storedHash)`
so callers that can persist credentials can upgrade legacy hashes after a
successful validation.

### CRYPT Values

The current four-part `CRYPT` format does not contain an iteration count. If the
decryptor changes four-part values from 65,536 iterations to 600,000 iterations
without a version marker, existing four-part values will become unreadable.

Therefore, the compatibility fix should keep both three-part CBC and four-part
GCM values on the existing 65,536-iteration KDF. A later cost increase for
encrypted configuration should introduce a versioned format, for example:

```text
CRYPT2:<kdf-algorithm>:<iterations>:<hex-salt>:<cipher-algorithm>:<hex-iv>:<hex-ciphertext-and-tag>
```

That future format can use 600,000 iterations, or another measured value, while
the existing `CRYPT` formats remain readable.

Because configuration decryption usually happens during startup and unique salts
can be cached after the first derivation, a higher iteration count is more
practical here than it is for per-request API-key validation. A reasonable
target for the future versioned format is a measured startup cost, for example
100 ms to 200 ms per unique salt on supported deployment hardware. The exact
iteration count should be selected from benchmark data rather than copied
directly from password-storage guidance.

## Rollout Plan

1. Update `light-4j` `AESSaltDecryptor` to read both three-part CBC and
   four-part GCM formats.
2. Add regression tests for both formats with known ciphertext fixtures.
3. Release the compatible `light-4j` decryptor.
4. Keep `light-encryptor` generating only four-part GCM values.
5. Log a throttled warning whenever the legacy three-part CBC path is used.
6. Gradually regenerate old three-part values with the new `light-encryptor`
   when configuration files are touched.
7. Add a scanner or CI check that can report remaining three-part `CRYPT`
   values, but do not fail builds until downstream repositories have migrated.
8. Add a command-line migration utility that can rewrite legacy values after
   decrypting them with the configured master password.
9. Design and implement a separate versioned format before increasing the KDF
   iteration count for `CRYPT` values.

## Test Plan

Add focused tests in `decryptor`:

* decrypts a known legacy `CRYPT:salt:hash` value to the expected plaintext
* decrypts a known current `CRYPT:salt:iv:hash` value to the expected plaintext
* rejects malformed values with fewer than three parts
* rejects malformed values with more than four parts
* rejects empty salt, IV, or ciphertext fields
* verifies `AutoAESSaltDecryptor` and `ManualAESSaltDecryptor` inherit the same
  compatibility behavior

Add config-loading coverage:

* one YAML fixture with a legacy three-part value
* one YAML fixture with a four-part value
* one JSON-style config fixture if the existing config tests cover encrypted
  values in JSON strings

Add migration checks:

```bash
rg -n "CRYPT:[^:[:space:]]+:[^:[:space:]]+(\\s|$)" .
```

This reports likely legacy three-part values. Each match should be reviewed
before replacement because grep is only a locator, not a parser.

## Operational Notes

Services should be upgraded to a compatible `light-4j` decryptor before new
four-part encrypted values are deployed to their configuration. Older runtimes
that only support the three-part CBC format cannot decrypt values generated by
the current `light-encryptor`.

The compatibility decryptor should log enough detail to identify a format
problem, but it must not log plaintext secrets, derived keys, ciphertext bytes,
or the master password.

## Open Questions

* What package should own the migration utility: `light-encryptor`, `light-4j`,
  or a small standalone migration artifact?
* What benchmark baseline should define the future `CRYPT2` KDF cost across
  container, VM, and local developer environments?
* Which `HashUtil` callers can safely persist upgraded hashes after
  `needsRehash(...)` returns true?
