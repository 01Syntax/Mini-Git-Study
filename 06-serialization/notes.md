# Serialization

## Learning Resources

**Official Docs**
- [System.Text.Json overview — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/overview)
- [JsonSerializer — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.jsonserializer)
- [JsonPropertyName attribute — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.serialization.jsonpropertynameattribute)
- [How to serialize and deserialize JSON — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/how-to)

**Books**
- *C# in a Nutshell* — Chapter 17 (Serialization). Covers both JSON and binary serialization, with a good comparison of the trade-offs.

---

## 1. What Is Serialization?

**Serialization** is converting an in-memory object to a format that can be stored on disk or sent over a network.

**Deserialization** is the reverse: reading that stored format and reconstructing the object.

```
In-memory object                   On-disk (or network) representation
────────────────────               ────────────────────────────────────
Dictionary<string, IndexEntry>  →  JSON text in .minigit/index
                               ←   (deserialized back on next run)
```

Without serialization, your staging index disappears every time the process exits.

---

## 2. Why JSON for the Index?

Real Git uses a custom binary format for its index. We use JSON because:
- Human-readable — you can open `.minigit/index` in a text editor and understand it
- Easy to debug — when something goes wrong, you can inspect the state directly
- Standard library support — `System.Text.Json` is built into .NET, no extra packages
- Teaches the same concept — the data structure is what matters, not the encoding

---

## 3. System.Text.Json Basics

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
```

### Serialize (object → JSON string)

```csharp
var entry = new IndexEntry("abc123", "100644", 1024);
string json = JsonSerializer.Serialize(entry);
// → {"Hash":"abc123","Mode":"100644","Size":1024}

// With indented (pretty-printed) output
var options = new JsonSerializerOptions { WriteIndented = true };
string pretty = JsonSerializer.Serialize(entry, options);
// →
// {
//   "Hash": "abc123",
//   "Mode": "100644",
//   "Size": 1024
// }
```

### Deserialize (JSON string → object)

```csharp
var restored = JsonSerializer.Deserialize<IndexEntry>(json);
Console.WriteLine(restored.Hash);  // "abc123"
```

### Serialize a Dictionary

```csharp
var index = new Dictionary<string, IndexEntry>
{
    ["README.md"]  = new IndexEntry("abc123", "100644", 512),
    ["Program.cs"] = new IndexEntry("def456", "100644", 2048),
};

string json = JsonSerializer.Serialize(index, options);
// →
// {
//   "README.md": { "Hash": "abc123", "Mode": "100644", "Size": 512 },
//   "Program.cs": { "Hash": "def456", "Mode": "100644", "Size": 2048 }
// }

var restored = JsonSerializer.Deserialize<Dictionary<string, IndexEntry>>(json);
```

---

## 4. Controlling Field Names with `[JsonPropertyName]`

By default, JSON property names match C# property names exactly. Use `[JsonPropertyName]` to override:

```csharp
public record IndexEntry(
    [property: JsonPropertyName("hash")]  string Hash,
    [property: JsonPropertyName("mode")]  string Mode,
    [property: JsonPropertyName("size")]  long Size,
    [property: JsonPropertyName("mtime")] DateTimeOffset ModifiedAt
);
```

Now the JSON uses lowercase keys:
```json
{
  "hash": "abc123",
  "mode": "100644",
  "size": 1024,
  "mtime": "2024-01-15T10:30:00+00:00"
}
```

This is important if you want your index format to be consistent regardless of C# naming conventions.

---

## 5. JsonSerializerOptions

```csharp
var options = new JsonSerializerOptions
{
    WriteIndented = true,                    // pretty-print for human readability
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,  // auto-convert to camelCase
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,  // skip null properties
    Converters = { new JsonStringEnumConverter() },  // serialize enums as strings
};
```

For Mini-Git, `WriteIndented = true` is the most useful option — the index file becomes human-readable.

---

## 6. Handling Polymorphism

If you need to serialize/deserialize a base type where the actual type could be a derived record (e.g. `GitObject` which could be `BlobObject`, `TreeObject`, or `CommitObject`), you need a type discriminator:

```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "type")]
[JsonDerivedType(typeof(BlobObject),   "blob")]
[JsonDerivedType(typeof(TreeObject),   "tree")]
[JsonDerivedType(typeof(CommitObject), "commit")]
public abstract record GitObject(string Hash);
```

This adds a `"type": "blob"` field to the JSON so deserialization knows which derived type to create.

For Mini-Git, a simpler approach is to use separate serialization methods for each object type rather than polymorphic serialization.

---

## 7. The FileIndex Class

```csharp
public class FileIndex : IIndex
{
    private readonly string _indexPath;
    private Dictionary<string, IndexEntry> _entries = new();

