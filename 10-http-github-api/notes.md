# HTTP & the GitHub API

## Learning Resources

**Official Docs**
- [HttpClient — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/api/system.net.http.httpclient)
- [Make HTTP requests with HttpClient — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient)
- [GitHub REST API documentation](https://docs.github.com/en/rest)
- [GitHub Git Data API](https://docs.github.com/en/rest/git) — the four endpoints for creating commits

**Books**
- *C# in a Nutshell* — Chapter 16 (Networking). Covers `HttpClient`, REST, and JSON with the modern .NET APIs.
- *HTTP: The Definitive Guide* by Gourley & Totty — comprehensive but optional. Good if you want to understand HTTP deeply.

**Articles**
- [GitHub API: Creating a file — official guide](https://docs.github.com/en/rest/repos/contents#create-or-update-file-contents)
- [GitHub: Understanding the Git data API](https://docs.github.com/en/rest/git/commits)

---

## 1. HTTP Fundamentals

### Request/Response Model
Every HTTP interaction is a request from a client and a response from a server.

```
Client → Request  → Server
Client ← Response ← Server
```

**Request structure:**
```
POST /repos/alice/my-repo/git/blobs HTTP/1.1
Host: api.github.com
Authorization: Bearer ghp_mytoken123
Content-Type: application/json
Accept: application/vnd.github+json

{"content": "aGVsbG8=", "encoding": "base64"}
```

**Response structure:**
```
HTTP/1.1 201 Created
Content-Type: application/json

{"sha": "abc123...", "url": "https://api.github.com/..."}
```

### HTTP Verbs (Methods)

| Verb | Meaning | Idempotent? |
|------|---------|-------------|
| GET | Read a resource, no side effects | Yes |
| POST | Create a new resource | No |
| PUT | Replace a resource entirely | Yes |
| PATCH | Partially update a resource | Sometimes |
| DELETE | Delete a resource | Yes |

For Mini-Git push:
- `POST /git/blobs` — create a blob
- `POST /git/trees` — create a tree
- `POST /git/commits` — create a commit
- `PATCH /git/refs/heads/main` — update branch pointer

### Status Codes

| Code | Meaning | When you see it |
|------|---------|-----------------|
| 200 OK | Success (read) | GET succeeded |
| 201 Created | Resource created | POST succeeded |
| 204 No Content | Success, no body | DELETE succeeded |
| 400 Bad Request | Malformed request | Wrong JSON |
| 401 Unauthorized | Not authenticated | Missing or bad token |
| 404 Not Found | Resource doesn't exist | Wrong repo/path |
| 409 Conflict | Conflict | Branch tip has changed |
| 422 Unprocessable | Validation error | Invalid input |
| 429 Too Many Requests | Rate limited | Slow down |

---

## 2. HttpClient in C\#

```csharp
using System.Net.Http;
using System.Net.Http.Headers;

// IMPORTANT: create ONE HttpClient per application, reuse it
// Creating a new one per request causes socket exhaustion
private static readonly HttpClient _client = new();

// --- Setup default headers ---
_client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", Environment.GetEnvironmentVariable("GITHUB_TOKEN"));
_client.DefaultRequestHeaders.UserAgent.ParseAdd("mini-git/1.0");
_client.DefaultRequestHeaders.Accept.ParseAdd("application/vnd.github+json");
```

### GET request

```csharp
HttpResponseMessage response = await _client.GetAsync("https://api.github.com/user");
response.EnsureSuccessStatusCode();  // throws if 4xx/5xx
string body = await response.Content.ReadAsStringAsync();
var user = JsonSerializer.Deserialize<GitHubUser>(body);
Console.WriteLine($"Authenticated as: {user.Login}");
```

### POST request with JSON body

```csharp
var payload = new { content = Convert.ToBase64String(fileBytes), encoding = "base64" };
var json    = JsonSerializer.Serialize(payload);
var content = new StringContent(json, Encoding.UTF8, "application/json");

var response = await _client.PostAsync(url, content);
response.EnsureSuccessStatusCode();

var body = await response.Content.ReadAsStringAsync();
var result = JsonSerializer.Deserialize<BlobResponse>(body);
```

### PATCH request

```csharp
var payload = new { sha = newCommitHash, force = false };
var json    = JsonSerializer.Serialize(payload);
var content = new StringContent(json, Encoding.UTF8, "application/json");

var response = await _client.PatchAsync(url, content);
response.EnsureSuccessStatusCode();
```

---

## 3. GitHub Personal Access Token (PAT)

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Scopes needed: `repo` (full control of private repositories)
4. Copy the token — you will not see it again

**Never hardcode the token in your code.** Store it in an environment variable:

```powershell
$env:GITHUB_TOKEN = "ghp_your_token_here"
```

Or in a `.env` file (never commit this file):
```
GITHUB_TOKEN=ghp_your_token_here
```

In code, read it:
```csharp
var token = Environment.GetEnvironmentVariable("GITHUB_TOKEN")
    ?? throw new InvalidOperationException("GITHUB_TOKEN environment variable is not set");
```

---

## 4. The Four API Calls to Push a Commit

To push a local commit to GitHub, you make four API calls in sequence. The output of each becomes the input to the next.

### Step 1: Create Blobs

For each file in the commit's tree, upload the content as a blob.

```csharp
async Task<string> CreateBlob(string owner, string repo, byte[] fileContent)
{
    var url = $"https://api.github.com/repos/{owner}/{repo}/git/blobs";
    var payload = new
    {
        content  = Convert.ToBase64String(fileContent),
        encoding = "base64"
    };

    var response = await PostJson(url, payload);
    var result   = JsonSerializer.Deserialize<BlobResponse>(response);
    return result!.Sha;  // the blob SHA on GitHub
}

record BlobResponse([property: JsonPropertyName("sha")] string Sha);
```

### Step 2: Create a Tree

Assemble all the blob SHAs into a tree.

```csharp
async Task<string> CreateTree(string owner, string repo, string? baseTreeSha,
                               IEnumerable<(string path, string blobSha)> files)
{
    var url = $"https://api.github.com/repos/{owner}/{repo}/git/trees";
    var treeItems = files.Select(f => new
    {
        path = f.path,
        mode = "100644",
        type = "blob",
        sha  = f.blobSha
    });

    var payload  = new { base_tree = baseTreeSha, tree = treeItems };
    var response = await PostJson(url, payload);
    var result   = JsonSerializer.Deserialize<TreeResponse>(response);
    return result!.Sha;
}

record TreeResponse([property: JsonPropertyName("sha")] string Sha);
```

### Step 3: Create a Commit

```csharp
async Task<string> CreateCommit(string owner, string repo, string message,
                                 string treeSha, string? parentSha)
{
    var url = $"https://api.github.com/repos/{owner}/{repo}/git/commits";
    var payload = new
    {
        message,
        tree    = treeSha,
        parents = parentSha != null ? new[] { parentSha } : Array.Empty<string>()
    };

    var response = await PostJson(url, payload);
    var result   = JsonSerializer.Deserialize<CommitResponse>(response);
    return result!.Sha;
}

record CommitResponse([property: JsonPropertyName("sha")] string Sha);
```

### Step 4: Update the Branch Ref

```csharp
async Task UpdateBranchRef(string owner, string repo, string branch, string commitSha)
{
    var url = $"https://api.github.com/repos/{owner}/{repo}/git/refs/heads/{branch}";
    var payload = new { sha = commitSha, force = false };

    var json    = JsonSerializer.Serialize(payload);
    var content = new StringContent(json, Encoding.UTF8, "application/json");
    var response = await _client.PatchAsync(url, content);
    response.EnsureSuccessStatusCode();
}
```

### Putting it together

```csharp
public async Task Push(string owner, string repo, string branch, CommitObject localCommit,
                        IObjectStore store, IRefStore refs)
{
    // 1. Get current remote HEAD
    var remoteHead = await GetRemoteBranchSha(owner, repo, branch);

    // 2. Upload all blobs
    var blobMap = new Dictionary<string, string>(); // local hash → github sha
    foreach (var entry in GetAllBlobEntries(localCommit, store))
    {
        var content  = store.Retrieve(entry.LocalHash);
        var githubSha = await CreateBlob(owner, repo, content);
        blobMap[entry.LocalHash] = githubSha;
    }

    // 3. Create tree
    var files     = GetFilePaths(localCommit, store, blobMap);
    var treeSha   = await CreateTree(owner, repo, null, files);

    // 4. Create commit
    var commitSha = await CreateCommit(owner, repo, localCommit.Message, treeSha, remoteHead);

    // 5. Update branch
    await UpdateBranchRef(owner, repo, branch, commitSha);

    Console.WriteLine($"[{branch} {commitSha[..7]}] {localCommit.Message}");
}
```

---

## 5. Rate Limiting

GitHub allows 5,000 API requests per hour for authenticated users. Check the response headers:

```csharp
var response = await _client.GetAsync(url);

var remaining = response.Headers.GetValues("X-RateLimit-Remaining").FirstOrDefault();
var resetAt   = response.Headers.GetValues("X-RateLimit-Reset").FirstOrDefault();

Console.WriteLine($"Rate limit remaining: {remaining}");

if (response.StatusCode == HttpStatusCode.TooManyRequests)
{
    // Wait until reset time
    var resetUnix = long.Parse(resetAt!);
    var resetTime = DateTimeOffset.FromUnixTimeSeconds(resetUnix);
    Console.WriteLine($"Rate limited. Resets at {resetTime:HH:mm:ss}");
}
```

---

## 6. Error Handling for HTTP

```csharp
async Task<string> PostJson(string url, object payload)
{
    var json    = JsonSerializer.Serialize(payload);
    var content = new StringContent(json, Encoding.UTF8, "application/json");

    HttpResponseMessage response;
    try
    {
        response = await _client.PostAsync(url, content);
    }
    catch (HttpRequestException ex)
    {
        throw new PushException($"Network error: {ex.Message}", ex);
    }

    if (!response.IsSuccessStatusCode)
    {
        var body = await response.Content.ReadAsStringAsync();
        throw new PushException(
            $"GitHub API error {(int)response.StatusCode}: {response.ReasonPhrase}\n{body}");
    }

    return await response.Content.ReadAsStringAsync();
}
```

---

## Exercises

**Exercise 1 — GET the GitHub user**
Set `GITHUB_TOKEN` in your environment. Call `GET https://api.github.com/user`.
Parse the response and print your login and name.

**Exercise 2 — GET a repository**
Call `GET https://api.github.com/repos/{owner}/{repo}` for a public repo.
Parse and print the star count, fork count, and default branch.

**Exercise 3 — Create a blob**
Create a test repository on GitHub. Using the API:
1. Create a blob for the text `"Hello from Mini-Git!\n"`
2. Print the SHA returned by GitHub
3. Verify it appears at `GET /repos/{owner}/{repo}/git/blobs/{sha}`

**Exercise 4 — Full four-step push**
In the test repository from Exercise 3:
1. Create a blob for a text file
2. Create a tree containing that blob
3. Create a commit (parent = current branch HEAD)
4. Update the branch ref to the new commit hash
5. Reload the page on GitHub and verify the commit appears

**Exercise 5 — Error handling**
Deliberately send a bad request (wrong token, or wrong repo name).
Catch the error, print a useful message, and handle the 401/404 status codes separately.
