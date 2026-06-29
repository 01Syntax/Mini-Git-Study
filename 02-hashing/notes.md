# Hashing / SHA-256

## Learning Resources

**Official Docs**
- [Cryptographic Services — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/security/cryptographic-services)
- [SHA256 class — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.security.cryptography.sha256)
- [Convert.ToHexString — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.convert.tohexstring)

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 11 covers hash functions in the context of hash tables.
- *Serious Cryptography* by Jean-Philippe Aumasson — Chapter 6 (Hash Functions). A practitioner-level deep dive into SHA-256 and cryptographic properties. Not required for Mini-Git but excellent background.

**Articles / Videos**
- [SHA-256 — Wikipedia](https://en.wikipedia.org/wiki/SHA-2) — good overview of the algorithm and its properties.
- [How SHA-256 works step by step (visual)](https://sha256algorithm.com/) — interactive walkthrough of the algorithm internals.
- [Git Objects chapter from Pro Git](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) — shows exactly how Git uses hashing for content-addressing.

---

## 1. What Makes a Hash Function Useful

A **hash function** maps arbitrary input data to a fixed-size output (called a digest or hash).

The three properties that make a hash function useful for storage and security:

### 1.1 Deterministic
The same input **always** produces the same output, no matter when, where, or how many times you run it.

```
SHA-256("hello\n") = 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
SHA-256("hello\n") = 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  (same)
SHA-256("hello\n") = 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  (always)
```

This is what makes content-addressable storage possible: hash the content once, use the result as the filename forever.

### 1.2 One-Way (Pre-image Resistant)
You cannot reverse the hash. Given `5891b5b5...` you cannot figure out that the input was `"hello\n"` except by trying every possible input (brute force, computationally infeasible for SHA-256).

This matters for integrity: storing hashes instead of content means you can verify content without storing passwords in plain text (though Mini-Git doesn't do password hashing).

### 1.3 Collision-Resistant
It is computationally infeasible to find two different inputs that produce the same output.

```
SHA-256("file1 content") ≠ SHA-256("file2 content")   ← almost certainly
```

For SHA-256, the probability of a collision is 1 in 2^256 — effectively zero for any practical purpose. SHA-1 (which real Git uses) has known theoretical weaknesses; SHA-256 does not.

### 1.4 Avalanche Effect
A tiny change in input produces a completely different output. There is no way to tell how similar two inputs are from their hashes.

```
SHA-256("hello\n") = 5891b5b5...
SHA-256("Hello\n") = 185f8db3...  ← completely different despite one character change
```

---

## 2. SHA-256 Specifics

- Output: 256 bits = 32 bytes = 64 hex characters
- Designed by the NSA, standardised by NIST (FIPS 180-4)
- Considered cryptographically secure as of 2025 — no practical attacks

A 64-character hex string like:
```
e3b0c44298fc1c149afb4c8996fb924 27ae41e4649b934ca495991b7852b855
```
This is the SHA-256 hash of an empty input (zero bytes).

---

## 3. Computing SHA-256 in C\#

```csharp
using System.Security.Cryptography;
using System.Text;

// --- Method 1: Hash raw bytes ---
byte[] content = File.ReadAllBytes("file.txt");
byte[] hashBytes = SHA256.HashData(content);
string hex = Convert.ToHexString(hashBytes).ToLower();
// hex is a 64-character lowercase hex string

// --- Method 2: Hash a string ---
byte[] textBytes = Encoding.UTF8.GetBytes("hello\n");
byte[] hashBytes2 = SHA256.HashData(textBytes);
string hex2 = Convert.ToHexString(hashBytes2).ToLower();

// --- Method 3: Hash incrementally (streaming, for large files) ---
using var sha = SHA256.Create();
using var stream = File.OpenRead("large-file.bin");
byte[] hashBytes3 = sha.ComputeHash(stream);
string hex3 = Convert.ToHexString(hashBytes3).ToLower();
```

### Why `.ToLower()`?
`Convert.ToHexString` returns uppercase hex (`"5891B5B5..."`). By convention, git object hashes are lowercase. Always normalise to lowercase.

### The `SHA256.HashData` static method
Available since .NET 5. Preferred over the older pattern of calling `SHA256.Create()` for one-off hashes — it is slightly more efficient (avoids allocating a `SHA256` instance you then discard).

---

## 4. What "Hex" Means

Raw SHA-256 output is 32 **bytes** (binary data). Binary bytes can be any value from 0 to 255, including non-printable characters. You cannot use them directly as filenames or print them to a terminal.

**Hex encoding** converts each byte to two printable characters using the digits `0–9` and `a–f`:
```
byte value 0   → "00"
byte value 10  → "0a"
byte value 255 → "ff"
```

So 32 bytes → 64 hex characters. The output is safe to use as a filename, URL, or anywhere text is expected.

---

## 5. Content-Addressable Storage

In a traditional filesystem, files have arbitrary names. You need a separate index to find them.

In content-addressable storage (CAS), **the hash of the content IS its address**. There is no separate index — you know where any object lives as soon as you hash it.

```
content = "hello\n"
hash    = "5891b5b5..."
path    = ".minigit/objects/58/91b5b5..."
```

**Consequences:**
- **Deduplication is free.** If you add the same file twice, it hashes to the same address. The second write just overwrites with identical bytes (or you check with `Exists()` and skip it).
- **Integrity is built-in.** To verify an object, re-hash it and check that the hash matches its filename. Any corruption is immediately detectable.
- **History is immutable.** Changing a file's content changes its hash, which changes the tree's hash, which changes the commit's hash. You cannot silently alter history.

---

## 6. Real Git's Object Format

Real Git does not just hash raw content. It prepends a header:

```
"blob {content-length}\0{content}"
```

Example for a file containing `"hello\n"` (6 bytes):
```
"blob 6\0hello\n"
```

Then hashes the whole thing. That is why `git hash-object file.txt` gives a different hash than `sha256sum file.txt`.

Mini-Git will use a similar approach. You can decide on the exact format, but using a header makes it harder to accidentally confuse a blob hash for a tree hash.

---

## 7. Building a Hasher Class

```csharp
using System.Security.Cryptography;

public static class Sha256Hasher
{
    public static string Hash(byte[] content)
    {
        var bytes = SHA256.HashData(content);
        return Convert.ToHexString(bytes).ToLower();
    }

    public static string HashText(string text)
    {
        var bytes = Encoding.UTF8.GetBytes(text);
        return Hash(bytes);
    }

    // Verify that an object's content matches its claimed hash
    public static bool Verify(string hash, byte[] content)
    {
        return Hash(content) == hash;
    }
}
```

---

## Exercises

**Exercise 1 — Basic hashing**
Write a console app that reads a file path from the command line and prints its SHA-256 hash.
Run it on several files. Verify one result against `sha256sum` (Linux/Mac) or `certutil -hashfile file SHA256` (Windows).

**Exercise 2 — Known-value test**
The SHA-256 hash of an empty string is:
`e3b0c44298fc1c149afb4c8996fb92427ae41e4649b934ca495991b7852b855`
Write a `[Fact]` test that verifies `Sha256Hasher.Hash(Array.Empty<byte>())` equals that value.

**Exercise 3 — Content-addressable store**
Implement `InMemoryContentStore`:
- `string Store(byte[] content)` — hashes, stores, returns hash
- `byte[] Retrieve(string hash)` — returns content or throws
- `bool Exists(string hash)` — membership check

Store `"hello\n"` twice. Verify that `Count` is still 1 (deduplication).

**Exercise 4 — Avalanche effect**
Hash `"the quick brown fox"`, then `"The quick brown fox"` (capital T).
How many hex characters differ? Is there any pattern in which bits changed?
(Spoiler: roughly half the bits flip — this is the avalanche effect.)

**Exercise 5 — Object path from hash**
Given hash `"5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03"`, compute the file path:
`.minigit/objects/58/91b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03`
Use `hash[..2]` and `hash[2..]` (C# range syntax). Verify the result is correct.
