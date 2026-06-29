# Myers Diff Algorithm

## Learning Resources

**Papers**
- *An O(ND) Difference Algorithm and Its Variations* by Eugene W. Myers (1986) — the original paper. Surprisingly readable. 25 pages. Available by searching the title.

**Articles / Posts**
- [The Myers Diff Algorithm — James Coglan's blog](https://blog.jcoglan.com/2017/02/12/the-myers-diff-algorithm-part-1/) — a four-part series with excellent diagrams and Ruby code. The best practical explanation available.
- [Diff algorithms — Robert Elder](https://blog.robertelder.org/diff-algorithm/) — visual walkthrough.
- [Edit distance — Wikipedia](https://en.wikipedia.org/wiki/Edit_distance)

**Books**
- *Introduction to Algorithms (CLRS)* — Chapter 15.4 (Longest Common Subsequence). Myers diff is closely related to LCS — study LCS first.
- *Algorithm Design Manual* by Skiena — Chapter 8 (Dynamic Programming) covers edit distance.

---

## 1. The Problem: What Is a Diff?

Given two sequences of lines (old file, new file), produce the shortest list of changes (additions and deletions) that transforms the old file into the new file.

```
Old file:          New file:
  A                  A
  B                  C
  C                  D
  D
```

The diff:
```
  A          ← unchanged (match)
- B          ← deleted (was in old, not in new)
  C          ← unchanged
  D          ← unchanged
```

The goal is to find the **minimum edit distance** — the smallest number of insertions and deletions.

---

## 2. Edit Distance (Levenshtein Distance) — Start Here

Before Myers, understand edit distance. Given two strings (or sequences), the edit distance is the minimum number of single-element insertions, deletions, and substitutions to transform one into the other.

For diff purposes, we only use **insertions and deletions** (no substitutions — a substitution = 1 delete + 1 insert).

**Example:**
```
Old: [A, B, C, D]
New: [A, C, D, E]

One possible edit sequence (cost: 2):
  Keep A
  Delete B
  Keep C
  Keep D
  Insert E

Another (also cost 2):
  Keep A
  Delete B
  Keep C
  Keep D
  Insert E
```

---

## 3. The Edit Graph

Myers frames the problem as a shortest-path problem on a grid:

- Rows = elements of the old sequence (deletions move down)
- Columns = elements of the new sequence (insertions move right)
- Diagonal moves = elements match (no edit needed)

```
       ""   A    C    D    E       ← new sequence
    ┌────┬────┬────┬────┬────┐
 "" │(0,0)→  →   →   →   →  │
    │  ↓  ↘  ↓   ↓   ↓   ↓  │
  A │  ↓   (1,1)→  →   →   →│
    │  ↓    ↓  ↘   ↓   ↓   ↓│
  B │  ↓    ↓   ↓ (no match)↓│
    │  ↓    ↓   ↓   ↓   ↓   ↓│
  C │  ↓    ↓   (2,2)→  →   ↓│
    │  ↓    ↓   ↓  ↘   ↓   ↓│
  D │  ↓    ↓   ↓   (3,3)→ ↓│
    └────┴────┴────┴────┴────┘
                              (4,4) = goal
```

- Moving **right** = insert an element from new (cost 1)
- Moving **down** = delete an element from old (cost 1)
- Moving **diagonally** (↘) = elements match (cost 0, called a "snake")

The shortest path from (0,0) to (end,end) is the minimum-cost edit sequence.

---

## 4. Naive Approach: Dynamic Programming

```csharp
int EditDistance(string[] oldLines, string[] newLines)
{
    int m = oldLines.Length;
    int n = newLines.Length;
    var dp = new int[m + 1, n + 1];

    // Base cases: cost to convert empty to sequence = length of sequence
    for (int i = 0; i <= m; i++) dp[i, 0] = i;  // delete all old lines
    for (int j = 0; j <= n; j++) dp[0, j] = j;  // insert all new lines

    for (int i = 1; i <= m; i++)
    {
        for (int j = 1; j <= n; j++)
        {
            if (oldLines[i - 1] == newLines[j - 1])
                dp[i, j] = dp[i - 1, j - 1];       // match (diagonal, free)
            else
                dp[i, j] = 1 + Math.Min(
                    dp[i - 1, j],    // delete from old
                    dp[i, j - 1]     // insert from new
                );
        }
    }

    return dp[m, n];
}
```

This is O(MN) time and O(MN) space. Myers improves this to O(ND) time, where D is the number of edits — much faster when files are similar (D is small).

---

## 5. Myers Algorithm: Key Insight

Myers reformulates the problem using **diagonals** instead of the full grid.

**Diagonal k** = the set of grid positions where `column - row = k`.

```
k = -2:  (2,0), (3,1), (4,2), ...
k = -1:  (1,0), (2,1), (3,2), (4,3), ...
k =  0:  (0,0), (1,1), (2,2), (3,3), ...  (the main diagonal)
k = +1:  (0,1), (1,2), (2,3), ...
k = +2:  (0,2), (1,3), (2,4), ...
```

**The algorithm:**
- Tracks `V[k]` = the furthest point reached on diagonal k
- Each iteration (`d` = number of edits so far), extends from all reached diagonals by one move (right or down), then slides along any matching diagonal (a "snake")
- Stops when the goal `(m, n)` is reached

The number of iterations D equals the edit distance (minimum number of edits).

---

## 6. Myers Algorithm: Implementation

```csharp
public static List<DiffLine> Diff(string[] oldLines, string[] newLines)
{
    int m = oldLines.Length;
    int n = newLines.Length;
    int max = m + n;

    // V[k] = furthest x reached on diagonal k
    // Offset by max because k can be negative
    var v = new int[2 * max + 1];
    var trace = new List<int[]>();  // record V after each edit step

    for (int d = 0; d <= max; d++)
    {
        trace.Add((int[])v.Clone());

        for (int k = -d; k <= d; k += 2)
        {
            int x;
            // Choose: move right (insert) or move down (delete)?
            if (k == -d || (k != d && v[k - 1 + max] < v[k + 1 + max]))
                x = v[k + 1 + max];      // move right (insert) — take from k+1
            else
                x = v[k - 1 + max] + 1;  // move down (delete) — take from k-1, step x

            int y = x - k;

            // Slide along snake (matching elements)
            while (x < m && y < n && oldLines[x] == newLines[y])
            {
                x++;
                y++;
            }

            v[k + max] = x;

            if (x >= m && y >= n)
                return Backtrack(trace, oldLines, newLines);  // found the path!
        }
    }

    return new List<DiffLine>(); // should not reach here
}
```

### Backtracking to produce the actual diff

After finding the minimum edit distance, backtrack through the recorded `trace` to reconstruct which lines were inserted, deleted, or kept.

```csharp
static List<DiffLine> Backtrack(List<int[]> trace, string[] oldLines, string[] newLines)
{
    var result = new List<DiffLine>();
    int x = oldLines.Length, y = newLines.Length;

    for (int d = trace.Count - 1; d >= 0; d--)
    {
        var v   = trace[d];
        int max = (v.Length - 1) / 2;
        int k   = x - y;

        int prevK;
        if (k == -d || (k != d && v[k - 1 + max] < v[k + 1 + max]))
            prevK = k + 1;  // we came from an insert move
        else
            prevK = k - 1;  // we came from a delete move

        int prevX = v[prevK + max];
        int prevY = prevX - prevK;

        // Collect snake (matching lines, going forward)
        while (x > prevX && y > prevY)
        {
            result.Add(new DiffLine(' ', oldLines[x - 1]));  // kept
            x--;
            y--;
        }

        if (d > 0)
        {
            if (x == prevX)
                result.Add(new DiffLine('+', newLines[y - 1]));  // inserted
            else
                result.Add(new DiffLine('-', oldLines[x - 1]));  // deleted
        }

        x = prevX;
        y = prevY;
    }

    result.Reverse();
    return result;
}

public record DiffLine(char Tag, string Content)
{
    public override string ToString() => $"{Tag} {Content}";
}
```

---

## 7. Reading the Output

```
  A     ← space = unchanged (match)
- B     ← minus = deleted from old
  C     ← unchanged
  D     ← unchanged
+ E     ← plus = inserted in new
```

---

## 8. Suggested Study Order

1. **Edit distance (DP)** — understand the grid and the DP table. Code it.
2. **Longest Common Subsequence (LCS)** — closely related. The LCS of old and new = the unchanged lines.
3. **Myers short summary** — understand diagonals and why they are faster.
4. **Myers implementation** — work through the forward pass, then backtracking.
5. Read James Coglan's blog series (linked above) — it is the clearest walkthrough available.

---

## Exercises

**Exercise 1 — Edit distance (DP)**
Implement `int EditDistance(string[] a, string[] b)` using the DP table approach.
Test cases:
- `(["A","B","C"], ["A","C"])` → 1
- `(["A","B"], ["C","D"])` → 4 (delete both, insert both)
- `(["A"], ["A"])` → 0

**Exercise 2 — Visualise the edit grid**
For `old = ["A","B","C"]` and `new = ["A","C","D"]`:
Draw the edit grid on paper. Mark the diagonal moves (matches). Find the shortest path by hand.
What is the edit distance? What are the actual edits?

**Exercise 3 — Simple diff output**
Without implementing Myers, write a `Diff` function using the LCS approach:
- Find the LCS of old and new (the lines that stay)
- Lines in old but not in LCS → `-`
- Lines in new but not in LCS → `+`
- Lines in LCS → ` ` (space)

**Exercise 4 — Myers forward pass**
Implement just the forward pass of Myers (the part that finds D, the edit distance).
Do not worry about backtracking yet.
Test: `old = ["A","B","C"]`, `new = ["A","C","D"]` → D should be 2.

**Exercise 5 — Full Myers**
Add backtracking to your Exercise 4 implementation.
Verify the output matches the DP approach from Exercise 3.
