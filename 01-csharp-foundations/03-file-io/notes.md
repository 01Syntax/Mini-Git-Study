# File I/O

## Learning Resources

**Official Docs**
- [File and Stream I/O — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/io/)
- [File class — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.io.file)
- [Path class — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.io.path)
- [Directory class — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.io.directory)

**Books**
- *C# in a Nutshell* — Chapter 15 (Streams and I/O). Covers everything from simple helpers to low-level stream APIs.
- *Pro C# 10 and .NET 6* — Chapter 20 covers file I/O with modern .NET APIs.

---

## 1. Reading and Writing Files

### Text files (for config, index, refs)

```csharp
// Read entire file as a string
string text = File.ReadAllText("path/to/file.txt");

// Write entire file (creates or overwrites)
File.WriteAllText("path/to/file.txt", "content here");

// Append to existing file (does not overwrite)
File.AppendAllText("path/to/file.txt", "\nmore content");

// Read line by line (efficient for large files)
string[] lines = File.ReadAllLines("path/to/file.txt");
foreach (var line in lines) { ... }
```

### Binary files (for objects — blobs, trees, commits)

```csharp
// Read entire file as raw bytes
byte[] data = File.ReadAllBytes("path/to/object");

// Write raw bytes
File.WriteAllBytes("path/to/object", data);
```

### Why the distinction matters
- `ReadAllText` / `WriteAllText` apply an encoding (default UTF-8). They insert a BOM on some platforms and convert line endings. This corrupts binary data.
- `ReadAllBytes` / `WriteAllBytes` are completely faithful — they never touch the bytes.
- Rule: use `AllBytes` for git objects. Use `AllText` for human-readable files like the index or HEAD.

---

## 2. Path Construction

**Never concatenate paths with string `+` or `\`.**
On Windows paths use `\`. On Linux/Mac they use `/`. Hardcoding separators breaks cross-platform code.
`Path.Combine` handles this correctly on every OS.

```csharp
// WRONG — breaks on Linux
string bad = ".minigit" + "\\" + "objects" + "\\" + hash;

// RIGHT — works on all platforms
string good = Path.Combine(".minigit", "objects", hash[..2], hash[2..]);

// Other useful Path methods
string dir  = Path.GetDirectoryName("src/Core/BlobObject.cs"); // "src/Core"
string name = Path.GetFileName("src/Core/BlobObject.cs");      // "BlobObject.cs"
string ext  = Path.GetExtension("BlobObject.cs");              // ".cs"
string noExt = Path.GetFileNameWithoutExtension("BlobObject.cs"); // "BlobObject"
string abs  = Path.GetFullPath("./relative/path");             // absolute path
string temp = Path.GetTempPath();                              // system temp dir
```

---

## 3. Existence Checks and Directory Creation

```csharp
// Check if a file exists
if (File.Exists(path))
    // ...

// Check if a directory exists
if (Directory.Exists(dirPath))
    // ...

// Create a directory — safe to call even if it already exists
Directory.CreateDirectory(".minigit/objects/ab");

// Delete a file
File.Delete(path);

// Delete a directory and everything inside it
Directory.Delete(".minigit", recursive: true);
```

`Directory.CreateDirectory` will create all intermediate directories as well (like `mkdir -p` on Linux). It does not throw if the directory already exists.

---

## 4. Listing Files and Directories

```csharp
// Get all files in a directory (top level only)
string[] files = Directory.GetFiles("src");

// Get all files recursively
string[] allFiles = Directory.GetFiles("src", "*", SearchOption.AllDirectories);

// Filter by extension
string[] csFiles = Directory.GetFiles("src", "*.cs", SearchOption.AllDirectories);

// Get subdirectories
string[] dirs = Directory.GetDirectories(".minigit/objects");

// Lazy enumeration (preferred for large directories — does not load all paths at once)
foreach (var file in Directory.EnumerateFiles("src", "*.cs", SearchOption.AllDirectories))
{
    Console.WriteLine(file);
}
```

---

## 5. Working with Streams (for larger files)

For large files, `ReadAllBytes` loads everything into memory at once. Use a `FileStream` for streaming:

```csharp
using var stream = new FileStream("large-file.bin", FileMode.Open, FileAccess.Read);
var buffer = new byte[4096];
int bytesRead;
while ((bytesRead = stream.Read(buffer, 0, buffer.Length)) > 0)
{
    // process buffer[0..bytesRead]
}
```

In Mini-Git you will not need this for most files (objects are small). But it is good to know.

---

## 6. The Object Store Layout in Practice

When `minigit hash-object` stores a file with hash `e3b0c44298fc1c149afb...`:

```
.minigit/
└── objects/
    └── e3/
        └── b0c44298fc1c149afb4c8996fb92427ae41e4649b934ca495991b7852b855
```

Code to compute the path and write it:

```csharp
void StoreObject(string repoRoot, string hash, byte[] content)
{
    var dir  = Path.Combine(repoRoot, ".minigit", "objects", hash[..2]);
    var path = Path.Combine(dir, hash[2..]);

    Directory.CreateDirectory(dir);   // create the 2-char directory if needed
    File.WriteAllBytes(path, content);
}

byte[] RetrieveObject(string repoRoot, string hash)
{
    var path = Path.Combine(repoRoot, ".minigit", "objects", hash[..2], hash[2..]);
    if (!File.Exists(path))
        throw new FileNotFoundException($"Object not found: {hash}");
    return File.ReadAllBytes(path);
}
```

---

## Exercises

**Exercise 1 — Safe file writer**
Write `void WriteWithDirs(string path, string content)` that:
- Creates all parent directories if they do not exist
- Writes the text to the file
- Throws a descriptive exception if the path is invalid

**Exercise 2 — Object store on disk**
Implement `FileObjectStore.Store(byte[] content)`:
- Hash the content (see section 02-hashing)
- Build the path: `.minigit/objects/{hash[..2]}/{hash[2..]}`
- Create the directory and write the file
- Return the hash

Implement `FileObjectStore.Retrieve(string hash)`:
- Build the path from the hash
- Read and return the bytes
- Throw with a clear message if it does not exist

**Exercise 3 — Walk a directory**
Write `List<string>` `GetAllFilePaths(string directory)` that returns all file paths under a given directory, excluding any path that contains `.minigit`.
Use `Directory.EnumerateFiles` with `SearchOption.AllDirectories`.

**Exercise 4 — Temp directory test harness**
Write a helper class `TempRepo` that:
- Creates a unique temp directory in its constructor: `Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString())`
- Exposes the path as a property
- Deletes the directory recursively in `Dispose()`
Use it in a test to verify that `FileObjectStore.Store` then `Retrieve` round-trips correctly.
