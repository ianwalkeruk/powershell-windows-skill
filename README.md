# powershell-windows-skill

An Agent Skill that teaches LLM coding agents how to write PowerShell 7 commands on Windows correctly the first time.

Coding agents often expose a shell tool named `bash` that actually runs `pwsh`. Bash syntax does not survive that swap, and PowerShell's Unix-style aliases (`ls`, `cat`, `rm`, `cp`, `mv`) hide the mismatch: the command name resolves, then the POSIX flag fails with `A parameter cannot be found that matches parameter name 'la'`. The agent retries blindly, or switches dialects, and the turn is wasted.

This skill gives the agent the translation table, the error signatures, and the recovery rules it needs to get it right immediately.

## What the skill covers

- **Why commands fail** - PowerShell aliases and the errors they produce.
- **POSIX to PowerShell translation** - `ls`, `find`, `grep`, `sed`, `xargs`, `export`, `du`, `time`, `&&`, symlinks, and more.
- **Exit codes** - when `$LASTEXITCODE` is meaningful, why it goes stale after a cmdlet, and when to branch on `$?` instead.
- **CLIXML errors** - decoding serialized cmdlet errors instead of treating `#< CLIXML` as an opaque failure.
- **Quoting and here-strings** - single vs double quotes, the backtick escape, and why heredocs do not work.
- **Pipelines** - objects rather than text, and the cmdlets that replace `head`, `tail`, `xargs`, `sort`, and `wc`.
- **A symptom/cause/fix table** for the failures agents hit most often.
- **A pre-flight checklist** before running any shell command, including when a dedicated Read/Grep/Glob/Edit tool is the better choice.

The skill is guidance only. It contains no scripts and requires no runtime beyond PowerShell 7.

## Requirements

- PowerShell 7 (`pwsh`) on Windows. Windows PowerShell 5.1 differs in places, most notably `&&`.
- An agent that supports Agent Skills, or a way to add the skill text to that agent's instructions (see [Other tools](#other-tools)).

## Installation

The skill must live in a directory named `powershell-windows` with `SKILL.md` directly inside it. Keep that layout: most tools require the directory name to match the `name` field in the frontmatter.

Clone the repository once, then copy the skill directory into your tool's skills location:

```powershell
git clone https://github.com/ianwalkeruk/powershell-windows-skill.git "$HOME\src\powershell-windows-skill"
$skill = "$HOME\src\powershell-windows-skill\skill\powershell-windows"
```

### Where each tool looks

| Tool | Project | Personal |
|---|---|---|
| Kilo Code | `.kilo/skills/powershell-windows/` | `~/.config/kilo/skills/powershell-windows/` |
| Claude Code | `.claude/skills/powershell-windows/` | `~/.claude/skills/powershell-windows/` |
| Codex CLI | `.agents/skills/powershell-windows/` | `~/.agents/skills/powershell-windows/` |
| Cursor | `.cursor/skills/powershell-windows/` or `.agents/skills/powershell-windows/` | `~/.cursor/skills/powershell-windows/` or `~/.agents/skills/powershell-windows/` |
| GitHub Copilot (VS Code) | `.github/skills/powershell-windows/` or `.agents/skills/powershell-windows/` | `~/.copilot/skills/powershell-windows/` or `~/.agents/skills/powershell-windows/` |
| Gemini CLI | `.gemini/skills/powershell-windows/` or `.agents/skills/powershell-windows/` | `~/.gemini/skills/powershell-windows/` or `~/.agents/skills/powershell-windows/` |

A single copy in `.agents/skills/` is picked up by Codex CLI, Cursor, GitHub Copilot, and Gemini CLI. A copy in `.claude/skills/` is also read by Claude Code, Cursor, and Copilot. Kilo Code uses its own `.kilo/skills/` directory.

`~` in the table means your home directory (`$HOME` in PowerShell). The commands below assume the variables from the clone step.

### Kilo Code

Project-level install:

```powershell
$dest = ".kilo\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item -LiteralPath $skill -Destination $dest -Recurse -Force
```

For every project on the machine, set `$dest` to `"$HOME\.config\kilo\skills"` instead. Kilo Code also loads skills from the legacy `.kilocode/` directory and from any directories listed in `skills.paths` in `kilo.json`.

### Claude Code

```powershell
$dest = "$HOME\.claude\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item -LiteralPath $skill -Destination $dest -Recurse -Force
```

Claude Code picks up new skills without a restart. If the `skills` directory did not exist when the session started, run `/reload-skills`. The skill is available as `/powershell-windows` and is also selected automatically when a task matches its description.

### Codex CLI

```powershell
$dest = "$HOME\.agents\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item -LiteralPath $skill -Destination $dest -Recurse -Force
```

Codex discovers skills from `.agents/skills/` in the repository and in `$HOME`. Type `$` in a prompt to pick the skill explicitly, or run `/skills` to confirm discovery. Restart Codex if a change does not appear.

### Cursor

```powershell
$dest = ".cursor\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item -LiteralPath $skill -Destination $dest -Recurse -Force
```

For a personal install, set `$dest` to `"$HOME\.cursor\skills"` instead. Cursor also reads `.agents/skills/`, `.claude/skills/`, and `.codex/skills/`. Verify under **Customize > Skills**, then type `/powershell-windows` in Agent chat or let the description trigger it.

### GitHub Copilot in VS Code

```powershell
$dest = ".github\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item -LiteralPath $skill -Destination $dest -Recurse -Force
```

For a personal install, set `$dest` to `"$HOME\.copilot\skills"` instead. Copilot also reads `.agents/skills/` and `.claude/skills/`. The skill appears in the chat `/` menu, and Copilot can load it automatically based on the description.

### Gemini CLI

```powershell
$dest = ".gemini\skills"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Copy-Item -LiteralPath $skill -Destination $dest -Recurse -Force
```

For a personal install, set `$dest` to `"$HOME\.gemini\skills"` instead. Run `/skills list` to confirm discovery and `/skills reload` after adding files. Gemini CLI discovers `SKILL.md` at the skills root or exactly one directory deep, so do not nest the skill further.

### Other tools

Any agent that supports the Agent Skills standard can use the folders above. Tools without skill support can still benefit by adding the contents of `skill/powershell-windows/SKILL.md` to their always-on instructions, for example `AGENTS.md`, `.github/copilot-instructions.md`, `.cursor/rules/`, `.clinerules`, or `.windsurfrules`.

## Verifying it works

Start a new session and ask the agent to list the current directory including hidden files. With the skill loaded, it should reach for `Get-ChildItem -Force` rather than `ls -la`, and it should not retry the POSIX form after an error.

## Updating

Pull the repository and copy the directory again, overwriting the installed copy:

```powershell
git -C "$HOME\src\powershell-windows-skill" pull
$dest = "$HOME\.claude\skills"
Remove-Item -LiteralPath "$dest\powershell-windows" -Recurse -Force -ErrorAction SilentlyContinue
Copy-Item -LiteralPath "$HOME\src\powershell-windows-skill\skill\powershell-windows" -Destination $dest -Recurse -Force
```

A symbolic link to the repository's `skill\powershell-windows` directory avoids the copy step if your agent follows links.

## License

MIT. See [LICENSE](LICENSE).
