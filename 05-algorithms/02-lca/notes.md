# Lowest Common Ancestor (LCA)

## Learning Resources

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 21 covers LCA in the context of disjoint sets (offline algorithms). For the online BFS approach used in Mini-Git, the graph traversal chapters (22) are more relevant.
- *Algorithm Design* by Kleinberg & Tardos — Chapter 3 covers graph algorithms that inform LCA strategies.

**Articles**
- [Lowest common ancestor — Wikipedia](https://en.wikipedia.org/wiki/Lowest_common_ancestor)
- [Finding the merge base — Git documentation internals](https://git-scm.com/docs/git-merge-base)

---

## 1. What Is the Lowest Common Ancestor?

Given a directed graph (or tree) and two nodes A and B, their **Lowest Common Ancestor (LCA)** is the node that:
1. Is an ancestor of both A and B
2. Is as deep as possible (i.e., the "lowest" in the tree — closest to A and B, farthest from the root)

In a tree this is unambiguous. In a DAG there can be multiple valid answers, and Git's merge-base uses a more nuanced algorithm (we'll use the basic version).

---

## 2. In Git Terms: The Merge Base

When you run `git merge feature` on `main`, Git needs to know **where the two branches diverged**. That divergence point is the merge base — the LCA of the two branch tip commits.

```
root ← A ← B ← C   (main, HEAD at C)
              ↑
              D ← E  (feature, HEAD at E)
```

LCA of C and E:
- C's ancestors: C, B, A, root
- E's ancestors: E, D, B, A, root
- Common ancestors: B, A, root
- **LCA (nearest common ancestor): B**  ← merge base

So Git merges `main` (C) and `feature` (E) using B as the common baseline.

---

## 3. Why It Matters for Merge

A three-way merge compares three versions of each file:
1. **Base** — the file at the merge base (commit B's tree)
2. **Ours** — the file at main tip (commit C's tree)
3. **Theirs** — the file at feature tip (commit E's tree)

```
Base:   "line 1\nline 2\n"
Ours:   "line 1\nline 2\nline 3\n"   (we added line 3)
Theirs: "line 1\nmodified line 2\n"  (they modified line 2)

Three-way result:
  "line 1\nmodified line 2\nline 3\n"  ← both changes applied

If both sides changed line 2 differently → CONFLICT
```

Without the merge base you would not know which changes are new vs which were already in the common history.

---

## 4. Finding the LCA with BFS

**Basic approach (easiest to understand):**

1. Collect all ancestors of A using BFS → set `ancestorsOfA`
2. BFS from B, stop the moment you find a node in `ancestorsOfA`
3. The first such node is the LCA

```csharp
string? FindLca(string hashA, string hashB, Dictionary<string, List<string>> parents)
{
    // Step 1: collect all ancestors of A
    var ancestorsOfA = new HashSet<string>();
    var queue = new Queue<string>();
    queue.Enqueue(hashA);
    while (queue.Count > 0)
    {
        var node = queue.Dequeue();
        if (!ancestorsOfA.Add(node)) continue;
        foreach (var p in parents.GetValueOrDefault(node, new()))
            queue.Enqueue(p);
    }

    // Step 2: BFS from B, stop when we hit an ancestor of A
    var visited = new HashSet<string>();
    queue = new Queue<string>();
    queue.Enqueue(hashB);
    while (queue.Count > 0)
    {
        var node = queue.Dequeue();
        if (!visited.Add(node)) continue;
        if (ancestorsOfA.Contains(node)) return node;  // ← found the LCA
        foreach (var p in parents.GetValueOrDefault(node, new()))
            queue.Enqueue(p);
    }

    return null;  // no common ancestor
}
```

**Why BFS (not DFS) for step 2?**
BFS explores by distance from the start. The first common ancestor found via BFS from B is the one **closest to B** — i.e., the deepest common ancestor, which is what we want.

---

## 5. Interleaved BFS (More Efficient)

Instead of two separate passes, interleave BFS from both ends simultaneously:

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
        // Process one step from A's side
        if (queueA.TryDequeue(out var a) && visitedA.Add(a))
        {
            if (visitedB.Contains(a)) return a;  // a is an ancestor of both
            EnqueueParents(a, queueA, store);
        }

        // Process one step from B's side
        if (queueB.TryDequeue(out var b) && visitedB.Add(b))
        {
            if (visitedA.Contains(b)) return b;  // b is an ancestor of both
            EnqueueParents(b, queueB, store);
        }
    }

    return null;
}

void EnqueueParents(string hash, Queue<string> queue, IObjectStore store)
{
    var commit = DeserializeCommit(store.Retrieve(hash));
    foreach (var parent in commit.ParentHashes)
        queue.Enqueue(parent);
}
```

This terminates earlier because you stop as soon as any overlap is detected.

---

## 6. Tracing Through an Example

```
root ← A ← B ← C   (main, HEAD at C)
              ↑
              D ← E  (feature, HEAD at E)
```

Parents:
```
C → [B]
E → [D]
B → [A]
D → [B]
A → [root]
root → []
```

Finding LCA of C and E:

**Iteration 1:**
- queueA dequeues C. visitedA = {C}. Not in visitedB. Enqueue B.
- queueB dequeues E. visitedB = {E}. Not in visitedA. Enqueue D.

**Iteration 2:**
- queueA dequeues B. visitedA = {C, B}. Not in visitedB. Enqueue A.
- queueB dequeues D. visitedB = {E, D}. Not in visitedA. Enqueue B.

**Iteration 3:**
- queueA dequeues A. visitedA = {C, B, A}. Not in visitedB. Enqueue root.
- queueB dequeues B. visitedB = {E, D, B}. **Is B in visitedA? YES!** → return "B"

Result: B is the merge base. ✓

---

## 7. Fast-Forward vs True Merge

Before doing a three-way merge, check if one branch is an ancestor of the other:

```csharp
bool IsAncestor(string possibleAncestor, string descendant, IObjectStore store)
{
    // BFS from descendant — if we reach possibleAncestor, it IS an ancestor
    var visited = new HashSet<string>();
    var queue   = new Queue<string>();
    queue.Enqueue(descendant);

    while (queue.Count > 0)
    {
        var hash = queue.Dequeue();
        if (!visited.Add(hash)) continue;
        if (hash == possibleAncestor) return true;
        EnqueueParents(hash, queue, store);
    }

    return false;
}

// In MergeCommand:
if (IsAncestor(featureTip, mainTip, store))
{
    Console.WriteLine("Already up to date.");
    return;
}

if (IsAncestor(mainTip, featureTip, store))
{
    // Fast-forward: just move the branch pointer
    refs.SetRef("refs/heads/main", featureTip);
    Console.WriteLine("Fast-forward merge.");
    return;
}

// Otherwise: find merge base and do a three-way merge
var mergeBase = FindMergeBase(mainTip, featureTip, store);
```

---

## Exercises

**Exercise 1 — Trace by hand**
Draw the DAG:
```
root ← A ← B ← C ← D  (main, at D)
              ↑
              E ← F ← G  (feature, at G)
```
Find the LCA of D and G by tracing BFS manually on paper.

**Exercise 2 — Implement FindLca**
Using the two-pass approach (collect all ancestors of A, then BFS from B):
```csharp
string? FindLca(string hashA, string hashB, Dictionary<string, List<string>> parents)
```
Test with the diamond pattern: `{root: [], A: [root], B: [root], C: [A, B]}`
LCA of A and B should be root. LCA of C and A should be A.

**Exercise 3 — Interleaved BFS**
Implement the interleaved BFS version.
Verify it gives the same result as the two-pass version on several test DAGs.

**Exercise 4 — Detect fast-forward**
Given a DAG, implement `bool IsAncestor(string a, string b, Dictionary<...> parents)`.
Test all combinations:
- IsAncestor(root, C) → true (root is an ancestor of C)
- IsAncestor(C, root) → false
- IsAncestor(A, B) → false (siblings)

**Exercise 5 — Full merge simulation**
Given two branch tips (C and E from earlier), simulate the full merge decision:
1. Is one an ancestor of the other? → fast-forward
2. If not, find the merge base
3. Print: "Merge base: {hash}"
