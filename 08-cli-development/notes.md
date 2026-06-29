# CLI Development with System.CommandLine

## Learning Resources

**Official Docs**
- [System.CommandLine overview — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/commandline/)
- [Get started with System.CommandLine — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/commandline/get-started-tutorial)
- [How to define commands — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/standard/commandline/define-commands)
- [.NET Global Tools — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/tools/global-tools)

**GitHub**
- [dotnet/command-line-api — GitHub](https://github.com/dotnet/command-line-api) — source, issues, samples, and release notes.

**Books**
- *Pro .NET 6* by Adam Freeman — Chapter 17 covers building CLI tools including System.CommandLine.

---

## 1. Why System.CommandLine?

Alternatives:
- **Manual parsing** (`args[0]`, `args[1]`, etc.) — brittle, no help text, no validation
- **CommandLineParser** (NuGet) — popular but not an official .NET library
- **System.CommandLine** — the modern, official .NET approach. Handles: help text generation, tab completion, argument parsing, async handlers, option aliases

For Mini-Git, `System.CommandLine` is the right choice.

---

## 2. Installation

```xml
<!-- In MiniGit.Cli.csproj -->
<PackageReference Include="System.CommandLine" Version="2.0.0-beta4.22272.1" />
```

Or:
```powershell
dotnet add package System.CommandLine --prerelease
```

---

## 3. Core Concepts

| Concept | Description | Example |
|---|---|---|
| `RootCommand` | The top-level command (the program itself) | `minigit` |
| `Command` | A named subcommand | `minigit commit` |
| `Argument<T>` | A positional parameter | `minigit hash-object file.txt` |
| `Option<T>` | A named parameter with `--` prefix | `minigit commit --message "fix"` |
| Handler | The code that runs when the command is invoked | Lambda or method |

---

## 4. Building the Root Command

```csharp
// Program.cs
using System.CommandLine;

var rootCommand = new RootCommand("Mini-Git — a simplified version control system");

// Subcommands will be added here
rootCommand.AddCommand(BuildInitCommand());
rootCommand.AddCommand(BuildHashObjectCommand());
rootCommand.AddCommand(BuildCatFileCommand());
rootCommand.AddCommand(BuildAddCommand());
rootCommand.AddCommand(BuildCommitCommand());
rootCommand.AddCommand(BuildLogCommand());
rootCommand.AddCommand(BuildStatusCommand());
rootCommand.AddCommand(BuildBranchCommand());
rootCommand.AddCommand(BuildCheckoutCommand());
rootCommand.AddCommand(BuildDiffCommand());

return await rootCommand.InvokeAsync(args);
```

---

## 5. Defining a Command

### `init` — no arguments

```csharp
Command BuildInitCommand()
{
    var cmd = new Command("init", "Initialize a new Mini-Git repository");

    cmd.SetHandler(() =>
    {
        var init = new InitCommand();
        init.Execute(Directory.GetCurrentDirectory());
        Console.WriteLine("Initialized empty Mini-Git repository in .minigit/");
    });

    return cmd;
}
```

### `hash-object <file>` — positional argument

```csharp
Command BuildHashObjectCommand()
{
    var fileArg = new Argument<FileInfo>("file", "Path to the file to hash and store")
    {
        Arity = ArgumentArity.ExactlyOne
    };

    var cmd = new Command("hash-object", "Store a file as a blob and print its hash")
    {
        fileArg
    };

    cmd.SetHandler((FileInfo file) =>
    {
        if (!file.Exists)
        {
            Console.Error.WriteLine($"error: cannot stat '{file.Name}': No such file or directory");
            return;
        }

        var store   = BuildObjectStore();
        var content = File.ReadAllBytes(file.FullName);
        var hash    = store.Store(content);
        Console.WriteLine(hash);
    }, fileArg);

    return cmd;
}
```

### `commit -m "message"` — option with alias

```csharp
Command BuildCommitCommand()
{
    var messageOpt = new Option<string>("--message", "Commit message")
    {
        IsRequired = true
    };
    messageOpt.AddAlias("-m");

    var cmd = new Command("commit", "Record changes to the repository")
    {
        messageOpt
    };

    cmd.SetHandler(async (string message) =>
    {
        var (store, index, refs) = BuildDependencies();
        var command = new CommitCommand(store, index, refs);
        try
        {
            var hash = command.Execute(GetAuthor(), message);
            Console.WriteLine($"[main {hash[..7]}] {message}");
        }
        catch (InvalidOperationException ex)
        {
            Console.Error.WriteLine(ex.Message);
        }
    }, messageOpt);

    return cmd;
}
```

### `cat-file <hash>` — positional argument

```csharp
Command BuildCatFileCommand()
{
    var hashArg = new Argument<string>("hash", "Object hash to display");
    var typeFlag = new Option<bool>("-t", "Show the object type instead of content");

    var cmd = new Command("cat-file", "Display the contents of an object")
    {
        hashArg,
        typeFlag
    };

    cmd.SetHandler((string hash, bool showType) =>
    {
        var store = BuildObjectStore();
        if (!store.Exists(hash))
        {
            Console.Error.WriteLine($"fatal: Not a valid object name '{hash}'");
            return;
        }
        var content = store.Retrieve(hash);
        if (showType)
            Console.WriteLine(DetectType(content));
        else
            Console.Write(Encoding.UTF8.GetString(content));
    }, hashArg, typeFlag);

    return cmd;
}
```

---

## 6. Argument Types

`System.CommandLine` supports typed arguments — it parses the string automatically:

```csharp
new Argument<int>("count")           // parses as integer
new Argument<FileInfo>("file")       // creates FileInfo from path string
new Argument<DirectoryInfo>("dir")   // creates DirectoryInfo
new Argument<string[]>("files")      // multiple values
    { Arity = ArgumentArity.OneOrMore }
```

For `FileInfo` and `DirectoryInfo`, the library creates the object from the path string. The file does not need to exist unless you check `file.Exists` yourself.

---

## 7. Option Arity and Defaults

```csharp
// Optional option with a default value
var verboseOpt = new Option<bool>("--verbose", () => false, "Enable verbose output");
verboseOpt.AddAlias("-v");

// Required option (no default)
var messageOpt = new Option<string>("--message", "Commit message") { IsRequired = true };

// Option that can appear multiple times
var pathsOpt = new Option<string[]>("--path")
{
    AllowMultipleArgumentsPerToken = true,
    Arity = ArgumentArity.ZeroOrMore
};
```

---

## 8. Error Handling in Handlers

```csharp
cmd.SetHandler((string message) =>
{
    try
    {
        var hash = command.Execute(message);
        Console.WriteLine($"[main {hash[..7]}] {message}");
    }
    catch (NotARepositoryException ex)
    {
        Console.Error.WriteLine(ex.Message);
        // System.CommandLine uses the exit code from InvokeAsync
        // To set a non-zero exit code, use Environment.Exit(1)
        Environment.Exit(1);
    }
}, messageOpt);
```

---

## 9. .NET Global Tools

A global tool is a .NET console app that can be installed system-wide and run by name, like `git` or `dotnet`.

### Configure the .csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <PackAsTool>true</PackAsTool>
    <ToolCommandName>minigit</ToolCommandName>
    <PackageId>MiniGit.Cli</PackageId>
    <Version>1.0.0</Version>
  </PropertyGroup>
</Project>
```

### Install locally during development

```powershell
# Pack the project into a NuGet package
dotnet pack src/MiniGit.Cli -o ./nupkg

# Install from the local package directory
dotnet tool install --global --add-source ./nupkg MiniGit.Cli

# Verify it works
minigit --help

# Uninstall when you want to reinstall a newer version
dotnet tool uninstall --global MiniGit.Cli
```

### During active development (without reinstalling each time)

```powershell
# Just run directly without installing
dotnet run --project src/MiniGit.Cli -- init
dotnet run --project src/MiniGit.Cli -- hash-object file.txt
```

---

## 10. `CommandRouter.cs` Pattern

Separate the command registration from `Program.cs` to keep it clean:

```csharp
// CommandRouter.cs
public static class CommandRouter
{
    public static RootCommand Build(string repoRoot)
    {
        var root = new RootCommand("Mini-Git — a simplified version control system");

        root.AddCommand(InitCommandFactory.Build());
        root.AddCommand(HashObjectCommandFactory.Build(repoRoot));
        root.AddCommand(CommitCommandFactory.Build(repoRoot));
        // ...

        return root;
    }
}

// Program.cs
var repoRoot = Directory.GetCurrentDirectory();
var root = CommandRouter.Build(repoRoot);
return await root.InvokeAsync(args);
```

---

## Exercises

**Exercise 1 — Hello CLI**
Build a `greet` tool:
- `greet --name Alice` prints `Hello, Alice!`
- `greet --name Alice --shout` prints `HELLO, ALICE!`
- Running with no args prints a helpful error (System.CommandLine handles this automatically if `IsRequired = true`)

**Exercise 2 — File hasher**
Build `hashfile <path>` using `Argument<FileInfo>`.
If the file does not exist, print an error. If it does, print its SHA-256 hash.
Test with a known file and verify against `certutil -hashfile file SHA256`.

**Exercise 3 — minigit init**
Implement the full `init` command:
- Creates `.minigit/objects/`, `.minigit/refs/heads/`, `.minigit/refs/tags/`
- Writes `.minigit/HEAD` with content `ref: refs/heads/main`
- Prints confirmation message
- Warns (does not fail) if `.minigit` already exists

**Exercise 4 — Install as a global tool**
Pack your CLI project and install it globally.
Run `minigit --help` from a different directory and verify it works.
Run `minigit init`, then inspect the `.minigit` directory that was created.
