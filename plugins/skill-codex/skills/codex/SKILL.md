---
name: codex
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing
---

# Codex Skill Guide

## Model and reasoning policy

These are this skill's operating limits, not a claim about every effort the underlying model supports. Apply them to new sessions, resumes, configuration defaults, and model changes.

| Model | Permitted reasoning efforts | Default effort | Guidance |
| --- | --- | --- | --- |
| `gpt-6-astra` | `low`, `medium` | `medium` | Explicit model selection; never escalate above Medium |
| `gpt-5.6-sol` | `medium`, `high`; exceptional `xhigh` | `high` | Default GPT-5.6 option |
| `gpt-5.6-terra` | `medium`, `high`, `xhigh` | `high` | Balanced everyday option |
| `gpt-5.6-luna` | `medium`, `high`, `xhigh`, `max` | `high` | Fast and affordable option |

- Overall default: `gpt-5.6-sol` at `high`. Adding Astra does not change that default.
- Never launch or resume with `model_reasoning_effort="ultra"`, including an inherited setting. Max is permitted only for Luna among the primary models.
- Sol XHigh is exceptional. Before using it, state a concrete reason why High is insufficient, such as an unresolved correctness problem after a substantive High attempt or a difficult architectural conflict. Task size alone is insufficient. An explicit request still needs task-specific justification; ask for missing context only if the task does not supply it.
- Honor a model/effort already specified by the user. For missing choices, use `AskUserQuestion` to select the model first, then offer only its permitted efforts; do not ask again for choices already supplied. If the user has no preference or asks you to choose, use the defaults.
- For an explicitly requested disallowed pair, report the mismatch and propose that model's default; obtain the user's replacement choice unless they already authorized you to choose. Never silently clamp to the model's technical maximum or substitute another model.
- Legacy compatibility remains available on explicit request: `gpt-5.5`, `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex-spark`, `gpt-5.3-codex`. Offer only efforts supported by that model and installed CLI, with a skill ceiling of `xhigh`; default to `high` only if supported. Do not assume every legacy model supports the same efforts.
- Check the installed CLI/account's available models when availability is uncertain. If a requested model is unavailable, report it instead of changing the model without agreement.

## Running a Task

1. Identify the execution tool's actual shell and workspace. Windows and the VS Code terminal profile do not by themselves tell you which shell Claude's tool uses. For native PowerShell, read [Windows PowerShell execution](references/windows-powershell.md). For Git Bash or WSL, use Bash syntax and paths appropriate to that environment; keep the Codex executable, authentication context, and repository in the intended environment.
2. Verify `codex --version` from that execution environment. Resolve the model and reasoning effort using the policy above. Check `codex exec --help` and `codex exec resume --help` if the installed version's flag placement or capabilities are uncertain.
3. Select the sandbox required by the task; default to `--sandbox read-only`. Use `workspace-write` for authorized local edits. Broad access requires existing authorization; do not change sandbox or execution policy just to fix shell syntax, PATH, authentication, or network errors.
4. Assemble arguments separately from prompt text:
   - `-m, --model <MODEL>` and `-c, --config <KEY=VALUE>`, with an explicit policy-compliant `model_reasoning_effort`.
   - `--sandbox <read-only|workspace-write|danger-full-access>`.
   - `-C, --cd <DIR>` for the intended workspace; on resume, set the process working directory first if that subcommand does not accept `-C`.
   - Always include `--skip-git-repo-check`, subject to the existing permission rule below.
   - Use `--full-auto` only if the installed CLI/workflow requires it and it is authorized; current CLI documentation deprecates it. Do not add it to read-only examples.
   - Prefer `-` as the prompt argument and send the complete prompt through stdin. End input with EOF; never send the prompt both positionally and through stdin.
5. For a continuation, use the resume procedure below before assembling the command.
6. Capture stdout and stderr separately. Keep routine progress out of the conversation, but retain stderr in a task-specific temporary log and inspect it on failure or an empty result. Do not treat stderr as only thinking tokens. If suppressing it intentionally, use the shell's syntax: Bash `2>/dev/null`, PowerShell `2>$null`.
7. Run the command, record the exit status immediately, and summarize the result and any material warnings. If using a positional prompt in a Bash harness, close unused stdin with `</dev/null`; PowerShell does not support that syntax. Piping prompt text is the preferred approach for either shell.
8. After completion, tell the user they can resume the session by saying "codex resume" or asking for further analysis or changes. Retain the session ID when available.

### Bash example

The single-quoted heredoc preserves literal prompt content. Choose a delimiter absent from the prompt. The temporary stderr file is outside the repository; inspect it on failure before cleaning it up.

```bash
codex_log=$(mktemp)
codex_args=(exec -m gpt-5.6-sol
  -c "model_reasoning_effort='high'"
  --sandbox read-only --skip-git-repo-check
  -C "/path/to/project")
codex "${codex_args[@]}" - 2>"$codex_log" <<'CODEX_PROMPT'
Analyze this repository and summarize the main correctness risks.
CODEX_PROMPT
codex_status=$?
if [ "$codex_status" -ne 0 ]; then
  cat "$codex_log" >&2
fi
# Retain codex_status and inspect the log before any retry.
```

### Quick Reference

