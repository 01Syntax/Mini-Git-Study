# Clean Architecture

## Learning Resources

**Books**
- *Clean Architecture: A Craftsman's Guide to Software Structure and Design* by Robert C. Martin — the primary source. Chapters 5 (Object-Oriented Programming), 22 (The Clean Architecture), and 26 (The Main Component) are most relevant.
- *Dependency Injection Principles, Practices, and Patterns* by Steven van Deursen & Mark Seemann — the definitive book on DI in .NET. Chapter 1 gives the best explanation of why DI exists.
- *Growing Object-Oriented Software, Guided by Tests* by Freeman & Pryce — explains how to design with interfaces and injection from the test-first perspective.

**Official Docs**
- [.NET Architecture Guides — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/)
- [Clean architecture with ASP.NET Core — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/common-web-application-architectures#clean-architecture)

**Articles**
- [The Clean Architecture — Robert C. Martin's blog](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) — the original article (free).
- [Dependency Inversion Principle — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion)

---

## 1. The Problem Clean Architecture Solves

Without architecture, a codebase grows entangled:
- The command that commits also reads files, also calls the API, also formats output
- You cannot test one piece without all the others
- Changing the storage format breaks the commit logic
- You cannot run tests without a real disk, a real network, real git repos

Clean Architecture separates concerns so each part can evolve independently and be tested in isolation.

---

## 2. The Dependency Rule

**Inner layers know nothing about outer layers.**

```
┌──────────────────────────────────────┐
│  CLI  (Delivery)         outer        │
│  ┌────────────────────────────────┐  │
│  │  Application  (Use Cases)      │  │
│  │  ┌──────────────────────────┐  │  │
│  │  │  Core  (Domain)  inner   │  │  │
│  │  └──────────────────────────┘  │  │
│  │  Infrastructure  (I/O)         │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Rules:**
- `Core` → no dependencies. Pure C#. No file system, no HTTP, no JSON, no framework.
- `Application` → depends on `Core` only (uses its interfaces and domain objects)
- `Infrastructure` → depends on `Core` (implements its interfaces). Does NOT reference `Application`.
- `CLI` → depends on everything. Wires it all together. Entry point.

**Why this matters:**
If `Core` does not know about files, you can test every domain rule without a filesystem.
If `Application` does not know about JSON, you can change the storage format without touching command logic.

---

## 3. Project Structure in Practice

```
Mini-Git.sln
src/
  MiniGit.Core/
    Objects/
      GitObject.cs
      BlobObject.cs
      TreeObject.cs
      CommitObject.cs
      TreeEntry.cs
    Graph/
      CommitGraph.cs       ← DAG traversal (log, merge-base)
    Diff/
      DiffEngine.cs        ← Myers algorithm
    Interfaces/
      IObjectStore.cs
      IIndex.cs
      IRefStore.cs
    Hashing/
      Sha256Hasher.cs      ← pure function, no I/O

  MiniGit.Application/
    Commands/
      InitCommand.cs
      AddCommand.cs
      CommitCommand.cs
      LogCommand.cs
      ...
    ICommand.cs

  MiniGit.Infrastructure/
    Storage/
      FileObjectStore.cs   ← implements IObjectStore
      FileRefStore.cs      ← implements IRefStore
    Staging/
      FileIndex.cs         ← implements IIndex

  MiniGit.Cli/
    Program.cs             ← entry point, wires everything
    CommandRouter.cs       ← routes args to Application commands
```

### Project references (.csproj)

```xml
<!-- MiniGit.Core — no references -->
<ProjectReference Include="..." />  <!-- nothing -->

<!-- MiniGit.Application -->
<ProjectReference Include="..\MiniGit.Core\MiniGit.Core.csproj" />

<!-- MiniGit.Infrastructure -->
<ProjectReference Include="..\MiniGit.Core\MiniGit.Core.csproj" />

<!-- MiniGit.Cli -->
<ProjectReference Include="..\MiniGit.Core\..." />
<ProjectReference Include="..\MiniGit.Application\..." />
<ProjectReference Include="..\MiniGit.Infrastructure\..." />
```

---

## 4. Dependency Injection

**Dependency Injection (DI)** means: pass what a class needs into its constructor. Do not create dependencies inside the class.

### Without DI (bad)

```csharp
public class CommitCommand
{
    // These are created inside — CommitCommand controls their lifetime and implementation
    private readonly FileObjectStore _store = new(".minigit");
    private readonly FileIndex _index = new(".minigit");
    private readonly FileRefStore _refs = new(".minigit");
}
```

Problems:
- Cannot test without real files
- Cannot swap implementations (e.g. for an in-memory store)
- Hard to change the constructor arguments of `FileObjectStore` without touching `CommitCommand`

### With DI (correct)

```csharp
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

    public string Execute(string author, string message)
    {
        if (string.IsNullOrWhiteSpace(message))
            throw new ArgumentException("Commit message cannot be empty");

        var entries = _index.GetEntries();
        if (!entries.Any())
            throw new InvalidOperationException("Nothing to commit (empty index)");

        var parentHash = _refs.GetRef("HEAD");

        // Build tree from index, create commit object, store it, update HEAD
        // ...

        return newCommitHash;
    }
}
```

In production (`Program.cs`):
```csharp
var repoRoot = Directory.GetCurrentDirectory();
var store    = new FileObjectStore(repoRoot);
var index    = new FileIndex(repoRoot);
var refs     = new FileRefStore(repoRoot);
var command  = new CommitCommand(store, index, refs);
command.Execute(author: "Alice", message: args[0]);
```

In a test:
```csharp
var store    = new InMemoryObjectStore();
var index    = new InMemoryIndex();
var refs     = new InMemoryRefStore();
refs.SetRef("HEAD", null);  // no parent commit

var command  = new CommitCommand(store, index, refs);
index.Add("README.md", new IndexEntry("abc", "100644", 10));
var hash = command.Execute("Alice", "initial commit");

Assert.NotEmpty(hash);
```

---

## 5. Separation of Concerns

Each class has **one job**. Signs that a class is doing too much:
- Its name contains "and" (`FileStorageAndIndexManager`)
- Its constructor takes more than ~5 parameters
- It is hundreds of lines long
- Changing one feature requires editing it

**Mini-Git responsibilities:**

| Class | Responsibility |
|---|---|
| `Sha256Hasher` | Hash bytes to a hex string. Nothing else. |
| `FileObjectStore` | Read/write object bytes to disk. Does not know what a commit is. |
| `CommitCommand` | Orchestrate a commit. Delegates all I/O to injected interfaces. |
| `CommitGraph` | DAG traversal (log, merge-base). Does not touch disk directly. |
| `DiffEngine` | Myers algorithm. Pure function — takes two string arrays, returns diff lines. |
| `CommandRouter` | Parse CLI args, instantiate the right command, call it. No business logic. |

---

## 6. Testing Without Infrastructure

The payoff of clean architecture: you can test all domain logic without touching the disk.

```csharp
[Fact]
public void Commit_WithValidState_StoresCommitAndUpdatesHead()
{
    // Arrange
    var store = new InMemoryObjectStore();
    var index = new InMemoryIndex();
    var refs  = new InMemoryRefStore();

    index.Add("file.txt", new IndexEntry("blob-hash", "100644", 10));
    refs.SetRef("HEAD", null);  // empty repo

    var command = new CommitCommand(store, index, refs);

    // Act
    var commitHash = command.Execute("Alice <alice@example.com>", "initial commit");

    // Assert
    Assert.NotEmpty(commitHash);
    Assert.True(store.Exists(commitHash));
    Assert.Equal(commitHash, refs.GetRef("HEAD"));
}
```

This test runs in milliseconds, in memory, with no files created or deleted.

---

## Exercises

**Exercise 1 — Draw the dependency graph**
On paper, draw boxes for `Core`, `Application`, `Infrastructure`, `CLI`.
Draw arrows showing which references which.
Then add boxes for `Core.Tests`, `Application.Tests`, `Infrastructure.Tests` and draw their arrows too.

**Exercise 2 — Detect a violation**
If `CommitObject` (in `Core`) imports `System.Text.Json` to serialize itself, is that a violation?
Explain why or why not. Where should serialization live?

**Exercise 3 — Constructor injection**
Write `LogCommand(IObjectStore store, IRefStore refs)`.
The `Execute` method should call `refs.GetRef("HEAD")`, then traverse the commit DAG using `store.Retrieve`, and return a list of `CommitObject`s.
Do NOT use `File` or `Directory` inside `LogCommand`.

**Exercise 4 — In-memory implementations**
Implement `InMemoryObjectStore`, `InMemoryIndex`, and `InMemoryRefStore` — all simple Dictionary-backed classes that implement their respective interfaces.
Use these in tests for `CommitCommand` and `LogCommand`.

**Exercise 5 — Integration wiring**
In `Program.cs`, wire `FileObjectStore`, `FileIndex`, `FileRefStore` into `CommitCommand`.
The `Main` method should determine `repoRoot` from the current directory and construct all dependencies before passing them to the command.
