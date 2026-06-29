# LINQ (Language Integrated Query)

## Learning Resources

**Official Docs**
- [LINQ overview — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)
- [Standard Query Operators — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/linq/standard-query-operators/)
- [101 LINQ Samples — Microsoft Learn](https://learn.microsoft.com/en-us/samples/dotnet/try-samples/101-linq-samples/)

**Books**
- *C# in a Nutshell* — Chapter 9 (LINQ). The most thorough reference, covers lazy evaluation, query syntax vs method syntax, and performance.
- *LINQ in Action* by Fabrice Marguerie et al. — a dedicated deep dive if you want to master it.

---

## 1. What LINQ Is

LINQ is a set of extension methods on `IEnumerable<T>` that let you query and transform collections declarably, instead of writing explicit loops.

Compare:

```csharp
// Without LINQ
var result = new List<string>();
foreach (var entry in entries)
{
    if (entry.Mode == "100644")
        result.Add(entry.Hash);
}
```

```csharp
// With LINQ
var result = entries
    .Where(e => e.Mode == "100644")
    .Select(e => e.Hash)
    .ToList();
```

Both do the same thing. LINQ is not faster — it is more readable and composable.

---

## 2. Core Operators

### `Where` — filter

```csharp
// Keep only entries that are files (not subtrees)
var files = entries.Where(e => e.Mode == "100644");

// Keep commits with a message containing a keyword
var matches = commits.Where(c => c.Message.Contains("fix"));
```

### `Select` — transform (map)

```csharp
// Get just the hashes from a list of entries
var hashes = entries.Select(e => e.Hash);

// Project to a different type
var summaries = commits.Select(c => new { c.Hash, Short = c.Hash[..7], c.Message });
```

### `FirstOrDefault` vs `First`

```csharp
// First — throws InvalidOperationException if no match
var entry = entries.First(e => e.Name == "README.md");

// FirstOrDefault — returns null (or default) if no match
var entry = entries.FirstOrDefault(e => e.Name == "README.md");
if (entry is null) Console.WriteLine("Not found");
```

Use `FirstOrDefault` when absence is possible and normal. Use `First` when absence means a bug.

### `Any` — existence check

```csharp
bool hasBlobs = entries.Any(e => e.Mode == "100644");
bool isEmpty  = !entries.Any();
```

### `All` — universal check

```csharp
bool allBlobs = entries.All(e => e.Mode == "100644");
```

### `Count`

```csharp
int fileCount = entries.Count(e => e.Mode == "100644");
int total     = entries.Count(); // or entries.Count (property, faster for List)
```

### `OrderBy` / `OrderByDescending`

```csharp
var sorted = entries.OrderBy(e => e.Name);
var newest = commits.OrderByDescending(c => c.Timestamp).First();
```

### `ToList`, `ToArray`, `ToDictionary`

LINQ operations are **lazy** — they do not execute until you enumerate the result or call a materializing method.

```csharp
// Materialize into a List (executes the query)
List<string> hashes = entries.Select(e => e.Hash).ToList();

// Materialize into a Dictionary
Dictionary<string, string> byName = entries.ToDictionary(e => e.Name, e => e.Hash);
// byName["README.md"] → "abc123"
```

### `GroupBy`

```csharp
// Group entries by mode
var groups = entries.GroupBy(e => e.Mode);
foreach (var group in groups)
{
    Console.WriteLine($"{group.Key}: {group.Count()} entries");
}
```

---

## 3. Lazy Evaluation

LINQ queries are **lazy** — the code inside `Where`, `Select`, etc. does not run until something iterates the result.

```csharp
var query = entries.Where(e => e.Mode == "100644"); // nothing runs yet

// The filter runs here as the foreach pulls values one by one
foreach (var e in query)
    Console.WriteLine(e.Name);

// Or you can force execution all at once with ToList()
var list = query.ToList(); // runs the filter, stores results in a new List
```

This matters for performance: if you call `.ToList()` early, you lose laziness. If you enumerate twice, the work is done twice. For Mini-Git this is rarely an issue, but it explains why LINQ sometimes surprises people.

---

## 4. Chaining

Operators can be chained because each one returns `IEnumerable<T>`:

```csharp
var top5BlobHashes = entries
    .Where(e => e.Mode == "100644")       // filter to files only
    .OrderBy(e => e.Name)                  // sort alphabetically
    .Take(5)                               // first 5
    .Select(e => e.Hash)                   // extract hashes
    .ToList();                             // materialize
```

---

## 5. Query Syntax (alternative style)

LINQ also has a SQL-like query syntax that compiles to the same method calls:

```csharp
// Query syntax
var result = from e in entries
             where e.Mode == "100644"
             orderby e.Name
             select e.Hash;

// Equivalent method syntax
var result = entries
    .Where(e => e.Mode == "100644")
    .OrderBy(e => e.Name)
    .Select(e => e.Hash);
```

In Mini-Git, the method syntax is more common. Use whichever reads clearer to you.

---

## 6. Mini-Git Scenarios

```csharp
// Build the staging index as a dictionary
Dictionary<string, string> indexMap = indexEntries.ToDictionary(e => e.Filename, e => e.Hash);

// Find files in the index that are not in HEAD's tree
var newFiles = indexEntries.Where(e => !headTreeHashes.Contains(e.Hash)).ToList();

// Find all commit hashes reachable from HEAD (using a loop + LINQ together)
var allMessages = commitChain
    .Select(c => c.Message)
    .Where(m => m.StartsWith("fix"))
    .ToList();

// Build a set of all known object hashes
var knownHashes = new HashSet<string>(objectStore.GetAllHashes());
```

---

## Exercises

**Exercise 1 — Filter and project**
Given `List<TreeEntry> entries` (with Mode, Name, Hash), use LINQ to:
- Get all blob entries (mode `"100644"`)
- Get their names, sorted alphabetically
- Return as `List<string>`

**Exercise 2 — ToDictionary**
Convert `List<(string filename, string hash)>` to `Dictionary<string, string>`.
Then do the same for `List<IndexEntry>` where `IndexEntry` is a record with Filename and Hash properties.

**Exercise 3 — Find or null**
Write `string? FindHashByName(IEnumerable<TreeEntry> entries, string name)` using `FirstOrDefault`.
Return null if not found.

**Exercise 4 — Group and count**
Given a `List<CommitObject>`, group by author name and print:
```
Alice: 5 commits
Bob:   3 commits
```

**Exercise 5 — Real scenario**
Given:
- `IEnumerable<string> indexedFiles` — files in the staging area
- `IEnumerable<string> headFiles`    — files in the last commit's tree

Use LINQ to find:
- Files added (in index but not in HEAD)
- Files deleted (in HEAD but not in index)
- Files that exist in both (potentially modified)
