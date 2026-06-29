# Git Internals

## Learning Resources

**Books — Free Online**
- [Pro Git, 2nd Edition by Scott Chacon & Ben Straub](https://git-scm.com/book/en/v2) — **read chapters 10.1–10.3 (Git Internals)**. This is the definitive free resource. The internals chapter shows exactly what files Git creates and why.

**Books — Paid**
- *Git Internals* by Scott Chacon (Peepcode PDF) — an older but still excellent short book focused entirely on Git's object model and storage.

**Articles / Guides**
- [Git Objects — Pro Git Chapter 10.2](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) — blobs, trees, commits, and tags with hands-on commands.
- [Git References — Pro Git Chapter 10.3](https://git-scm.com/book/en/v2/Git-Internals-Git-References) — HEAD, branches, tags.
- [Git Packfiles — Pro Git Chapter 10.4](https://git-scm.com/book/en/v2/Git-Internals-Packfiles) — how Git compresses objects (stretch goal).
- [Git Index format — Git source documentation](https://git-scm.com/docs/index-format) — the binary format spec.
- [Think Like (a) Git](http://think-like-a-git.net/) — a free tutorial that builds intuition through visualisations.

---

## 1. The Four Object Types

Everything stored in a Git repository is one of four object types.

### Blob
Stores raw file content — nothing else. No filename, no path, no permissions.

```
Content of .git/objects/ce/013625030ba8dba906f756967f9e9ca394464a:
  blob 6\0hello\n

Breaking it down:
  "blob"  — the type header
  " 6"    — a space followed by the byte length of the content
  "\0"    — a null byte separator
  "hello\n" — the actual content (6 bytes)
```

The SHA-1 hash of the entire header+content string is the object's identity.
Mini-Git uses SHA-256 instead.

### Tree
Stores a directory snapshot. Lists files and subdirectories with their modes, names, and hashes.

```
Content (binary in real Git, readable format from git cat-file -p):
  100644 blob abc123 README.md
  100644 blob def456 Program.cs
  040000 tree ghi789 src
```

Each line: `{mode} {type} {hash}\t{name}`

### Commit
Stores a snapshot reference, author information, and parent links.

```
tree aaa111bbb222...          ← hash of the root TreeObject
parent ddd444eee555...        ← hash of the parent CommitObject (absent for root commit)
author Alice <alice@x.com> 1705320000 +0200
committer Alice <alice@x.com> 1705320000 +0200

initial commit                ← blank line, then the message
```

For a merge commit: two `parent` lines.

### Tag
A named, annotated pointer to any object (usually a commit).

```
object abc123...
type commit
tag v1.0
tagger Alice <alice@x.com> 1705320000 +0200

Release version 1.0
```

Mini-Git does not need to implement tags — focus on blob, tree, commit.

---

## 2. The Object Store Layout

```
.git/objects/
  e3/
    b0c44298fc1c149afb4c8996fb92427ae41e4649b934ca495991b7852b855
  ce/
    013625030ba8dba906f756967f9e9ca394464a
  pack/
    pack-abc123.idx
    pack-abc123.pack
```

- First 2 hex characters of the hash → subdirectory name
- Remaining 38 characters → filename
- Objects are zlib-compressed (real Git). Mini-Git stores uncompressed for simplicity.

---

## 3. HEAD and References

### HEAD
`.git/HEAD` is a text file. It usually contains a **symbolic reference**:

```
ref: refs/heads/main
```

This means "HEAD points to the `main` branch". When `main` moves forward (a new commit), HEAD follows automatically.

**Detached HEAD** state: HEAD contains a raw commit hash instead of a branch name:
```
d22a981b5c2a05d0f4e91a2b3c4d5e6f7a8b9c0d
```

This happens when you checkout a commit directly (not a branch).

### Branches
Each branch is a file containing a commit hash:

```
.git/refs/heads/main    → "d22a981b5c2a05d..."
.git/refs/heads/feature → "abc1234def5678..."
```

Creating a branch = creating one of these files.
Moving a branch = updating the file's content.

### Tags
```
.git/refs/tags/v1.0    → "abc123..."  (lightweight tag — just a hash)
.git/objects/...       → annotated tag object (full object with message)
```

### Resolving HEAD to a commit hash

```csharp
string ResolveHead(string repoRoot)
{
    var headPath = Path.Combine(repoRoot, ".git", "HEAD");
    var headContent = File.ReadAllText(headPath).Trim();

    if (headContent.StartsWith("ref: "))
    {
        // Symbolic ref — resolve the branch
        var refName = headContent[5..];  // e.g. "refs/heads/main"
        var refPath = Path.Combine(repoRoot, ".git", refName);
        return File.ReadAllText(refPath).Trim();
    }
    else
    {
        // Detached HEAD — content is already a commit hash
        return headContent;
    }
}
```

---

## 4. The Index (Staging Area)

Real Git's index is a binary file at `.git/index`. It stores the current staging state.

Human-readable view:
```bash
git ls-files --stage
# 100644 abc123... 0    README.md
# 100644 def456... 0    src/Program.cs
```

Columns: `{mode} {object-hash} {stage-number} {filename}`

- **Mode** `100644` = regular file, `100755` = executable, `120000` = symlink
- **Stage number** 0 = normal, 1/2/3 = merge conflict (base/ours/theirs)
- In Mini-Git, the index stores the same information as JSON instead of binary

---

## 5. Fast-Forward vs Three-Way Merge

### Fast-Forward

When the target branch is a **direct ancestor** of the source branch:

```
Before:
  main:    A ← B ← C   (HEAD)
  feature: A ← B ← C ← D ← E

main can be "fast-forwarded" to E because C is in E's history.
No new commit needed — just move the branch pointer.

After:
  main:    A ← B ← C ← D ← E   (HEAD)
  feature: same as main
```

```bash
git merge feature
# Fast-forward
```

### Three-Way Merge

When branches have **diverged** (they have different commits since their common ancestor):

```
Before:
  main:    A ← B ← C ← F   (HEAD)
  feature: A ← B ← C ← D ← E
                ↑
                merge base (C)

Three-way merge needs:
  Base:   C's tree     (where they diverged)
  Ours:   F's tree     (our latest)
  Theirs: E's tree     (their latest)

For each file:
  - If only we changed it → use our version
  - If only they changed it → use their version
  - If both changed it differently → CONFLICT
  - If both changed it identically → use either (same result)

After: a new merge commit M with two parents (F and E)
  main: A ← B ← C ← F ← M   (HEAD)
                  ↑ ← D ← E ← ┘
```

---

## 6. Hands-On Study Session

Run these commands in a fresh terminal to see Git's internal structure:

```bash
# Set up a fresh repo
mkdir git-study && cd git-study
git init

# Create a file and stage it
echo "hello" > file.txt
git add file.txt

# Inspect the index
git ls-files --stage
# 100644 ce013625030ba8dba906f756967f9e9ca394464a 0    file.txt

# The blob object was just created
git cat-file -t ce013625030ba8dba906f756967f9e9ca394464a
# blob
git cat-file -p ce013625030ba8dba906f756967f9e9ca394464a
# hello

# Make a commit
git commit -m "initial commit"

# Find HEAD
cat .git/HEAD
# ref: refs/heads/main
cat .git/refs/heads/main
# (the commit hash)

# Inspect the commit object
git cat-file -p HEAD
# tree ...
# author ...
# initial commit

# Inspect the tree
git cat-file -p HEAD^{tree}
# 100644 blob ce013625... file.txt

# Make another commit and inspect the parent link
echo "world" >> file.txt
git add file.txt
git commit -m "second commit"
git cat-file -p HEAD
# tree ...
# parent (first commit hash)
# second commit
```

---

## 7. What Mini-Git Mirrors

| Real Git | Mini-Git |
|---|---|
| `.git/` | `.minigit/` |
| SHA-1 hashes | SHA-256 hashes |
| zlib-compressed objects | Uncompressed objects |
| Binary index format | JSON index |
| `git init` | `minigit init` |
| `git hash-object` | `minigit hash-object` |
| `git cat-file` | `minigit cat-file` |
| `git add` | `minigit add` |
| `git commit` | `minigit commit` |
| `git log` | `minigit log` |
| `git diff` | `minigit diff` |
| `git merge` | `minigit merge` |

---

## Exercises

**Exercise 1 — Complete hands-on session**
Follow every command in Section 6. After each command, write down what you see and what it means.
Build a glossary: one sentence per concept (blob, tree, commit, HEAD, ref, stage).

**Exercise 2 — Explore a real repo's objects**
In any existing Git repo on your machine:
- Run `git cat-file -p HEAD` and trace all the hashes
- Find a blob and a tree in `.git/objects/`
- Run `git log --oneline` and trace the parent chain manually with `git cat-file`

**Exercise 3 — Three-way merge on paper**
Create a scenario where two branches both modify the same file differently.
Draw the commit DAG. Identify the merge base. List the three file versions (base/ours/theirs).
Manually determine what the merged result should be.

**Exercise 4 — Reproduce a commit hash**
In a fresh Git repo, make a commit. Note the commit hash.
Then run `git cat-file -p HEAD` to get the exact content of the commit object.
Try to compute the SHA-1 yourself:
```
sha1("commit {length}\0{content}")
```
Verify your result matches what Git computed.
(Note: Git uses SHA-1, Mini-Git will use SHA-256 — adapt accordingly.)

**Exercise 5 — Design Mini-Git's .minigit layout**
Without looking at the project spec, design the complete `.minigit/` directory structure on paper.
For each file, write one sentence about what format it uses and what it stores.
Then compare to the spec in the main project README.
