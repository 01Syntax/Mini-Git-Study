# Hash Map (Dictionary)

## Learning Resources

**Official Docs**
- [Dictionary<TKey,TValue> — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2)
- [GetHashCode — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.object.gethashcode)

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 11 (Hash Tables). The authoritative academic treatment. Read sections 11.1–11.3 for the conceptual model and collision resolution strategies.
- *Data Structures and Algorithm Analysis in C#* by Mark Allen Weiss — Chapter 5 (Hashing). More accessible than CLRS with C-style code that maps easily to C#.
- *C# in a Nutshell* — Chapter 7 (Collections), Dictionary section. Practical .NET specifics.

**Articles**
- [How Dictionary works in C# — detailed](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2#remarks)

---

## 1. The Core Idea

A hash map answers the question: *"given this key, what is the associated value?"* in O(1) time on average.

Without a hash map, finding a value by key in a list requires scanning every element — O(n). With a hash map, you go directly to the right location in O(1).

The trick: **use the key's hash as an array index**.

---

## 2. Internal Structure: Buckets

Imagine an array of "buckets" (slots). Each bucket can hold multiple key-value pairs.

```
buckets (array of size 8):
  [0]  →  []
  [1]  →  [("src/main.cs", "hash-a")]
  [2]  →  []
  [3]  →  [("README.md", "hash-b"), ("LICENSE", "hash-c")]  ← collision!
  [4]  →  []
  [5]  →  [("src/util.cs", "hash-d")]
  [6]  →  []
  [7]  →  []
```

**To store** `("README.md", "hash-b")`:
1. `hashCode = "README.md".GetHashCode()` → some integer, e.g. `1547823`
2. `bucketIndex = Math.Abs(hashCode) % 8` → e.g. `3`
3. Append to bucket `[3]`

**To look up** `"README.md"`:
1. `hashCode = "README.md".GetHashCode()` → `1547823`
2. `bucketIndex = 1547823 % 8` → `3`
3. Scan bucket `[3]` for an entry where `key == "README.md"` (linear scan of the bucket)
4. Return `"hash-b"`

If all entries land in the same bucket, lookup degrades to O(n). A good hash function distributes keys evenly across buckets.

---

## 3. Collisions

Two different keys can produce the same `bucketIndex` — this is a collision. The `Dictionary` handles it by **chaining**: each bucket is a linked list of entries.

```
bucket [3]: README.md → "hash-b" → LICENSE → "hash-c" → (end)
```

A well-distributed hash function keeps buckets short (usually 0 or 1 items). Lookup stays close to O(1).

---

## 4. Load Factor and Resize

The **load factor** is `count / capacity`. A high load factor means crowded buckets and more collisions.

`Dictionary<TKey,TValue>` automatically resizes (approximately doubles) when the load factor exceeds a threshold (~0.72). Resizing re-hashes all entries into a larger array — expensive but infrequent.

---

## 5. Dictionary API in Depth

```csharp
var store = new Dictionary<string, byte[]>();

// --- Adding and updating ---
store["abc123"] = content;          // insert or update (no error if key exists)
store.Add("def456", content2);      // insert only — throws if key exists

// --- Lookup ---
var data = store["abc123"];          // throws KeyNotFoundException if missing

// Safe lookup patterns:
if (store.TryGetValue("abc123", out var val))
    Use(val);                        // val is only valid inside this block

byte[]? maybeVal = store.GetValueOrDefault("abc123"); // returns null if missing

// --- Removal ---
store.Remove("abc123");              // no error if key doesn't exist
bool removed = store.Remove("abc123", out var removedValue);

// --- Membership ---
store.ContainsKey("abc123");
store.ContainsValue(someBytes);     // O(n) — scans all values

// --- Iteration ---
foreach (var kvp in store)
    Console.WriteLine($"{kvp.Key}: {kvp.Value.Length} bytes");

foreach (var (key, value) in store)  // deconstruction syntax
    Console.WriteLine(key);

foreach (var key in store.Keys) { }
foreach (var val in store.Values) { }

// --- Count ---
int count = store.Count;

// --- Initialisation ---
var byName = new Dictionary<string, string>
{
    ["README.md"] = "abc123",
    ["Program.cs"] = "def456",
};
```

---

## 6. Key Requirements

A type used as a Dictionary key must correctly implement:
- `GetHashCode()` — must return the same value for equal objects
- `Equals()` — must be consistent with `GetHashCode()`

Built-in types (`string`, `int`, etc.) do this correctly. For custom key types, you often need to override both:

```csharp
// If you use a custom record as a key, C# records implement these correctly by default
public record FileKey(string Path, string Mode);

var dict = new Dictionary<FileKey, string>();
dict[new FileKey("README.md", "100644")] = "hash-abc";
// Later lookup with a new FileKey with the same values works correctly
// because records implement value-based GetHashCode and Equals
```

---

## 7. Mini-Git: The Three Core Dictionaries

### Object Store
```csharp
// In InMemoryObjectStore
private readonly Dictionary<string, byte[]> _store = new();

// hash → raw bytes of the object
_store["5891b5b5..."] = blobBytes;
```

### Staging Index
```csharp
// filename → index entry
private Dictionary<string, IndexEntry> _entries = new();

record IndexEntry(string Hash, string Mode, long Size, DateTimeOffset Modified);

_entries["src/Program.cs"] = new IndexEntry("abc123", "100644", 1024, DateTimeOffset.Now);
```

### Ref Store
```csharp
// branch name → commit hash
private Dictionary<string, string> _refs = new();

_refs["main"]    = "d22a981...";
_refs["feature"] = "abc123...";
```

---

## Exercises

**Exercise 1 — Implement InMemoryObjectStore**
Build a class with:
- `string Store(byte[] content)` — compute hash with `SHA256.HashData`, store in Dictionary, return hash
- `byte[] Retrieve(string hash)` — lookup, throw `KeyNotFoundException` with a good message if missing
- `bool Exists(string hash)` — `ContainsKey`
- `int Count` — number of stored objects
Verify deduplication: storing the same content twice produces one entry.

**Exercise 2 — Phone book**
Build a `Dictionary<string, string>` phone book.
Add 10 entries. Look up 3. Remove 1. Print all entries sorted by name (use LINQ `.OrderBy`).

**Exercise 3 — Word frequency**
Given a paragraph of text, split it into words, and count each word's frequency using a `Dictionary<string, int>`.
Print the 5 most common words.
Hint: `dict[word] = dict.GetValueOrDefault(word) + 1;`

**Exercise 4 — Collision simulation**
Write a method that generates 1000 strings and checks if any two produce the same `GetHashCode() % 10` value.
Count how many "collisions" occur. Then switch to `% 100` and compare.
Observe how a larger bucket array reduces collisions.
