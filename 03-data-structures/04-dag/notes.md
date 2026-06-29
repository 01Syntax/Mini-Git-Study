# Directed Acyclic Graph (DAG)

## Learning Resources

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 22 (Elementary Graph Algorithms). Sections 22.1–22.3 cover graph representations and DFS. Section 22.4 covers topological sort (a DAG-specific algorithm).
- *Grokking Algorithms* — Chapter 7 (Dijkstra's algorithm introduces graphs accessibly).
- *Algorithm Design* by Kleinberg & Tardos — Chapter 3.6 covers DAGs and their properties.

**Articles**
- [Directed acyclic graph — Wikipedia](https://en.wikipedia.org/wiki/Directed_acyclic_graph) — comprehensive overview with examples.
- [Git is a DAG — Stack Overflow answer](https://stackoverflow.com/questions/26395521/dag-vs-tree-using-git) — explains why git uses a DAG, not a tree.

---

## 1. Graph Fundamentals

Before understanding a DAG, understand what a graph is.

A **graph** is a set of:
- **Nodes** (also called vertices) — the entities
- **Edges** (also called arcs) — connections between nodes

```
A --- B --- C
|         |
D --------+
```

Unlike a tree, a graph has no restriction on the structure:
- A node can connect to any other node
- A node can have any number of connections (not just one parent)
- There can be cycles (paths that loop back to the start)

---

## 2. Directed Graphs

In a **directed** graph, edges have a direction. `A → B` means "A points to B" but NOT "B points to A".

```
A → B → C
    ↑
    D
```

D points to B, but B does not point to D.

In Git, commit edges point **from child to parent** (backward in time):
- Newest commit → older parent commit
- The arrows model "this commit was built on top of that one"

---

## 3. Acyclic: No Cycles

**Acyclic** means you can never follow edges and return to your starting node.

```
Cyclic (NOT a DAG):      Acyclic (valid DAG):
A → B → C → A           A → B → C
              ↑                   ↓
              cycle!              D
```

Why Git commits cannot form cycles:
- A commit's hash depends on its parent's hash
- To create a cycle, commit B would need to reference commit A, and A would need to reference B
- But A's hash is computed before B exists — so B cannot be part of A's hash
- **SHA-256 hashing makes cycles physically impossible**

---

## 4. From Linked List to DAG

Linear history (linked list): each commit has exactly one parent.

```
root ← A ← B ← C   (main branch, HEAD at C)
```

After branching:

```
root ← A ← B ← C      (main)
                ↑
                D ← E  (feature)
```

After merging feature into main:

```
root ← A ← B ← C ← F  (main, F is the merge commit)
                ↑   ↑
                D ← E  (feature)
```

Merge commit F has **two parents**: C and E. This is where the linked list becomes a DAG.

In code:
```csharp
public record CommitObject(
    string Hash,
    string TreeHash,
    IReadOnlyList<string> ParentHashes,  // normally 1, merge commits have 2
    string Author,
    DateTimeOffset Timestamp,
    string Message
);

// Simple commit: ParentHashes = ["abc123"]
// Root commit:   ParentHashes = []
// Merge commit:  ParentHashes = ["abc123", "def456"]
```

---

## 5. Representing a Graph in Code

Two common representations:

### Adjacency List (most common for sparse graphs)
```csharp
// commit hash → list of parent hashes
var graph = new Dictionary<string, List<string>>
{
    ["F"] = new() { "C", "E" },    // merge commit
    ["E"] = new() { "D" },
    ["D"] = new() { "B" },
    ["C"] = new() { "B" },
    ["B"] = new() { "A" },
    ["A"] = new() { "root" },
    ["root"] = new() { },
};
```

### Adjacency Matrix (better for dense graphs)
A 2D boolean array where `matrix[i][j] = true` means there is an edge from node i to node j.
Not suitable for git commits (too many nodes, sparse connections).

---

## 6. Topological Sort

A **topological sort** of a DAG orders nodes so that all edges point forward: if there is an edge `A → B`, then A comes before B in the order.

For git commits: topological order puts the root commit first and the latest commit last (chronological order).

```
Topological order: root, A, B, C, D, E, F
(every commit appears before commits that depend on it)
```

This is what `git log --topo-order` does. Implementation uses DFS with a post-order collection.

---

## 7. Reachability

A node Y is **reachable** from node X if there is a directed path `X → ... → Y`.

For git: commit C is reachable from HEAD if it is in HEAD's history (you can follow parent links to get there).

```csharp
HashSet<string> GetAllAncestors(string startHash, IObjectStore store)
{
    var visited = new HashSet<string>();
    var queue = new Queue<string>();
    queue.Enqueue(startHash);

    while (queue.Count > 0)
    {
        var hash = queue.Dequeue();
        if (!visited.Add(hash)) continue;

        var commit = Deserialize(store.Retrieve(hash));
        foreach (var parent in commit.ParentHashes)
            queue.Enqueue(parent);
    }
    return visited;
}
```

---

## 8. Why "Acyclic" Matters for Algorithms

Many algorithms that work on general graphs become simpler and more efficient on DAGs:

- **DFS** on a DAG has no risk of infinite loops (no cycles to revisit)
- **Shortest path** on a DAG can be solved in O(V+E) rather than O(V log V + E) (no need for Dijkstra)
- **Topological sort** only works on DAGs — cycles make it impossible
- **Merge-base** (LCA) is straightforward because the ancestor relationship is well-defined

---

## Exercises

**Exercise 1 — Draw the commit DAG**
On paper (or in ASCII art), draw the DAG for:
1. Three commits on main (root → A → B)
2. Branch from A: A → C → D
3. Merge B and D: merge commit E (parents: B, D)
Label each node with a short hash.

**Exercise 2 — Build the DAG in code**
Using `Dictionary<string, List<string>>` (hash → parent hashes), implement the DAG from Exercise 1.
Write `bool IsAncestor(string ancestor, string descendant, Dictionary<...> graph)` using BFS.

**Exercise 3 — All ancestors**
Write `HashSet<string> GetAncestors(string hash, Dictionary<string, List<string>> graph)`.
Test it: ancestors of the merge commit E should include all other commits.

**Exercise 4 — Topological sort**
Implement topological sort on your commit DAG using DFS.
The output should list commits from root to latest (root first).
This is the order `git log --reverse` would show them.

**Exercise 5 — Cycle detection**
Even though real commit DAGs cannot have cycles, implement `bool HasCycle(Dictionary<string, List<string>> graph)` using DFS with a "currently visiting" set (grey nodes) vs a "done" set (black nodes).
Test it with a graph that has a cycle and one that does not.
