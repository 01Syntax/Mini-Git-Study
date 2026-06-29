# BFS and DFS (Breadth-First / Depth-First Search)

## Learning Resources

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 22.2 (BFS) and 22.3 (DFS). The definitive reference with full proofs and complexity analysis.
- *Grokking Algorithms* — Chapter 6 (Breadth-First Search). Excellent diagrams, very approachable.
- *Algorithm Design* by Kleinberg & Tardos — Chapter 3 (Graphs). Great intuition for when to use each.

**Articles / Videos**
- [BFS vs DFS — Wikipedia comparison](https://en.wikipedia.org/wiki/Graph_traversal)
- [Breadth-First Search — Wikipedia](https://en.wikipedia.org/wiki/Breadth-first_search)
- [Depth-First Search — Wikipedia](https://en.wikipedia.org/wiki/Depth-first_search)
- [BFS/DFS visual — VisuAlgo](https://visualgo.net/en/dfsbfs) — interactive visualiser, highly recommended

---

## 1. The Problem: Visiting All Nodes in a Graph

Given a graph and a starting node, visit every reachable node exactly once.
Two systematic strategies: explore wide first (BFS) or explore deep first (DFS).

---

## 2. Breadth-First Search (BFS)

**Strategy:** Visit all neighbours of the current node before going deeper.
Think of it as ripples spreading out from a stone thrown in a pond — all nodes at distance 1 before distance 2, etc.

**Data structure:** `Queue<T>` (FIFO — first in, first out)

```
Graph:
    A
   / \
  B   C
 / \   \
D   E   F

BFS from A:
  Level 0: A
  Level 1: B, C
  Level 2: D, E, F

Visit order: A → B → C → D → E → F
```

### BFS Implementation

```csharp
IEnumerable<string> Bfs(string start, Dictionary<string, List<string>> graph)
{
    var visited = new HashSet<string>();
    var queue   = new Queue<string>();

    queue.Enqueue(start);

    while (queue.Count > 0)
    {
        var node = queue.Dequeue();

        if (!visited.Add(node))  // Add returns false if already present
            continue;

        yield return node;       // visit the node

        foreach (var neighbour in graph.GetValueOrDefault(node, new()))
            if (!visited.Contains(neighbour))
                queue.Enqueue(neighbour);
    }
}
```

**Key property of BFS:** Nodes are visited in order of their distance from the start node.
The first time you visit a node, you have found the **shortest path** (in edge count) to it.

---

## 3. Depth-First Search (DFS)

**Strategy:** Follow one path as deep as possible before backtracking.
Think of it as exploring a maze by always turning left — going as deep as possible before backing up.

**Data structure:** `Stack<T>` (LIFO — last in, first out), or recursion (which uses the call stack)

```
Graph:
    A
   / \
  B   C
 / \   \
D   E   F

DFS from A (left before right):
  Visit A → go to B → go to D (no more children) → backtrack to B → go to E → backtrack
  → backtrack to A → go to C → go to F

Visit order: A → B → D → E → C → F
```

### DFS Implementation (iterative with Stack)

```csharp
IEnumerable<string> Dfs(string start, Dictionary<string, List<string>> graph)
{
    var visited = new HashSet<string>();
    var stack   = new Stack<string>();

    stack.Push(start);

    while (stack.Count > 0)
    {
        var node = stack.Pop();

        if (!visited.Add(node))
            continue;

        yield return node;

        // Push in reverse so left children are processed first
        var neighbours = graph.GetValueOrDefault(node, new());
        foreach (var neighbour in neighbours.AsEnumerable().Reverse())
            if (!visited.Contains(neighbour))
                stack.Push(neighbour);
    }
}
```

### DFS Implementation (recursive — often cleaner)

```csharp
void DfsRecursive(string node, Dictionary<string, List<string>> graph, HashSet<string> visited)
{
    if (!visited.Add(node)) return;

    Console.WriteLine(node);  // visit

    foreach (var neighbour in graph.GetValueOrDefault(node, new()))
        DfsRecursive(neighbour, graph, visited);
}

// Call with:
var visited = new HashSet<string>();
DfsRecursive("A", graph, visited);
```

---

## 4. BFS vs DFS: Side by Side

| | BFS | DFS |
|---|---|---|
| Data structure | Queue (FIFO) | Stack (LIFO) / recursion |
| Visit order | Level by level | As deep as possible first |
| Finds shortest path? | **Yes** (unweighted) | No |
| Memory use | Can be large (holds all nodes of current level) | Smaller (holds one path at a time) |
| Implementation | Usually iterative | Easily recursive |
| Best for | Shortest path, nearest neighbour, level-by-level processing | Topological sort, cycle detection, backtracking |

---

## 5. Complexity

Both algorithms visit each node and each edge exactly once:
- **Time:** O(V + E) where V = number of vertices (nodes), E = number of edges
- **Space:** O(V) for the queue/stack and visited set

---

## 6. Mini-Git: `log` uses DFS

`git log` (and `minigit log`) follows the first parent of each commit, going as far back as possible before considering other parents (merge commits). This is a DFS on the commit DAG.

```csharp
IEnumerable<CommitObject> Log(string headHash, IObjectStore store)
{
    var visited = new HashSet<string>();
    var stack   = new Stack<string>();

    stack.Push(headHash);

    while (stack.Count > 0)
    {
        var hash   = stack.Pop();
        if (!visited.Add(hash)) continue;

        var commit = DeserializeCommit(store.Retrieve(hash));
        yield return commit;

        // Push parents — first parent is pushed last so it's processed first (DFS characteristic)
        foreach (var parent in commit.ParentHashes.Reverse())
            stack.Push(parent);
    }
}
```

For a linear history (`root ← A ← B ← C`), starting from C:
```
Push C → Pop C (visit C) → Push B
Push B → Pop B (visit B) → Push A
Push A → Pop A (visit A) → Push root
Push root → Pop root (visit root) → done
```
Output: C, B, A, root — newest first, exactly like `git log`.

---

## 7. Mini-Git: `merge-base` uses BFS

Finding the lowest common ancestor (covered in detail in `02-lca/notes.md`) uses BFS from both branch tips simultaneously. BFS is correct here because we want the **nearest** common ancestor.

```csharp
string? FindMergeBase(string hashA, string hashB, IObjectStore store)
{
    var visitedA = new HashSet<string>();
    var visitedB = new HashSet<string>();
    var queueA   = new Queue<string>();
    var queueB   = new Queue<string>();

    queueA.Enqueue(hashA);
    queueB.Enqueue(hashB);

    while (queueA.Count > 0 || queueB.Count > 0)
    {
        if (queueA.TryDequeue(out var a))
        {
            if (visitedA.Add(a))
            {
                if (visitedB.Contains(a)) return a;  // first common ancestor
                var commit = DeserializeCommit(store.Retrieve(a));
                foreach (var p in commit.ParentHashes) queueA.Enqueue(p);
            }
        }

        if (queueB.TryDequeue(out var b))
        {
            if (visitedB.Add(b))
            {
                if (visitedA.Contains(b)) return b;
                var commit = DeserializeCommit(store.Retrieve(b));
                foreach (var p in commit.ParentHashes) queueB.Enqueue(p);
            }
        }
    }

    return null;  // no common ancestor (shouldn't happen in a valid repo)
}
```

---

## Exercises

**Exercise 1 — Build a graph and run both**
Create this graph as `Dictionary<string, List<string>>`:
```
A → [B, C]
B → [D, E]
C → [F]
D → []
E → []
F → []
```
Run BFS from A and print the visit order.
Run DFS from A and print the visit order.
Draw the graph and trace through both algorithms by hand.

**Exercise 2 — Shortest path with BFS**
Extend your BFS to also track how far each node is from the start (its level/distance).
Print `node: distance` for each visited node.

**Exercise 3 — Cycle detection with DFS**
Modify DFS to use three colours: white (unvisited), grey (in progress), black (done).
If you ever see a grey node, there is a cycle.
Test it on a graph with a cycle and one without.

**Exercise 4 — git log simulation**
Create a commit DAG as `Dictionary<string, string?>` (hash → parentHash, null for root):
```
"A" → null
"B" → "A"
"C" → "B"
"D" → "B"
"E" (merge) → parents: ["C", "D"]
```
Use DFS from "E" to print commits in the same order as `git log`.

**Exercise 5 — Connected components**
Given a graph (not necessarily connected), find all groups of connected nodes.
Use DFS or BFS: every time you start a new traversal, you have found a new component.
