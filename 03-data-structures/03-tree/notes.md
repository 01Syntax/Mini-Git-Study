# Tree Data Structure

## Learning Resources

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 12 (Binary Search Trees) for tree fundamentals; Chapter B (appendix) for basic tree definitions.
- *Grokking Algorithms* — Chapter 7 (trees section). Visual, accessible.
- *Data Structures and Algorithm Analysis in C#* by Michael McMillan — Chapter 8 (Trees).

**Articles**
- [Tree traversal — Wikipedia](https://en.wikipedia.org/wiki/Tree_traversal) — pre-order, in-order, post-order explained with diagrams.
- [Merkle tree — Wikipedia](https://en.wikipedia.org/wiki/Merkle_tree) — the specific tree structure Git uses.
- [Git Tree Objects — Pro Git book](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects#_tree_objects)

---

## 1. What Is a Tree?

A tree is a hierarchical data structure where:
- There is exactly **one root** node (no parent)
- Every non-root node has exactly **one parent**
- Each node has zero or more **children**
- There are **no cycles**

```
                    root
                   /    \
               child1   child2
              /     \       \
           leaf1   leaf2   leaf3
```

Trees appear everywhere: file systems, XML/JSON documents, organisation charts, expression trees in compilers, and — in Mini-Git — directory snapshots.

---

## 2. Key Terminology

| Term | Meaning |
|------|---------|
| Root | The top node; has no parent |
| Leaf | A node with no children |
| Node | Any element in the tree |
| Edge | A connection between parent and child |
| Height | Longest path from root to a leaf |
| Depth | Distance from the root to a specific node |
| Subtree | A node and all its descendants |

---

## 3. Tree Traversal

Traversal means visiting every node exactly once. There are three classic orders:

```
        A
       / \
      B   C
     / \
    D   E
```

**Pre-order** (root → left → right): `A B D E C`
Use this when you need to process a node before processing its children (e.g. build a tree from a directory).

**In-order** (left → root → right): `D B E A C`
Use this for binary search trees. Not common for general trees.

**Post-order** (left → right → root): `D E B C A`
Use this when you need all children processed before the parent (e.g. compute a tree's hash from its children's hashes).

```csharp
// Pre-order traversal (recursive)
void PreOrder(TreeNode node)
{
    if (node is null) return;
    Process(node);              // visit root first
    foreach (var child in node.Children)
        PreOrder(child);        // then children
}

// Post-order traversal (recursive)
void PostOrder(TreeNode node)
{
    if (node is null) return;
    foreach (var child in node.Children)
        PostOrder(child);       // children first
    Process(node);              // then root
}
```

---

## 4. The Filesystem as a Tree

Your project directory IS a tree:

```
project/
├── README.md          ← leaf (file)
├── src/               ← internal node (directory)
│   ├── Program.cs     ← leaf
│   └── Core/          ← internal node
│       └── Blob.cs    ← leaf
└── tests/             ← internal node
    └── BlobTests.cs   ← leaf
```

Mapping to tree concepts:
- Directories = internal nodes (they have children)
- Files = leaves (they have no children)
- The project root = root node

---

## 5. TreeObject in Mini-Git

In Git/Mini-Git, a `TreeObject` captures a **snapshot of a directory** at a point in time.

```csharp
public record TreeEntry(
    string Mode,   // "100644" = file, "040000" = directory
    string Name,   // filename or directory name
    string Hash    // hash of a BlobObject (file) or another TreeObject (directory)
);

public record TreeObject(
    string Hash,
    IReadOnlyList<TreeEntry> Entries
) : GitObject(Hash);
```

Example tree for the `project/` directory above:
```
TreeObject {
    Hash: "root-tree-hash",
    Entries: [
        TreeEntry { Mode: "100644", Name: "README.md", Hash: "blob-hash-readme" },
        TreeEntry { Mode: "040000", Name: "src",       Hash: "tree-hash-src" },
        TreeEntry { Mode: "040000", Name: "tests",     Hash: "tree-hash-tests" }
    ]
}
```

The `src` entry's hash points to another TreeObject:
```
TreeObject {
    Hash: "tree-hash-src",
    Entries: [
        TreeEntry { Mode: "100644", Name: "Program.cs", Hash: "blob-hash-program" },
        TreeEntry { Mode: "040000", Name: "Core",       Hash: "tree-hash-core" }
    ]
}
```

---

## 6. Merkle Trees — The Key Concept

A **Merkle tree** is a tree where each node's hash depends on the hashes of all its children. Git uses Merkle trees for its object model.

```
           commit (hash: d22a981)
               |
           root tree (hash: aaa)
          /              \
    blob: README.md    subtree: src (hash: bbb)
    (hash: blob-a)     /         \
                  blob: main.cs  blob: util.cs
                  (hash: blob-b) (hash: blob-c)
```

**How the hash propagates upward:**
1. Hash `main.cs` content → `blob-b`
2. Hash `util.cs` content → `blob-c`
3. Hash the `src` tree (which includes `blob-b` and `blob-c`) → `bbb`
4. Hash the `root` tree (which includes `blob-a` and `bbb`) → `aaa`
5. Hash the commit (which includes `aaa`) → `d22a981`

**The consequence:** if you change one byte in `util.cs`, `blob-c` changes, which changes `bbb`, which changes `aaa`, which changes `d22a981`. **Every ancestor hash changes.** History is tamper-evident by construction.

---

## 7. Building a TreeObject Recursively

```csharp
string BuildTree(string dirPath, IObjectStore store)
{
    var entries = new List<TreeEntry>();

    foreach (var filePath in Directory.GetFiles(dirPath).OrderBy(f => f))
    {
        var name = Path.GetFileName(filePath);
        var content = File.ReadAllBytes(filePath);
        var hash = store.Store(content);    // store blob, get hash
        entries.Add(new TreeEntry("100644", name, hash));
    }

    foreach (var subDir in Directory.GetDirectories(dirPath).OrderBy(d => d))
    {
        if (Path.GetFileName(subDir) == ".minigit") continue; // skip the repo itself
        var name = Path.GetFileName(subDir);
        var hash = BuildTree(subDir, store);  // ← recursive call
        entries.Add(new TreeEntry("040000", name, hash));
    }

    // Serialize and store the tree
    var treeHash = StoreTree(entries, store);
    return treeHash;
}
```

---

## Exercises

**Exercise 1 — Model a file system**
Create `TreeNode` (has Name, IsFile, List<TreeNode> Children).
Build this tree in code:
```
root/
├── a.txt
├── b.txt
└── sub/
    ├── c.txt
    └── d.txt
```
Write `PrintTree(TreeNode, depth)` that prints it with indentation.

**Exercise 2 — Count nodes**
Write `int CountFiles(TreeNode root)` that returns the number of files (leaves) using recursion.
Write `int MaxDepth(TreeNode root)` that returns the height of the tree.

**Exercise 3 — Merkle hash**
Extend Exercise 1:
- Assign each file a "content" (any string)
- Hash each file: `fileHash = SHA256(name + content)`
- Hash each directory: `dirHash = SHA256(name + sorted children hashes concatenated)`
- Print the root hash

Then change `a.txt`'s content and verify the root hash changes completely.

**Exercise 4 — Recursive directory walk**
Write `BuildTree(string path, IObjectStore store)` that:
- Walks the real filesystem from `path`
- Creates BlobObjects for files, TreeObjects for directories
- Returns the root tree hash
Run it on a small test directory.
