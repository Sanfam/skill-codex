Note: If you want a more autonomous setup for agentic workflows, check out [klaudworks/ralph-meets-rex](https://github.com/klaudworks/ralph-meets-rex).

# Codex Integration for Claude Code

<img width="2288" height="808" alt="skillcodex" src="https://github.com/user-attachments/assets/85336a9f-4680-479e-b3fe-d6a68cadc051" />


## Purpose
Enable Claude Code to invoke the Codex CLI (`codex exec` and session resumes) for automated code analysis, refactoring, and editing workflows.

## Prerequisites

- Codex CLI installed, authenticated, and available to the shell Claude actually uses.
- Verify `codex --version` from that execution environment.
- Supported invocation guidance covers Bash (including Git Bash/WSL where appropriate), PowerShell 7, and Windows PowerShell 5.1. See the [Windows reference](plugins/skill-codex/skills/codex/references/windows-powershell.md) for compatibility checks; instructions alone do not certify every launcher/version.

## Installation

This fork is packaged as a Claude Code plugin with a marketplace. Install from **Sanfam/skill-codex** to receive this fork's model policy and Windows guidance.

### Plugin installation

```text
/plugin marketplace add Sanfam/skill-codex
/plugin install skill-codex@skill-codex
```

If a marketplace named `skill-codex` already points at the upstream repository, update its source to this fork through Claude's marketplace management before installing. Do not assume adding an identically named marketplace replaced the existing source.

### Standalone skill installation

Clone the fork to a new local directory, then copy the entire `codex` folder, including `references`. If a destination skill already exists, review/back it up before replacing it.

Bash:

```bash
git clone --depth 1 https://github.com/Sanfam/skill-codex.git skill-codex
mkdir -p ~/.claude/skills
cp -R skill-codex/plugins/skill-codex/skills/codex ~/.claude/skills/
```

PowerShell:

```powershell
git clone --depth 1 https://github.com/Sanfam/skill-codex.git skill-codex
if ($LASTEXITCODE -ne 0) { throw 'Clone failed.' }
$skillDestination = Join-Path $env:USERPROFILE '.claude\skills'
New-Item -ItemType Directory -Path $skillDestination -Force | Out-Null
Copy-Item -LiteralPath '.\skill-codex\plugins\skill-codex\skills\codex' -Destination $skillDestination -Recurse
```

## Model and reasoning policy

| Model | Permitted efforts | Default |
| --- | --- | --- |
| `gpt-6-astra` | Low, Medium | Medium |
| `gpt-5.6-sol` | Medium, High; exceptional XHigh | High |
| `gpt-5.6-terra` | Medium, High, XHigh | High |
| `gpt-5.6-luna` | Medium, High, XHigh, Max | High |

Overall default remains **Sol High**. These are this skill's operating limits, not the models' full technical capabilities. Sol XHigh requires a concrete task-specific justification; size alone is insufficient. Ultra is excluded from new sessions and resumes. Max is available only for Luna among these four models.

Legacy models remain explicit compatibility options: `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex-spark`, and `gpt-5.3-codex`. Their efforts must be checked against installed model support and may not exceed XHigh.

Claude honors choices supplied in the prompt, asks only for missing choices when needed, and offers model-specific effort options. A disallowed explicit pair is reported with a proposed permitted replacement; it is not silently clamped. Resumes validate and explicitly apply the resolved pair so inherited configuration cannot bypass the limits.

## Usage

Example requests:

```text
Use Codex to analyze this repository with GPT-5.6 Sol at High.
Use Codex with GPT-6 Astra at Medium to review the proposed architecture.
Continue the previous Codex session and investigate the remaining risks.
```

Claude selects the actual shell, validates the model/effort, chooses an authorized sandbox (read-only by default), runs Codex, and summarizes the result. Model availability still depends on the installed CLI and account.

### Bash execution

```bash
codex_log=$(mktemp)
codex exec -m gpt-6-astra -c "model_reasoning_effort='medium'" \
  --sandbox read-only --skip-git-repo-check - 2>"$codex_log" <<'CODEX_PROMPT'
Analyze this repository and summarize the main correctness risks.
CODEX_PROMPT
codex_status=$?
if [ "$codex_status" -ne 0 ]; then
  cat "$codex_log" >&2
fi
```

Run from the intended repository or supply `-C`. Retain the exit status and inspect diagnostics before any retry.

### PowerShell and Claude's VS Code extension

Use a PowerShell argument array, literal here-string or UTF-8 prompt file, and a finite stdin pipeline. For longer invocations, use a reviewed `.ps1` launched with `pwsh -NoProfile -NonInteractive -File` (or `powershell.exe` for 5.1). See [complete PowerShell new-session and resume examples](plugins/skill-codex/skills/codex/references/windows-powershell.md).

Detect Claude's actual execution tool: VS Code's terminal profile alone does not establish whether it uses PowerShell, Git Bash, or WSL. Do not paste Bash redirection such as `</dev/null` into PowerShell.

### Output and follow-up

Codex normally streams progress to stderr and returns its final answer on stdout. This skill captures diagnostics separately and keeps routine progress out of the conversation. Failures and partial results are reported; stderr is not discarded unconditionally.

Resumes prefer a known session ID when multiple sessions exist and preserve the authorized sandbox. The skill validates inherited settings, announces necessary replacements, and applies the resulting model/effort explicitly.

See [SKILL.md](plugins/skill-codex/skills/codex/SKILL.md) for the authoritative policy, permissions, timeout guidance, and resume procedure.

## Attribution

Fork of [skills-directory/skill-codex](https://github.com/skills-directory/skill-codex). Original author and MIT license attribution are retained.