| Use case | Execution guidance |
| --- | --- |
| Read-only review | Explicit permitted model/effort and `--sandbox read-only` |
| Authorized local edits | Explicit permitted model/effort and `--sandbox workspace-write` |
| Authorized broad access | `--sandbox danger-full-access`; do not use as a shell troubleshooting shortcut |
| Resume | Validate the effective model/effort, target the intended session, then pipe a follow-up prompt |
| PowerShell / VS Code on Windows | Read [Windows PowerShell execution](references/windows-powershell.md) |
| Another directory | Set the correct working directory; quote paths and preserve environment-specific path syntax |

## Resuming and following up

1. Identify the intended session and workspace. Prefer an explicit session ID when multiple workspaces or sessions are in use. Use `--last` only when the latest session in the intended working directory is the correct one; do not broaden selection with `--all` implicitly.
2. Determine the effective model and effort from recorded session settings and any requested changes. Preserve a permitted pair. Revalidate effort whenever the model changes: Astra cannot inherit Sol High.
3. For a disallowed inherited effort, announce replacement with that model's default (Sol XHigh also needs current task justification). If the user explicitly requests the disallowed setting, follow the mismatch rule in the policy above. If the inherited model is unknown, announce and use Sol High; if only the effort is unknown, use the known model's permitted default. Explicitly requested unavailable or unlisted models require resolution, not silent substitution.
4. Pass the resolved model and effort explicitly on resume so local configuration or session inheritance cannot bypass the policy. Preserve the authorized sandbox; do not broaden access. Use flags in the positions accepted by the installed `codex exec resume --help`. If that version cannot apply the necessary overrides, do not run a noncompliant resume; explain and offer a new compliant session with a concise handoff.
5. Pipe the follow-up through stdin using `-`. For example, after selecting Sol High and the intended working directory in Bash:
   ```bash
   codex_log=$(mktemp)
   printf '%s\n' 'Continue the analysis and investigate the remaining risks.' |
     codex exec --skip-git-repo-check resume --last \
       -m gpt-5.6-sol -c "model_reasoning_effort='high'" - 2>"$codex_log"
   codex_status=$?
   ```
   Replace `--last` with the known session ID when appropriate. See the PowerShell reference for equivalent input and argument handling.
6. Restate the selected model, effort, and sandbox when proposing further actions. Continue already authorized work; use `AskUserQuestion` when a genuine next-step decision or clarification is needed.

## Execution timeouts

Codex streams progress to stderr and normally writes the final answer to stdout; `--json` provides structured events. An empty final-output file alone does not establish a hang or failure. Check process state, stderr, and exit status.

Prefer foreground execution when the host can wait long enough. Foreground execution still has the host tool's timeout. For background execution, keep a process/session handle, capture output, poll for completion, and avoid launching a duplicate while the original is running.

These are initial host timeout budgets, not model latency guarantees:

| Reasoning effort | Initial timeout budget |
| --- | --- |
| `low` | 150s |
| `medium` | 300s |
| `high` | 600s |
| `xhigh` | 1200s |
| `max` | 1800s |

If the host yields before completion, resume monitoring the same process. On cancellation or timeout, establish whether the child is still running and terminate only that task's process tree when cancellation is intended. Preserve partial output and report the interruption.

## Critical Evaluation of Codex Output

Codex is powered by OpenAI models with their own knowledge cutoffs and limitations. Treat Codex as a **colleague, not an authority**.

### Guidelines
- **Trust your own knowledge** when confident. If Codex claims something you know is incorrect, push back directly.
- **Research disagreements** using WebSearch or documentation before accepting Codex's claims. Share findings with Codex via resume if needed.
- **Remember knowledge cutoffs** - Codex may not know about recent releases, APIs, or changes that occurred after its training data.
- **Don't defer blindly** - Codex can be wrong. Evaluate its suggestions critically, especially regarding:
  - Model names and capabilities
  - Recent library versions or API changes
  - Best practices that may have evolved

### When Codex is Wrong
1. State your disagreement clearly to the user
2. Provide evidence (your own knowledge, web search, docs)
3. Optionally resume the Codex session to discuss the disagreement. **Identify yourself as Claude** so Codex knows it's a peer AI discussion. Use your actual model name (e.g., the model you are currently running as) instead of a hardcoded name:
   Add this as prompt text using the selected shell's stdin method, after applying the resume policy:
   > This is Claude (<your current model name>) following up. I disagree with [X] because [evidence]. What's your take on this?
4. Frame disagreements as discussions, not corrections - either AI could be wrong
5. Let the user decide how to proceed if there's genuine ambiguity

## Error Handling

- Stop and report failures whenever `codex --version` or `codex exec` exits non-zero; include the exit status and relevant captured diagnostics, and request direction before retrying unless that recovery was already authorized.
- Before using high-impact flags (`--full-auto`, `--sandbox danger-full-access`, `--skip-git-repo-check`), ask for permission using `AskUserQuestion` unless it was already given.
- Summarize warnings or partial results; ask how to adjust when they leave a material decision unresolved.
- On Windows, diagnose launcher, shell, PATH, encoding, and authentication context using the PowerShell reference. Do not disable execution policy or elevate as an automatic workaround.

Sources: [Codex non-interactive mode](https://developers.openai.com/codex/noninteractive), [CLI reference](https://developers.openai.com/codex/cli/reference).
