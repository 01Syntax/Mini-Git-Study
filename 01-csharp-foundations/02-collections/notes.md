# Collections

## Learning Resources

**Official Docs**
- [Collections in .NET — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/collections/)
- [Dictionary<TKey,TValue> — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2)
- [List<T> — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1)
- [HashSet<T> — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.hashset-1)

**Books**
- *C# in a Nutshell* by Joseph & Ben Albahari — Chapter 7 covers all collection types exhaustively.
- *Introduction to Algorithms (CLRS)* by Cormen, Leiserson, Rivest, Stein — Chapter 11 covers hash tables (the data structure behind Dictionary).

---

## 1. List\<T\>

A `List<T>` is a dynamically resized array. Elements are ordered and accessible by index in O(1).

```csharp
var names = new List<string>();
names.Add("Alice");
names.Add("Bob");
names.Add("Charlie");

Console.WriteLine(names[0]);   // "Alice" — index access, O(1)
Console.WriteLine(names.Count); // 3

names.Remove("Bob");            // O(n) — has to scan to find it
names.Insert(1, "Dave");        // O(n) — shifts elements right
```

### When elements are stored
Internally a `List<T>` holds a fixed-size array. When it fills up, it allocates a new array of double the size and copies everything. This "doubling" strategy means most `Add` calls are O(1) — only the occasional resize is expensive.

### When to use List\<T\>
- You need ordered, indexed access
- You mostly add to the end and read by index
- You do not care about fast lookup by a key

### In Mini-Git
`TreeObject` holds its entries as `IReadOnlyList<TreeEntry>` — ordered, no lookup needed.

---

## 2. Dictionary\<TKey, TValue\>

A `Dictionary<TKey, TValue>` maps keys to values. Lookup, insert, and delete are O(1) average.

```csharp
var index = new Dictionary<string, string>(); // filename → hash

index["README.md"] = "abc123";
index["Program.cs"] = "def456";

// Lookup
string hash = index["README.md"];  // throws KeyNotFoundException if missing

// Safe lookup
if (index.TryGetValue("missing.txt", out var val))
    Console.WriteLine(val);
else
    Console.WriteLine("Not found");

// Check existence
bool exists = index.ContainsKey("README.md");

// Iterate
foreach (var (filename, hash2) in index)
    Console.WriteLine($"{filename}: {hash2}");
```

### How it works internally

1. Compute `key.GetHashCode()` → an integer (e.g. 94728)
2. Map to a bucket: `bucketIndex = hashCode % bucketCount`
3. The bucket holds a list of key-value pairs (to handle collisions)
4. On lookup: repeat steps 1–2, then scan the bucket for the exact key

With a good hash function and low load factor (number of items / number of buckets), buckets stay small and lookup is O(1). If many keys collide in the same bucket, lookup degrades to O(n).

```
key: "README.md"
hashCode: 94728
bucketCount: 16
bucket index: 94728 % 16 = 8

buckets:
  [8] → [("README.md", "abc123"), ("styles.css", "zzz")]  ← collision!
```

### When to use Dictionary
- You need O(1) lookup by a key
- You are building a map (hash → object, filename → metadata)
- The key must implement `GetHashCode()` and `Equals()` correctly (strings do)

### In Mini-Git
| Dictionary | Role |
|---|---|
| `Dictionary<string, byte[]>` | In-memory object store (hash → content) |
| `Dictionary<string, IndexEntry>` | Staging index (filename → hash + metadata) |
| `Dictionary<string, string>` | Ref store (branch name → commit hash) |

---

## 3. HashSet\<T\>

A `HashSet<T>` stores unique values. It is a Dictionary with no values — just keys.
Membership check (`Contains`) is O(1).

```csharp
var visited = new HashSet<string>();

visited.Add("abc123");
visited.Add("def456");
visited.Add("abc123"); // duplicate — silently ignored

Console.WriteLine(visited.Count);          // 2
Console.WriteLine(visited.Contains("abc123")); // true — O(1)

// Add returns false if the item was already present
bool isNew = visited.Add("abc123"); // false
```

### Set operations
```csharp
var a = new HashSet<string> { "x", "y", "z" };
var b = new HashSet<string> { "y", "z", "w" };

a.IntersectWith(b);  // a is now {"y", "z"}
a.UnionWith(b);      // a is now {"y", "z", "w"}
a.ExceptWith(b);     // a is now {"x"}
```

### When to use HashSet
- You only need "is this item present?" not "what is the value?"
- You want to deduplicate a collection
- You are traversing a graph and need to track visited nodes without revisiting them

### In Mini-Git
`HashSet<string>` tracks visited commit hashes during DAG traversal in `log` and `merge-base`.

```csharp
var visited = new HashSet<string>();
var current = headHash;
while (current != null)
{
    if (!visited.Add(current)) break;  // Add returns false if already visited
    // process commit
    current = GetParentHash(current);
}
```

---

## 4. Queue\<T\> and Stack\<T\>

These appear in algorithms (BFS and DFS). Covered in detail in `05-algorithms`.

```csharp
// Queue — first in, first out (BFS)
var queue = new Queue<string>();
queue.Enqueue("a");
queue.Enqueue("b");
var first = queue.Dequeue(); // "a"

// Stack — last in, first out (DFS)
var stack = new Stack<string>();
stack.Push("a");
stack.Push("b");
var top = stack.Pop(); // "b"
```

---

## 5. Choosing the Right Collection

| Need | Collection |
|---|---|
| Ordered list, indexed access | `List<T>` |
| Fast lookup by key | `Dictionary<TKey, TValue>` |
| Fast membership check, no duplicates | `HashSet<T>` |
| FIFO processing (BFS) | `Queue<T>` |
| LIFO processing (DFS, undo) | `Stack<T>` |
| Sorted by key | `SortedDictionary<TKey, TValue>` |
| Immutable, shared safely | `ImmutableList<T>`, `ImmutableDictionary<TKey, TValue>` |

---

## Exercises

**Exercise 1 — Dictionary as an object store**
Implement `InMemoryObjectStore` with a `Dictionary<string, byte[]>`.
Add methods `Store(byte[] content) → string` (returns a fake hash = content length as string),
`Retrieve(string hash) → byte[]`, and `Exists(string hash) → bool`.

**Exercise 2 — Deduplication with HashSet**
Given a `List<string>` with duplicates, use a `HashSet<string>` to return only unique items.
Then sort them with `List<T>.Sort()`.

**Exercise 3 — Frequency count**
Given a `List<string>` of words, use a `Dictionary<string, int>` to count how many times each word appears.
Print the top 3 most frequent words.

**Exercise 4 — Graph visited tracking**
Given a chain of commits as a `Dictionary<string, string?>` (hash → parentHash),
traverse from HEAD to root using a `HashSet` to detect if there are any unexpected cycles.
