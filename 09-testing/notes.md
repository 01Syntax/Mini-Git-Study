# Testing with xUnit

## Learning Resources

**Official Docs**
- [xUnit.net documentation](https://xunit.net/docs/getting-started/netcore/cmdline)
- [Unit testing C# with xUnit — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-with-dotnet-test)
- [Best practices for unit tests — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)

**Books**
- *The Art of Unit Testing* by Roy Osherove — the best book on the subject. Third edition covers .NET 6+ and xUnit. Covers what to test, how to structure tests, and what makes a good test.
- *Growing Object-Oriented Software, Guided by Tests* by Freeman & Pryce — teaches TDD and how interface-based design makes testing natural.
- *Unit Testing Principles, Practices, and Patterns* by Vladimir Khorikov — deep coverage of test doubles (mocks, stubs, fakes) and when to use each.

**GitHub**
- [xUnit.net — GitHub](https://github.com/xunit/xunit)

---

## 1. Setup

Add xUnit to your test projects:

```xml
<!-- In tests/MiniGit.Core.Tests/MiniGit.Core.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="xunit" Version="2.9.0" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.0" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\MiniGit.Core\MiniGit.Core.csproj" />
  </ItemGroup>
</Project>
```

Run tests:
```powershell
dotnet test
dotnet test --filter "DisplayName~BlobObject"  # filter to specific tests
dotnet test --verbosity normal                  # show test names
```

---

## 2. `[Fact]` — A Single Test Case

```csharp
public class Sha256HasherTests
{
    [Fact]
    public void Hash_EmptyBytes_ReturnsKnownValue()
    {
        // Arrange
        byte[] empty = Array.Empty<byte>();

        // Act
        string hash = Sha256Hasher.Hash(empty);

        // Assert
        Assert.Equal("e3b0c44298fc1c149afb4c8996fb92427ae41e4649b934ca495991b7852b855", hash);
    }

    [Fact]
    public void Hash_SameInput_AlwaysReturnsSameOutput()
    {
        byte[] content = "hello world"u8.ToArray();
        Assert.Equal(Sha256Hasher.Hash(content), Sha256Hasher.Hash(content));
    }

    [Fact]
    public void Hash_DifferentInputs_ReturnDifferentHashes()
    {
        var hash1 = Sha256Hasher.Hash("hello"u8.ToArray());
        var hash2 = Sha256Hasher.Hash("world"u8.ToArray());
        Assert.NotEqual(hash1, hash2);
    }
}
```

The AAA pattern (Arrange, Act, Assert) makes the test's structure explicit.

---

## 3. `[Theory]` with `[InlineData]` — Parameterized Tests

Run the same test logic with multiple inputs:

```csharp
public class Sha256HasherTheoryTests
{
    [Theory]
    [InlineData("",       "e3b0c44298fc1c149afb4c8996fb92427ae41e4649b934ca495991b7852b855")]
    [InlineData("hello\n","5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03")]
    [InlineData("a",      "ca978112ca1bbdcafac231b39a23dc4da786eff8147c4e72b9807785afee48bb")]
    public void Hash_KnownInput_ReturnsExpectedHash(string input, string expectedHex)
    {
        var bytes = Encoding.UTF8.GetBytes(input);
        var actual = Sha256Hasher.Hash(bytes);
        Assert.Equal(expectedHex, actual);
    }
}
```

Each `[InlineData]` creates a separate test case in the test runner. If one fails, the others still run.

---

## 4. `Assert` Methods

```csharp
Assert.Equal(expected, actual);         // values are equal
Assert.NotEqual(a, b);                  // values are NOT equal
Assert.True(condition);                 // condition is true
Assert.False(condition);                // condition is false
Assert.Null(obj);                       // obj is null
Assert.NotNull(obj);                    // obj is NOT null
Assert.Empty(collection);              // collection has 0 elements
Assert.NotEmpty(collection);           // collection has > 0 elements
Assert.Single(collection);             // collection has exactly 1 element
Assert.Contains(item, collection);     // collection contains item
Assert.StartsWith("prefix", str);
Assert.EndsWith("suffix", str);

// Exception testing
var ex = Assert.Throws<FileNotFoundException>(() => store.Retrieve("nonexistent"));
Assert.Contains("nonexistent", ex.Message);

// Async exception
await Assert.ThrowsAsync<HttpRequestException>(() => client.GetAsync("bad-url"));
```

---

## 5. Testing File System Code

Use a temporary directory. Clean it up after each test.

```csharp
public class FileObjectStoreTests : IDisposable
{
    private readonly string _tempDir;

    public FileObjectStoreTests()
    {
        _tempDir = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(_tempDir);
    }

    public void Dispose()
    {
        if (Directory.Exists(_tempDir))
            Directory.Delete(_tempDir, recursive: true);
    }

    [Fact]
    public void Store_ThenRetrieve_ReturnsOriginalContent()
    {
        // Arrange
        var store = new FileObjectStore(_tempDir);
        var content = "hello, world"u8.ToArray();

        // Act
        var hash = store.Store(content);
        var retrieved = store.Retrieve(hash);

        // Assert
        Assert.Equal(content, retrieved);
    }

    [Fact]
    public void Store_SameContentTwice_OnlyOneFileOnDisk()
    {
        var store = new FileObjectStore(_tempDir);
        var content = "duplicate"u8.ToArray();

        store.Store(content);
        store.Store(content);  // second call — same hash

        var objectsDir = Path.Combine(_tempDir, ".minigit", "objects");
        var fileCount = Directory.GetFiles(objectsDir, "*", SearchOption.AllDirectories).Length;

        Assert.Equal(1, fileCount);  // deduplication
    }

    [Fact]
    public void Retrieve_NonexistentHash_ThrowsWithUsefulMessage()
    {
        var store = new FileObjectStore(_tempDir);
        var ex = Assert.Throws<ObjectNotFoundException>(() => store.Retrieve("abc123"));
        Assert.Contains("abc123", ex.Message);
    }
}
```

---

## 6. Test Naming Convention

A widely-used naming pattern: `MethodName_Condition_ExpectedResult`

```csharp
// Good test names
Hash_EmptyBytes_ReturnsKnownValue
Hash_SameInput_AlwaysReturnsSameOutput
Store_ValidContent_ReturnsHashWith64Chars
Retrieve_NonexistentHash_Throws
Commit_EmptyMessage_ThrowsArgumentException
Commit_EmptyIndex_ThrowsInvalidOperationException
```

A good test name tells you:
1. What method is being tested
2. Under what condition
3. What the expected outcome is

When a test fails, the name alone should tell you what broke.

---

## 7. What to Test in Mini-Git

### Core (pure logic, no I/O)

```csharp
// Hashing
[Fact] Hash_KnownInput_ReturnsKnownHash()
[Fact] Hash_SameInput_AlwaysProducesSameHash()

// Diff engine
[Theory] Diff_TwoIdenticalFiles_ReturnsNoChanges()
[Theory] Diff_AddedLine_ReturnsInsertOperation()
[Theory] Diff_DeletedLine_ReturnsDeleteOperation()

// Commit graph
[Fact] GetAncestors_LinearChain_ReturnsAllAncestors()
[Fact] FindMergeBase_BranchedHistory_ReturnsCommonAncestor()
[Fact] FindMergeBase_SameBranch_ReturnsSelf()
```

### Application (command logic, in-memory I/O)

```csharp
[Fact] Commit_FirstCommit_NoParent()
[Fact] Commit_SecondCommit_HasParentHash()
[Fact] Commit_UpdatesHeadRef()
[Fact] Commit_EmptyMessage_Throws()
[Fact] Commit_EmptyIndex_Throws()
[Fact] Add_ValidFile_AddsToIndex()
[Fact] Log_SingleCommit_ReturnsOneEntry()
[Fact] Log_ThreeCommits_ReturnsThreeInOrder()
```

### Infrastructure (file system, temp directory)

```csharp
[Fact] FileObjectStore_Store_CreatesFileAtExpectedPath()
[Fact] FileObjectStore_Retrieve_ReturnsStoredBytes()
[Fact] FileIndex_Save_CreatesJsonFile()
[Fact] FileIndex_Load_RestoresEntries()
[Fact] FileRefStore_SetGet_RoundTrips()
```

---

## 8. Test Fixtures for Shared Setup

If many tests share the same setup, use `IClassFixture<T>`:

```csharp
public class RepoFixture : IDisposable
{
    public string RepoRoot { get; }
    public IObjectStore ObjectStore { get; }
    public IIndex Index { get; }
    public IRefStore Refs { get; }

    public RepoFixture()
    {
        RepoRoot = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString());
        Directory.CreateDirectory(RepoRoot);
        ObjectStore = new FileObjectStore(RepoRoot);
        Index       = new FileIndex(RepoRoot);
        Refs        = new FileRefStore(RepoRoot);
    }

    public void Dispose() => Directory.Delete(RepoRoot, recursive: true);
}

public class CommitCommandTests : IClassFixture<RepoFixture>
{
    private readonly RepoFixture _fixture;
    public CommitCommandTests(RepoFixture fixture) { _fixture = fixture; }

    [Fact]
    public void FirstCommit_CreatesHeadRef()
    {
        var command = new CommitCommand(_fixture.ObjectStore, _fixture.Index, _fixture.Refs);
        _fixture.Index.Add("file.txt", new IndexEntry("hash", "100644", 10));
        var hash = command.Execute("Alice", "first commit");
        Assert.Equal(hash, _fixture.Refs.GetRef("HEAD"));
    }
}
```

---

## 9. Test Doubles: Fakes vs Mocks

**Fake** — a real but simplified implementation (e.g. `InMemoryObjectStore`).
**Mock** — a generated object that records calls and lets you verify behaviour (using a library like Moq).

For Mini-Git, **fakes are better** than mocks. They:
- Are simpler to set up
- Test realistic behaviour (the fake must satisfy the same contract)
- Do not break when you refactor method signatures

Only use a mock when you need to verify that a specific method was called a specific number of times — for Mini-Git, you rarely need this.

---

## Exercises

**Exercise 1 — Your first Fact**
Write a `[Fact]` that verifies `Sha256Hasher.Hash(Array.Empty<byte>())` returns the known SHA-256 of empty input.
Run it with `dotnet test`. Make it fail first (wrong expected value), then fix it.

**Exercise 2 — Theory for diff**
Write a `[Theory]` for `DiffEngine.Diff` with these cases:
- Identical files → no changes
- One line added → one `+` line
- One line deleted → one `-` line
- One line changed → one `-` and one `+` (no substitution)

**Exercise 3 — File system test**
Write a test for `FileObjectStore` using a temp directory:
- Store some content
- Verify the file exists at the expected path
- Retrieve it and verify it equals the original

**Exercise 4 — Exception test**
Test that `CommitCommand.Execute` throws `ArgumentException` when the message is empty.
Use `Assert.Throws<ArgumentException>`.

**Exercise 5 — Integration test**
Write a full workflow test:
1. Create a temp directory
2. Create a `FileObjectStore`, `FileIndex`, `FileRefStore`
3. Add a file to the index
4. Call `CommitCommand.Execute`
5. Verify `refs.GetRef("HEAD")` is a valid hash
6. Verify `store.Exists(headHash)` is true
