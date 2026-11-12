---
name: powershell-windows
description: PowerShell 7 (pwsh) command syntax on Windows. Load when writing or debugging shell commands, translating bash/POSIX commands, or handling environment variables, paths, quoting, here-strings, pipelines, exit codes, and process control on Windows.
---

# PowerShell 7 on Windows

The shell tool is called `bash` but executes `pwsh`. Bash syntax does not survive.

## Why commands fail here

PowerShell defines aliases so short Unix names *appear* to work:

| Alias | Resolves to |
|---|---|
| `ls` | `Get-ChildItem` |
| `cat` | `Get-Content` |
| `rm` | `Remove-Item` |
| `cp` | `Copy-Item` |
| `mv` | `Move-Item` |
| `mkdir` | `New-Item` |
| `ps` | `Get-Process` |
| `kill` | `Stop-Process` |
| `curl` | removed in PS 6+; `curl.exe` is real curl |

The name works, then the POSIX flag does not: `ls -la` errors with
`A parameter cannot be found that matches parameter name 'la'`. That error is the signal to switch
to cmdlet syntax, not to retry the command differently.

## Translation table

| POSIX | PowerShell 7 |
|---|---|
| `ls -la` | `Get-ChildItem -Force` |
| `find . -name "*.rs"` | `Get-ChildItem -Recurse -Filter *.rs` |
| `cat f` / `head -n 20 f` / `tail -f f` | `Get-Content f` / `Get-Content f -TotalCount 20` / `Get-Content f -Wait -Tail 10` |
| `grep -r foo .` | `Get-ChildItem -Recurse -File \| Select-String foo` |
| `sed 's/a/b/g'` | `(Get-Content f -Raw) -replace 'a','b'` (regex) |
| `touch f` | `New-Item -ItemType File -Path f` |
| `mkdir -p a/b` | `New-Item -ItemType Directory -Force -Path a/b` |
| `rm -rf d` | `Remove-Item -LiteralPath d -Recurse -Force` |
| `cp -r a b` | `Copy-Item -LiteralPath a -Destination b -Recurse` |
| `export A=b` | `$env:A = "b"` |
| `echo $A` | `$env:A` |
| `A=b cmd` | `$env:A = "b"; cmd` |
| `which x` | `Get-Command x` |
| `xargs` / `awk '{print $1}'` | `ForEach-Object { ... }`, `Select-Object -ExpandProperty Name` |
| `wc -l` | `(Get-Content f \| Measure-Object -Line).Lines` |
| `du -sh d` | `(Get-ChildItem d -Recurse -File \| Measure-Object Length -Sum).Sum` |
| `time cmd` | `Measure-Command { cmd }` |
| `cmd1 && cmd2` | `cmd1; if ($?) { cmd2 }` or `cmd1 && cmd2` (PS 7 only) |
| `cmd &` / `nohup` | Use the background process tool, not shell backgrounding |
| `ln -s a b` | `New-Item -ItemType SymbolicLink -Path b -Target a` (needs developer mode) |
| `chmod` | Not applicable |
| `set -e` | `$ErrorActionPreference = 'Stop'` |

## Exit codes and error handling

- Native executables set `$LASTEXITCODE` (an integer).
- Cmdlets set `$?` (a boolean) and emit `ErrorRecord` objects instead of exit codes.
- `$LASTEXITCODE` is **only meaningful immediately after a native executable**. Cmdlets never reset
  it, so after a cmdlet it still holds the last native value — often a stale failure. Never branch
  on it after a cmdlet.
- Non-zero native exit does **not** throw by default. To fail fast:

```powershell
cargo build --workspace
if ($LASTEXITCODE -ne 0) { throw "cargo build failed" }
```

- Prefer `-ErrorAction Stop` on cmdlets whose failure must abort a script.

```powershell
cmd /c exit 3        # native
$LASTEXITCODE        # 3
Get-ChildItem > $null # cmdlet
$LASTEXITCODE        # still 3 — stale
$?                   # True — this is the cmdlet result
```

## Reading cmdlet errors (CLIXML)

When captured by a non-console host, cmdlet errors can arrive serialized as CLIXML instead of text:

```xml
#< CLIXML
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <S S="Error">_x001B_[31;1mGet-ChildItem: _x001B_[31;1mA parameter cannot be found that matches parameter name 'la'._x001B_[0m_x000D__x000A_</S>
</Objs>
```

This is not an opaque failure — it contains the real message. Decode it:

- `_x001B_[...m` are ANSI color escapes; ignore them.
- `_x000D__x000A_` is CRLF.
- The text inside `<S S="Error">` is the actual error, e.g.
  `A parameter cannot be found that matches parameter name 'la'`.

To avoid CLIXML noise entirely, capture the message directly:

```powershell
try {
  Get-ChildItem -Force -ErrorAction Stop
} catch {
  "ERROR: $($_.Exception.Message)"
}
```

## Quoting

- Single quotes are literal: `'$env:PATH'` stays as written.
- Double quotes interpolate: `"$env:PATH"`.
- Backtick `` ` `` is the escape character. `\` is not special, so `C:\Users\me` needs no escaping.
- There are no heredocs. Use a here-string; the closing `'@` must sit at column 0:

```powershell
$body = @'
line one
line two
'@
```

- When a native tool mangles quotes, pass an argument array or use the stop-parsing token `--%`.
  `--flag=value` avoids most quoting bugs.

## Pipelines

- Objects, not text. `Get-ChildItem | Where-Object Length -gt 1MB` compares real numbers.
- `Select-Object -First 3`, `-Last 3`, `-Skip 2` replace `head`/`tail`.
- `ForEach-Object { ... }` replaces `xargs`; inside it use `$_`.
- `Sort-Object`, `Group-Object`, `Measure-Object` replace the usual Unix pipeline stages.

## Common failures and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `A parameter cannot be found that matches parameter name 'la'` | POSIX flag on a cmdlet alias | Use cmdlet parameters, e.g. `-Force` |
| `The term 'export' is not recognized` | bash builtin | `$env:NAME = "value"` |
| `The token '&&' is not a valid statement separator` | PS 5.1 | Use `;` and `$?`, or confirm you are on PS 7 |
| `Unexpected token 'EOF'` | bash heredoc | Use `@' ... '@` here-string |
| Empty result from `$VAR` | bash variable syntax | `$env:VAR` for environment, `$var` for locals |
| Path not found with `/c/...` | MSYS path | Use `C:\...` or `C:/...` |
| Quoted argument arrives split | native quoting | Pass an array, or use `--%` / `--flag=value` |
| `#< CLIXML` with `_x001B_[31;1m` | cmdlet error serialized for a non-console host | Read the `<S S="Error">` text; strip ANSI and `_x000D__x000A_` |
| Native command "succeeds" but did nothing | native non-zero exit ignored | Check `$LASTEXITCODE` right after it |
| Branch on `$LASTEXITCODE` is wrong | value stale from an earlier native command | Use `$?` after cmdlets |

## Before running a shell command

1. Decide whether a dedicated tool does the job: Read, Grep, Glob, Edit beat `Get-Content`,
   `Select-String`, and `Set-Content`.
2. Translate any POSIX flags to cmdlet parameters.
3. Read the actual error text on failure. Never swap to another shell dialect and retry blindly.