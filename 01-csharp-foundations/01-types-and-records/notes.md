# Types & Records

## Learning Resources

**Official Docs**
- [C# Records — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record)
- [Classes vs Structs — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/choosing-between-class-and-struct)
- [Value types — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-types)

**Books**
- *C# in a Nutshell* by Joseph & Ben Albahari — Chapter 3 (Types), Chapter 4 (Value vs Reference types). The most thorough reference for C# type system.
- *Pro C# 10 and .NET 6* by Andrew Troelsen — Chapter 4 covers classes, structs, and records with worked examples.

**Articles**
- [Records in C# — detailed breakdown](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/records)

---

## 1. Classes

A **class** is a reference type. When you assign one class variable to another, both variables point to the same object in memory.

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

var a = new Person { Name = "Alice", Age = 30 };
var b = a;            // b points to the SAME object as a
b.Name = "Bob";
Console.WriteLine(a.Name); // "Bob" — a and b share the same object
```

**Equality:** By default, `==` on classes compares *references* (are these the same object?), not values.

```csharp
var x = new Person { Name = "Alice" };
var y = new Person { Name = "Alice" };
Console.WriteLine(x == y); // false — different objects, even though content is identical
```

---

## 2. Structs

A **struct** is a value type. Assignment copies the entire value.

```csharp
public struct Point
{
    public int X;
    public int Y;
}

var p1 = new Point { X = 1, Y = 2 };
var p2 = p1;    // full copy
p2.X = 99;
Console.WriteLine(p1.X); // 1 — p1 is unaffected
```

**When to use a struct:**
- Small, simple data (coordinates, RGB colour, a 2D vector)
- You want copy semantics — passing to a method should not allow the method to mutate the original
- You create millions of them (avoids heap allocation)

**When NOT to use a struct:**
- Large data (copying is expensive)
- You need inheritance
- Mutability is required and you pass it around frequently

---

## 3. Records

A **record** is a reference type with built-in value equality and easy immutability. Introduced in C# 9.

```csharp
public record BlobObject(string Hash, byte[] Content);

var b1 = new BlobObject("abc123", new byte[] { 1, 2, 3 });
var b2 = new BlobObject("abc123", new byte[] { 1, 2, 3 });

Console.WriteLine(b1 == b2); // true — records compare by value, not reference
```

Records give you these for free:
- `==` / `.Equals()` based on property values
- `ToString()` that prints all properties
- `with` expressions for non-destructive updates

```csharp
var original = new BlobObject("abc123", content);
var copy = original with { Hash = "def456" }; // new record, only Hash changed
```

### Immutability with `init` setters

```csharp
public record CommitObject
{
    public string Hash    { get; init; }   // can be set only during construction
    public string Message { get; init; }
    public string? ParentHash { get; init; }
}

var c = new CommitObject { Hash = "abc", Message = "init", ParentHash = null };
// c.Hash = "xyz"; // COMPILE ERROR — init-only property
```

### `readonly` on fields

On a class or struct, `readonly` means the field can only be set in the constructor:

```csharp
public class Hasher
{
    private readonly SHA256 _sha = SHA256.Create(); // set once in field initializer
}
```

---

## 4. Value Equality vs Reference Equality

| | Class (default) | Struct | Record |
|---|---|---|---|
| `==` compares | Reference (same object?) | Value | Value |
| Mutable? | Yes | Yes | No (with `init`) |
| Heap allocated? | Yes | No (stack) | Yes |

```csharp
// Reference equality check (works on any reference type)
object.ReferenceEquals(a, b);

// Value equality (override .Equals or use records)
a.Equals(b);
a == b; // on records this is value equality automatically
```

---

## 5. Why Records for Git Objects

Git objects are immutable: once a blob is written, it never changes. Its hash IS its identity. Two blobs with the same content are the same object — this is exactly what value equality gives you.

Using records:
- Enforces immutability at compile time (`init` setters)
- Makes equality checks correct by default (`blob1 == blob2` compares content)
- Makes debugging easy (`Console.WriteLine(blob)` prints all fields)

```csharp
// This is what the Mini-Git object model looks like
public abstract record GitObject(string Hash);

public record BlobObject(string Hash, byte[] Content) : GitObject(Hash);

public record TreeEntry(string Mode, string Name, string Hash);

public record TreeObject(string Hash, IReadOnlyList<TreeEntry> Entries) : GitObject(Hash);

public record CommitObject(
    string Hash,
    string TreeHash,
    string? ParentHash,
    string Author,
    DateTimeOffset Timestamp,
    string Message
) : GitObject(Hash);
```

---

## Exercises

**Exercise 1 — Equality experiment**
Create a `class Person` with Name and Age. Create two instances with the same values.
Then create a `record PersonRecord` with the same properties. Compare both with `==`.
Print results and explain why they differ.

**Exercise 2 — Immutability**
Create a `record` with three properties. Try to mutate one after construction (see the compiler error).
Then use a `with` expression to create a modified copy.

**Exercise 3 — Mini-Git warm-up**
Define `BlobObject`, `TreeEntry`, and `CommitObject` as records.
Create an instance of each. Print them. Create two `BlobObject`s with the same hash — verify they are `==`.
