# Recursion

## Learning Resources

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 4 (Divide-and-Conquer). Recursion is the implementation mechanism behind divide-and-conquer algorithms.
- *Grokking Algorithms* — Chapter 3 (Recursion). Excellent visual introduction — highly recommended for beginners.
- *Structure and Interpretation of Computer Programs (SICP)* — Chapter 1.2 (Procedures and the Processes They Generate). The classic treatment of recursion vs iteration.
- *The Recursive Book of Recursion* by Al Sweigart — entirely dedicated to recursion with practical examples. Very approachable.

**Articles / Videos**
- [Recursion — CS50 Harvard (YouTube)](https://www.youtube.com/watch?v=mz6tAJMVmfM) — visual and approachable explanation with stack diagrams.
- [Recursion vs Iteration — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/statements)

---

## 1. The Core Idea

A recursive function calls **itself** to solve a smaller version of the same problem.

Every recursive function has two mandatory parts:
1. **Base case** — a condition where the function does NOT call itself (stops the recursion)
2. **Recursive case** — a call to itself with a *smaller* or *simpler* input

If either is missing:
- No base case → infinite recursion → stack overflow
- No progress → same inputs forever → infinite recursion → stack overflow

---

## 2. The Call Stack

Every function call creates a **stack frame** — a block of memory holding the function's local variables and where to return to when done.

Recursive calls stack up these frames. When the base case is reached, frames unwind one by one.

```csharp
int Factorial(int n)
{
    if (n <= 1) return 1;           // base case
    return n * Factorial(n - 1);   // recursive case
}

// Calling Factorial(4) creates these frames:
// Factorial(4) — waiting for Factorial(3) to return
//   Factorial(3) — waiting for Factorial(2) to return
//     Factorial(2) — waiting for Factorial(1) to return
//       Factorial(1) — returns 1         ← base case, stack starts unwinding
//     returns 2 * 1 = 2
//   returns 3 * 2 = 6
// returns 4 * 6 = 24
```

**Stack overflow** happens when recursion is too deep — you create more stack frames than the stack can hold. C# has a default stack size of ~1 MB, which allows tens of thousands of recursive calls depending on the frame size. For normal directory trees this is not a problem.

---

## 3. Three Mental Models for Recursion

### Model 1: Leap of Faith
Assume the function already works correctly for smaller inputs. Use it to solve the larger input.

```csharp
int Sum(int[] arr, int index)
{
    if (index >= arr.Length) return 0;          // base: empty remainder
    return arr[index] + Sum(arr, index + 1);   // add current + (trust the rest is correct)
}
```

### Model 2: Smallest Example First
Find the smallest input the function must handle (the base case). Then find what one step of progress looks like (the recursive case).

For directory walking:
- Smallest case: a directory with no subdirectories (only files)
- One step of progress: handle one subdirectory by recursing into it

### Model 3: Unwinding
Think through how the call stack grows, then unwinds. Draw the stack on paper for small examples.

---

## 4. Recursion in Mini-Git: Building Trees

The core use of recursion in Mini-Git is walking a directory to build a `TreeObject`.

### The problem
Given `project/`, which contains files and subdirectories, create a `TreeObject` that captures the entire snapshot.

### Why recursion?
The structure of the problem mirrors the structure of the code:
- A directory (the problem) contains files and **other directories** (the same problem, smaller)

### Solution

```csharp
string BuildTree(string dirPath, IObjectStore store)
{
    var entries = new List<TreeEntry>();

    // 1. Handle all files in this directory (base-case-like: no recursion needed for files)
    foreach (var filePath in Directory.GetFiles(dirPath).Order())
    {
        var name    = Path.GetFileName(filePath);
        var content = File.ReadAllBytes(filePath);
        var hash    = store.Store(content);          // store blob
        entries.Add(new TreeEntry("100644", name, hash));
    }

    // 2. Handle all subdirectories — this is the recursive case
    foreach (var subDir in Directory.GetDirectories(dirPath).Order())
    {
        var name = Path.GetFileName(subDir);
        if (name == ".minigit") continue;             // skip the repo itself

        var hash = BuildTree(subDir, store);          // ← RECURSIVE CALL
        entries.Add(new TreeEntry("040000", name, hash));
    }

    // 3. Serialize and store this tree, return its hash
    return StoreTree(entries, store);
}
```

**Call stack for `project/`:**
```
BuildTree("project/")
  stores README.md blob
  BuildTree("project/src/")            ← recursive call
    stores Program.cs blob
    BuildTree("project/src/Core/")     ← recursive call
      stores Blob.cs blob
      returns tree hash "core-tree"
    returns tree hash "src-tree"
  BuildTree("project/tests/")          ← recursive call
    stores BlobTests.cs blob
    returns tree hash "tests-tree"
  returns tree hash "root-tree"        ← final result
```

---

## 5. Recursion in Mini-Git: Restoring Files (checkout)

The reverse operation: given a `TreeObject` hash, restore the directory structure.

```csharp
void RestoreTree(string treeHash, string targetDir, IObjectStore store)
{
    Directory.CreateDirectory(targetDir);
    var tree = DeserializeTree(store.Retrieve(treeHash));

    foreach (var entry in tree.Entries)
    {
        var targetPath = Path.Combine(targetDir, entry.Name);

        if (entry.Mode == "100644")
        {
            // It's a file — write the blob content (base case)
            var content = store.Retrieve(entry.Hash);
            File.WriteAllBytes(targetPath, content);
        }
        else if (entry.Mode == "040000")
        {
            // It's a directory — recurse into it
            RestoreTree(entry.Hash, targetPath, store);   // ← RECURSIVE CALL
        }
    }
}
```

---

## 6. Iteration vs Recursion

Many recursive algorithms can be rewritten iteratively (using a stack explicitly):

```csharp
// Recursive directory walk
void WalkRecursive(string path)
{
    foreach (var file in Directory.GetFiles(path))
        Process(file);
    foreach (var dir in Directory.GetDirectories(path))
        WalkRecursive(dir);     // ← call stack handles the "stack"
}

// Iterative equivalent (explicit stack)
void WalkIterative(string rootPath)
{
    var stack = new Stack<string>();
    stack.Push(rootPath);

    while (stack.Count > 0)
    {
        var path = stack.Pop();
        foreach (var file in Directory.GetFiles(path))
            Process(file);
        foreach (var dir in Directory.GetDirectories(path))
            stack.Push(dir);    // ← manual stack instead of call stack
    }
}
```

For Mini-Git's tree building, the recursive version is cleaner and the call depth is bounded by directory depth (rarely more than 10–20 levels).

---

## 7. Common Mistakes

```csharp
// MISTAKE 1: Missing base case — infinite recursion
int CountDown(int n)
{
    Console.WriteLine(n);
    return CountDown(n - 1);   // never stops!
}

// MISTAKE 2: Not making progress
int CountDown(int n)
{
    if (n == 0) return 0;
    return CountDown(n);    // n never changes — infinite recursion
}

// CORRECT
int CountDown(int n)
{
    if (n <= 0) return 0;        // base case
    Console.WriteLine(n);
    return CountDown(n - 1);    // progress: n decreases by 1 each call
}
```

---

## Exercises

**Exercise 1 — Sum of a list**
Write `int Sum(List<int> numbers)` recursively.
Base case: empty list returns 0.
Recursive case: first element + Sum(rest of list).

**Exercise 2 — Flatten**
Write `List<string> Flatten(List<object> items)` where items can contain either strings or other `List<object>`s.
Use recursion to return all strings at any nesting depth.
Example: `["a", ["b", ["c", "d"]], "e"]` → `["a", "b", "c", "d", "e"]`

**Exercise 3 — Directory byte count**
Write `long TotalBytes(string dirPath)` that returns the total size of all files in a directory and all its subdirectories, using recursion.

**Exercise 4 — Build and restore a tree**
1. Create a temp directory with some files and subdirectories
2. Run `BuildTree(tempDir, store)` to get a root tree hash
3. Create a second temp directory (empty)
4. Run `RestoreTree(rootHash, emptyDir, store)` to restore
5. Verify both directories have the same files and contents

**Exercise 5 — Fibonacci (classic)**
Write `int Fibonacci(int n)` recursively.
Then time how long `Fibonacci(40)` takes.
Observe the exponential explosion (millions of redundant calls).
Fix it with memoization using a `Dictionary<int, int>` cache.
