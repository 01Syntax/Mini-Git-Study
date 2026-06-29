# Linked List

## Learning Resources

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 10.2 (Linked Lists). Short and clear.
- *Data Structures and Algorithms in C#* by Michael McMillan — Chapter 3 (Linked Lists). C#-specific, lots of examples.
- *Grokking Algorithms* by Aditya Bhargava — Chapter 2 covers arrays vs linked lists with visual illustrations. Very accessible.

**Articles**
- [LinkedList<T> — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.linkedlist-1)

---

## 1. The Problem That Linked Lists Solve

Arrays and `List<T>` store elements in contiguous memory. This makes index access fast (O(1)) but makes insertion and deletion slow (O(n) because you must shift elements).

A linked list trades index access for fast insertion/deletion anywhere: each element (node) knows only about its neighbours, and you update pointers rather than shifting data.

For Mini-Git, the important thing is not the `LinkedList<T>` class but the **concept**: each commit knows only about its parent. Following the parent chain is how you traverse history. This is a singly-linked list implemented manually using hashes rather than pointers.

---

## 2. Anatomy of a Node

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│  Node               │     │  Node               │     │  Node               │
│  value: "commit C"  │────▶│  value: "commit B"  │────▶│  value: "commit A"  │──▶ null
│  next: →            │     │  next: →            │     │  next: null         │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
  HEAD (newest)                                             root commit
```

Each node points to the **previous** commit (parent), not the next. History flows backward.

---

## 3. Implementing a Simple Linked List

```csharp
public class Node<T>
{
    public T Value { get; }
    public Node<T>? Next { get; set; }

    public Node(T value) { Value = value; }
}

public class LinkedList<T>
{
    private Node<T>? _head;

    // Add to the front (O(1))
    public void Prepend(T value)
    {
        var node = new Node<T>(value) { Next = _head };
        _head = node;
    }

    // Traverse from head to tail (O(n))
    public IEnumerable<T> Traverse()
    {
        var current = _head;
        while (current != null)
        {
            yield return current.Value;
            current = current.Next;
        }
    }

    // Find by value (O(n))
    public Node<T>? Find(Func<T, bool> predicate)
    {
        var current = _head;
        while (current != null)
        {
            if (predicate(current.Value)) return current;
            current = current.Next;
        }
        return null;
    }
}
```

---

## 4. The Commit Chain as a Linked List

In Mini-Git, commits are not stored with C# pointers. They are stored as files on disk. The "pointer" is a **hash** — the hash of the parent commit.

```csharp
public record CommitObject(
    string Hash,
    string TreeHash,
    string? ParentHash,   // ← the "pointer" backward in time (null for the root commit)
    string Author,
    DateTimeOffset Timestamp,
    string Message
);
```

To traverse from the latest commit to the root:

```csharp
IEnumerable<CommitObject> TraverseHistory(string startHash, IObjectStore store)
{
    var currentHash = startHash;
    while (currentHash != null)
    {
        var commit = Deserialize(store.Retrieve(currentHash));
        yield return commit;
        currentHash = commit.ParentHash;  // follow the pointer
    }
}
```

This is exactly `git log`:
```
commit d22a981 (HEAD → main)
    third commit

commit abc1234
    second commit

commit 000beef
    initial commit
```

---

## 5. Visual: Commit Chain

```
.minigit/refs/heads/main → "d22a981"

object d22a981 (CommitObject)
  treeHash:   "aaa111"
  parentHash: "abc1234"   ──────┐
  message:    "third commit"    │
                                ▼
object abc1234 (CommitObject)  
  treeHash:   "bbb222"
  parentHash: "000beef"   ──────┐
  message:    "second commit"   │
                                ▼
object 000beef (CommitObject)
  treeHash:   "ccc333"
  parentHash: null         ← root commit (no parent)
  message:    "initial commit"
```

---

## 6. Singly vs Doubly Linked

| | Singly | Doubly |
|---|---|---|
| Each node has | `next` only | `prev` and `next` |
| Traverse | Forward only | Both directions |
| Memory | Less | More |
| Complexity | Simpler | More complex |

Git commits form a **singly linked list** (pointing to parent only). You traverse backward in time. If you want commits in chronological order, traverse the whole list then reverse it.

For merge commits (covered in DAG section), a commit can have **two parents**. The list becomes a graph. But the traversal logic is the same — follow parent hashes.

---

## 7. Time Complexity

| Operation | Array/List | Linked List |
|---|---|---|
| Access by index | O(1) | O(n) |
| Insert at front | O(n) | O(1) |
| Insert at back | O(1) amortised | O(n) or O(1) with tail pointer |
| Insert in middle | O(n) | O(1) once you have the node |
| Delete | O(n) | O(1) once you have the node |
| Search | O(n) | O(n) |

For commit traversal, O(n) search is fine — you always start at HEAD and follow the chain.

---

## Exercises

**Exercise 1 — Build and traverse**
Implement `LinkedList<string>` with `Prepend` and `Traverse`.
Add commits `"A"`, `"B"`, `"C"` (A is oldest, C is newest).
Traverse from newest to oldest. Then reverse the result to print oldest-first.

**Exercise 2 — Commit chain simulation**
Create a `Dictionary<string, (string message, string? parentHash)>` representing a 5-commit chain:
```
root: ("initial", null)
aaa: ("add readme", "root")
bbb: ("fix bug", "aaa")
ccc: ("add feature", "bbb")
ddd: ("update docs", "ccc")
```
Write `PrintLog(string startHash, Dictionary<...> commits)` that traverses the chain and prints each commit.

**Exercise 3 — Count commits**
Using the chain from Exercise 2, write `int CountCommits(string hash)` that returns the number of commits reachable from `hash`.
