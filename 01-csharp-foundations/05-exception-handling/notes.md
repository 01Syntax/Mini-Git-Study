# Exception Handling

## Learning Resources

**Official Docs**
- [Exceptions and Exception Handling — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/exceptions/)
- [Exception best practices — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Creating and throwing exceptions — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/exceptions/creating-and-throwing-exceptions)

**Books**
- *C# in a Nutshell* — Chapter 4 (Advanced C#), section on Exception Handling.
- *Framework Design Guidelines* by Cwalina, Gustafsson & Abrams — Chapter 7 (Exceptions). Authoritative guidance on when to throw vs when to return a result.

---

## 1. The Basic Structure

```csharp
try
{
    // Code that might fail
    byte[] content = File.ReadAllBytes(path);
    Process(content);
}
catch (FileNotFoundException ex)
{
    // Handle this specific failure
    Console.Error.WriteLine($"File not found: {ex.FileName}");
}
catch (UnauthorizedAccessException ex)
{
    Console.Error.WriteLine($"Permission denied: {ex.Message}");
}
catch (Exception ex)
{
    // Fallback for anything else — use sparingly
    Console.Error.WriteLine($"Unexpected error: {ex.Message}");
}
finally
{
    // Always runs — whether an exception occurred or not
    // Use for cleanup: closing files, releasing resources
    Cleanup();
}
```

**Catch from most specific to least specific.** If you put `catch (Exception)` first, the more specific catches below it will never run.

---

## 2. The Exception Hierarchy

```
Exception
├── SystemException
│   ├── IOException
│   │   ├── FileNotFoundException
│   │   ├── DirectoryNotFoundException
│   │   └── EndOfStreamException
│   ├── ArgumentException
│   │   ├── ArgumentNullException
│   │   └── ArgumentOutOfRangeException
│   ├── InvalidOperationException
│   ├── KeyNotFoundException
│   ├── NotSupportedException
│   └── OverflowException
└── ApplicationException  (rarely used today)
```

**Common ones you'll use in Mini-Git:**

| Exception | When to use |
|---|---|
| `FileNotFoundException` | Object hash doesn't exist on disk |
| `DirectoryNotFoundException` | `.minigit` directory missing (repo not initialized) |
| `InvalidOperationException` | Operation not valid in current state (e.g. staging area is empty) |
| `ArgumentException` | Bad argument passed to a method (null hash, empty message) |
| Custom exception | `NotARepositoryException`, `ObjectNotFoundException` |

---

## 3. Throwing Exceptions

```csharp
// Throw a built-in exception
throw new FileNotFoundException($"Object not found: {hash}");

// Throw with an inner exception (preserves the original cause)
try
{
    File.ReadAllBytes(path);
}
catch (IOException ex)
{
    throw new ObjectStoreException($"Failed to read object {hash}", ex);
}

// Guard clauses at the top of a method (fail fast)
public byte[] Retrieve(string hash)
{
    ArgumentException.ThrowIfNullOrEmpty(hash, nameof(hash));
    if (hash.Length != 64) throw new ArgumentException("Hash must be 64 hex characters", nameof(hash));
    // ...
}
```

### Custom exceptions

For Mini-Git, create a small exception hierarchy so callers can catch specific failure modes:

```csharp
// Base for all Mini-Git errors
public class MiniGitException : Exception
{
    public MiniGitException(string message) : base(message) { }
    public MiniGitException(string message, Exception inner) : base(message, inner) { }
}

public class NotARepositoryException : MiniGitException
{
    public NotARepositoryException()
        : base("fatal: not a minigit repository (or any parent directory): .minigit") { }
}

public class ObjectNotFoundException : MiniGitException
{
    public ObjectNotFoundException(string hash)
        : base($"fatal: not a valid object name '{hash}'") { }
}
```

---

## 4. When to Throw vs When to Return a Result

This is one of the most important design decisions in a codebase.

### Throw when the caller cannot reasonably continue

```csharp
// The object MUST exist — if it doesn't, something is deeply wrong
public byte[] Retrieve(string hash)
{
    if (!Exists(hash))
        throw new ObjectNotFoundException(hash);
    return File.ReadAllBytes(BuildPath(hash));
}
```

### Return a nullable or bool when absence is expected

```csharp
// A file might or might not be in the index — that's normal
public IndexEntry? TryGetEntry(string filename)
{
    _entries.TryGetValue(filename, out var entry);
    return entry; // null if not found — caller handles it
}

// A branch might not exist
public string? TryGetBranchHash(string branchName)
{
    var path = Path.Combine(_refsDir, branchName);
    return File.Exists(path) ? File.ReadAllText(path).Trim() : null;
}
```

### The "Result pattern" (advanced)

Some teams prefer never throwing and always returning a result type:

```csharp
public record Result<T>(bool Success, T? Value, string? Error);

public Result<byte[]> TryRetrieve(string hash)
{
    if (!Exists(hash))
        return new Result<byte[]>(false, null, $"Object not found: {hash}");
    return new Result<byte[]>(true, File.ReadAllBytes(BuildPath(hash)), null);
}
```

For Mini-Git, using exceptions for truly exceptional cases and nullable returns for expected-absence cases is cleaner than the Result pattern.

---

## 5. Writing Meaningful Error Messages

Look at real Git's error messages for inspiration:

```
fatal: not a git repository (or any of the parent directories): .git
error: pathspec 'badfile.txt' did not match any file(s) known to git
fatal: 'nonexistent-branch' is not a valid branch name
```

Patterns to follow:
- Start with `fatal:` for errors that stop execution, `error:` for recoverable errors
- Include the value that caused the problem
- Tell the user what they can do next (when possible)

```csharp
// Bad
throw new Exception("Error");

// Bad — no context
throw new Exception("File not found");

// Good
throw new ObjectNotFoundException(hash);
// → "fatal: not a valid object name 'abc123'"

// Good — for user-facing output
Console.Error.WriteLine($"error: '{filename}' did not match any file in the index");
```

---

## 6. `using` Statements (RAII Pattern)

For resources that must be released (file handles, network connections, database connections), use `using`:

```csharp
// The stream is automatically closed/disposed when the block exits,
// even if an exception is thrown
using var stream = new FileStream(path, FileMode.Open);
// work with stream
// stream.Dispose() called automatically here
```

In Mini-Git you rarely need this because `File.ReadAllBytes` handles the stream internally. But you need to understand it before reading infrastructure code.

---

## Exercises

**Exercise 1 — Catch specific exceptions**
Write a method `byte[] SafeRead(string path)` that:
- Returns the bytes if the file exists
- Prints "File not found: {path}" and returns `null` if it doesn't
- Prints "Permission denied: {path}" for access errors
- Re-throws any other exception

**Exercise 2 — Custom exception hierarchy**
Define `MiniGitException`, `NotARepositoryException`, and `ObjectNotFoundException`.
Write `Retrieve(string hash)` that throws `ObjectNotFoundException` for missing objects.
Write a `Main` that calls it and catches `MiniGitException` (the base class catches both).

**Exercise 3 — Guard clauses**
Write `CommitCommand.Execute(string message)` with guard clauses:
- Throw `ArgumentException` if `message` is null or empty
- Throw `InvalidOperationException` if the staging index is empty
- Throw `NotARepositoryException` if `.minigit` doesn't exist
Add these checks at the top of the method before doing any real work.
