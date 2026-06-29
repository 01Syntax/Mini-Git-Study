# Interfaces & Abstractions

## Learning Resources

**Official Docs**
- [Interfaces — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/interfaces)
- [Interface design guidelines — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/interface)

**Books**
- *C# in a Nutshell* — Chapter 3 (Types), section on interfaces.
- *Clean Code* by Robert C. Martin — Chapter 6 (Objects and Data Structures). While not C#-specific, it explains why hiding implementation behind interfaces is fundamental to maintainable code.
- *Dependency Injection Principles, Practices, and Patterns* by Steven van Deursen & Mark Seemann — the definitive book on programming to interfaces and injecting dependencies.

**Articles**
- [Dependency Inversion Principle — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion)

---

## 1. What Is an Interface?

An interface is a contract. It says: *"any type that implements me guarantees these methods exist."*
The interface says **what** — the implementing class says **how**.

```csharp
// The contract — no implementation allowed
public interface IObjectStore
{
    // Store content, return its hash
    string Store(byte[] content);

    // Retrieve content by hash — throws if not found
    byte[] Retrieve(string hash);

    // Returns true if the hash exists in the store
    bool Exists(string hash);
}
```

Anything that says `: IObjectStore` must provide all three methods. The compiler enforces this.

---

## 2. Implementing an Interface

```csharp
// File-based implementation — reads/writes to .minigit/objects/
public class FileObjectStore : IObjectStore
{
    private readonly string _objectsDir;

    public FileObjectStore(string repoRoot)
    {
        _objectsDir = Path.Combine(repoRoot, ".minigit", "objects");
    }

    public string Store(byte[] content)
    {
        var hash = ComputeHash(content);
        var dir  = Path.Combine(_objectsDir, hash[..2]);
        var path = Path.Combine(dir, hash[2..]);
        Directory.CreateDirectory(dir);
        File.WriteAllBytes(path, content);
        return hash;
    }

    public byte[] Retrieve(string hash)
    {
        var path = Path.Combine(_objectsDir, hash[..2], hash[2..]);
        if (!File.Exists(path))
            throw new FileNotFoundException($"Object not found: {hash}");
        return File.ReadAllBytes(path);
    }

    public bool Exists(string hash)
    {
        var path = Path.Combine(_objectsDir, hash[..2], hash[2..]);
        return File.Exists(path);
    }

    private static string ComputeHash(byte[] content)
    {
        var bytes = SHA256.HashData(content);
        return Convert.ToHexString(bytes).ToLower();
    }
}
```

---

## 3. In-Memory Implementation (for tests)

```csharp
// In-memory implementation — fast, no disk, used in tests
public class InMemoryObjectStore : IObjectStore
{
    private readonly Dictionary<string, byte[]> _store = new();

    public string Store(byte[] content)
    {
        var hash = ComputeHash(content);
        _store[hash] = content;
        return hash;
    }

    public byte[] Retrieve(string hash)
    {
        if (!_store.TryGetValue(hash, out var content))
            throw new KeyNotFoundException($"Object not found: {hash}");
        return content;
    }

    public bool Exists(string hash) => _store.ContainsKey(hash);

    private static string ComputeHash(byte[] content)
    {
        var bytes = SHA256.HashData(content);
        return Convert.ToHexString(bytes).ToLower();
    }
}
```

---

## 4. Why This Matters: Dependency Injection

If `CommitCommand` creates `FileObjectStore` directly inside itself, you can never test it without real files:

```csharp
// WRONG — hard dependency, untestable
public class CommitCommand
{
    private readonly FileObjectStore _store = new(".minigit"); // ← baked in
}
```

If `CommitCommand` takes `IObjectStore` in its constructor, you control what gets passed:

```csharp
// RIGHT — depends on the interface, not the implementation
public class CommitCommand
{
    private readonly IObjectStore _store;
    private readonly IIndex _index;
    private readonly IRefStore _refs;

    public CommitCommand(IObjectStore store, IIndex index, IRefStore refs)
    {
        _store = store;
        _index = index;
        _refs  = refs;
    }

    public string Execute(string message)
    {
        // Build a tree from the index
        // Create a commit object
        // Store it
        // Update HEAD
        // ...
    }
}
```

In `Program.cs` (production):
```csharp
var store   = new FileObjectStore(repoRoot);
var index   = new FileIndex(repoRoot);
var refs    = new FileRefStore(repoRoot);
var command = new CommitCommand(store, index, refs);
```

In a test:
```csharp
var store   = new InMemoryObjectStore();   // fast, no disk
var index   = new InMemoryIndex();
var refs    = new InMemoryRefStore();
var command = new CommitCommand(store, index, refs);
// test command.Execute("my message")
```

---

## 5. Interface Segregation

Do not put everything in one giant interface. Split by responsibility.

```csharp
// Too broad — forces implementations to do unrelated things
public interface IGitStorage
{
    void StoreBlob(byte[] content);
    void StoreRef(string name, string hash);
    void WriteIndex(Dictionary<string, IndexEntry> entries);
}

// Better — split into focused contracts
public interface IObjectStore { ... } // blobs, trees, commits
public interface IRefStore    { ... } // HEAD, branches, tags
public interface IIndex       { ... } // staging area
```

---

## 6. The Three Interfaces in Mini-Git

```csharp
public interface IObjectStore
{
    string Store(byte[] content);
    byte[] Retrieve(string hash);
    bool Exists(string hash);
}

public interface IRefStore
{
    string? GetRef(string name);          // e.g. GetRef("HEAD"), GetRef("refs/heads/main")
    void SetRef(string name, string hash);
    IEnumerable<string> ListBranches();
}

public interface IIndex
{
    void Add(string filename, string hash, string mode);
    void Remove(string filename);
    IReadOnlyDictionary<string, IndexEntry> GetEntries();
    void Save();
    void Load();
}
```

---

## Exercises

**Exercise 1 — Implement both versions**
Write `IObjectStore`, `FileObjectStore`, and `InMemoryObjectStore`.
Write a test that runs the same assertion against both implementations (parameterized test with `[MemberData]`).

**Exercise 2 — Swap without changing the caller**
Write `PrintAllObjects(IObjectStore store)` that prints all stored hashes.
Call it with `FileObjectStore`, then with `InMemoryObjectStore` — no change to `PrintAllObjects`.

**Exercise 3 — Design IRefStore**
Without looking at the notes above, design `IRefStore` from scratch.
Think about: what does `checkout` need to read? What does `commit` need to write?
After you have your design, compare it to the one above and note the differences.