    public FileIndex(string repoRoot)
    {
        _indexPath = Path.Combine(repoRoot, ".minigit", "index");
        Load();
    }

    public void Load()
    {
        if (!File.Exists(_indexPath))
        {
            _entries = new Dictionary<string, IndexEntry>();
            return;
        }
        var json = File.ReadAllText(_indexPath);
        _entries = JsonSerializer.Deserialize<Dictionary<string, IndexEntry>>(json)
                   ?? new Dictionary<string, IndexEntry>();
    }

    public void Save()
    {
        var options = new JsonSerializerOptions { WriteIndented = true };
        var json = JsonSerializer.Serialize(_entries, options);
        File.WriteAllText(_indexPath, json);
    }

    public void Add(string filename, IndexEntry entry)
    {
        _entries[filename] = entry;
        Save();   // persist immediately
    }

    public void Remove(string filename)
    {
        _entries.Remove(filename);
        Save();
    }

    public IReadOnlyDictionary<string, IndexEntry> GetEntries() => _entries;
}
```

---

## 8. Binary Serialization (Stretch)

If you want to understand the real Git index format or implement pack files, you need binary serialization.

```csharp
using System.IO;

// Write structured binary data
using var writer = new BinaryWriter(File.OpenWrite("data.bin"));
writer.Write(42);             // int32 (4 bytes)
writer.Write(3.14f);          // float32 (4 bytes)
writer.Write("hello");        // length-prefixed string

// Read it back
using var reader = new BinaryReader(File.OpenRead("data.bin"));
int i   = reader.ReadInt32();
float f = reader.ReadSingle();
string s = reader.ReadString();
```

**Big-endian vs little-endian:**
- Most x86/x64 systems are little-endian (least significant byte first)
- Network protocols and some file formats use big-endian (most significant byte first)
- Git's binary index uses big-endian integers

```csharp
// Read a big-endian uint32
byte[] bytes = reader.ReadBytes(4);
if (BitConverter.IsLittleEndian) Array.Reverse(bytes);
uint value = BitConverter.ToUInt32(bytes, 0);
```

---

## Exercises

**Exercise 1 — Round-trip test**
Create `IndexEntry` with all fields. Serialize to JSON. Deserialize back. Assert all fields are equal.
Use `JsonPropertyName` so the JSON uses `snake_case` keys.

**Exercise 2 — Save and load the index**
Implement `FileIndex`:
- `Load()` reads the JSON file (empty Dictionary if file doesn't exist)
- `Save()` writes with `WriteIndented = true`
- `Add(filename, entry)` adds to the dictionary and saves
- `Remove(filename)` removes and saves
Write a test using a temp directory.

**Exercise 3 — Inspect the output**
Run Exercise 2, then open the `.minigit/index` file in a text editor.
Verify it is readable and correct. Try editing a field manually, then loading it again.

**Exercise 4 — Serialization with DateTimeOffset**
Add `DateTimeOffset ModifiedAt` to `IndexEntry`.
Serialize and deserialize. Verify the timestamp round-trips correctly.
(Tip: `DateTimeOffset` serializes to ISO 8601 string by default — `"2024-01-15T10:30:00+02:00"`.)

**Exercise 5 — Error handling**
What happens if the JSON file is corrupted (e.g. half-written due to a crash)?
Modify `Load()` to catch `JsonException` and fall back to an empty index with a warning message.
