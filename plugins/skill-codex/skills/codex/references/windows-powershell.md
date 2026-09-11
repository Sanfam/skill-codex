# Windows PowerShell execution

Read this reference when Codex is invoked from native Windows PowerShell, including Claude Code's PowerShell tool in the VS Code extension. The model policy in [SKILL.md](../SKILL.md) applies unchanged. These are invocation examples, not a policy-enforcing wrapper: resolve the model, effort, sandbox, and permissions before running them.

## Select the actual execution environment

- Inspect the available shell tool and its runtime. VS Code's integrated-terminal profile does not determine every extension tool's shell.
- Native PowerShell: use the Windows Codex installation and Windows paths. Prefer PowerShell 7 when installed; account for Windows PowerShell 5.1's encoding and native-argument differences.
- Git Bash: use Bash syntax and Git Bash path conventions. WSL: use the Linux installation, credentials, and paths. Do not silently switch into WSL to work around a native-Windows problem.
- From the same tool that will invoke Codex, inspect:
  ```powershell
  $PSVersionTable.PSVersion
  Get-Command codex -All | Select-Object CommandType, Name, Source
  Get-Location
  codex --version
  $codexVersionExit = $LASTEXITCODE
  ```
  Stop on a failed preflight. A working command in a separate terminal does not prove the extension inherited its PATH or authentication context. Restart the extension host after an installation/PATH change if needed.

## Direct PowerShell execution

Use an argument array and the call operator `&`. Send the prompt as data through stdin, with `-` as the Codex prompt argument. A finite pipeline supplies EOF without Bash's unsupported `</dev/null`.

The example uses Astra Medium. Change only to a pair allowed by the core policy. The single-quoted here-string preserves dollar signs, quotes, and backticks literally. Keep its closing marker at the start of its own line. For arbitrary prompt content that contains the closing marker, use a UTF-8 prompt file and `Get-Content -LiteralPath $promptPath -Raw -Encoding UTF8` instead.

```powershell
$codexCommand = Get-Command codex -CommandType Application,ExternalScript -ErrorAction Stop |
    Select-Object -First 1
$codexExecutable = $codexCommand.Source
$codexWorkDir = 'C:\Work\My Project'
$codexPrompt = @'
Analyze this repository and summarize the main correctness risks.
Preserve literal examples such as $env:PATH, "quoted text", and café.
'@
$codexArgs = @(
    'exec'
    '-m', 'gpt-6-astra'
    '-c', "model_reasoning_effort='medium'"
    '--sandbox', 'read-only'
    '--skip-git-repo-check'
    '-C', $codexWorkDir
    '-'
)
$codexLog = [System.IO.Path]::GetTempFileName()
$codexOutput = @()
$codexExit = $null
$codexSavedEncoding = $OutputEncoding
$codexSavedErrorPreference = $ErrorActionPreference

try {
    # PowerShell 5.1 otherwise commonly encodes native stdin as ASCII.
    $OutputEncoding = New-Object System.Text.UTF8Encoding($false)
    # Native stderr may be represented as PowerShell error records.
    # Preserve it without aborting on ordinary Codex progress output.
    $ErrorActionPreference = 'Continue'
    $LASTEXITCODE = $null
    $codexOutput = $codexPrompt | & $codexExecutable @codexArgs 2> $codexLog
    $codexExit = $LASTEXITCODE
}
finally {
    $OutputEncoding = $codexSavedEncoding
    $ErrorActionPreference = $codexSavedErrorPreference
}

$codexOutput
if ($null -eq $codexExit -or $codexExit -ne 0) {
    Get-Content -LiteralPath $codexLog
    throw "Codex failed (exit status: $codexExit). Diagnostics: $codexLog"
}
# If output is empty, inspect the log before treating the run as successful.
# Remove this task's temporary log only after reviewing needed diagnostics.
```

The configuration value uses a TOML literal string: `model_reasoning_effort='medium'`. Its inner single quotes avoid relying on embedded double-quote preservation across Windows launchers. Keep the prompt out of the command line. Do not build an `Invoke-Expression` string or use `--%` to inject user content.

If persisting output, select UTF-8 explicitly (for example, `Set-Content -Encoding UTF8` for text output). PowerShell 5.1 and newer PowerShell versions differ in redirection encoding; do not assume `>` always creates UTF-8. Save a 5.1 script containing non-ASCII literals as UTF-8 with BOM, or read the prompt from a separately decoded UTF-8 file.

## Launcher resolution

`Get-Command codex -All` may reveal a native executable, an npm `.ps1` shim, or an npm `.cmd` shim.

- Prefer the intended installed launcher, and use its resolved path with `&`; never assume `codex.exe` exists.
- If an npm PowerShell shim is blocked by execution policy, an already-installed, permitted `codex.cmd` may be an alternative. Inspect it with `Get-Command codex.cmd -ErrorAction Stop` and use its resolved path. Do not alter execution policy, bypass organizational controls, or elevate.
- PowerShell 5.1 and `.cmd` argument passing use legacy quoting rules. Verify paths with spaces and configuration arguments against the actual launcher. If that launcher still corrupts arguments, use a verified native Codex executable or stop and report the incompatibility.
- Read local `codex exec --help` and `codex exec resume --help` before adapting examples to a different CLI version.

## Run through a script from Claude's VS Code extension

For substantial prompts or repeated calls, put the reviewed invocation in a task-specific `.ps1` script and invoke it through the available PowerShell tool:

```powershell
& pwsh -NoProfile -NonInteractive -File 'C:\Work\Invoke-CodexTask.ps1'
$codexScriptExit = $LASTEXITCODE
```

If only Windows PowerShell 5.1 is installed, use `powershell.exe` instead of `pwsh` and follow the encoding guidance above. Do not assume `pwsh` is installed.

When the script wraps a Codex process, propagate its failure status with `exit $codexExit` (or exit 1 for a launch failure), rather than returning a successful shell status after a failed Codex run. Reserve `exit` for the child script, not a snippet pasted into the user's interactive terminal. Keep prompt text in a file or stdin; avoid nested Bash-to-PowerShell command strings. If only a Bash tool is available, invoke the reviewed script using a correctly quoted `-File` path and the appropriate Windows shell executable.

`-NoProfile` can expose reliance on profile-defined aliases or PATH changes. Resolve executable paths explicitly. `-NonInteractive` means the wrapper must not rely on interactive prompts. It does not grant Codex broader permissions.

## Resume from PowerShell

Apply the core resume procedure first, including checking inherited effort and explaining any replacement. Use an explicit session ID when known; otherwise `--last` is scoped to the intended working directory.

Replace the direct example's prompt and arguments with the following, then reuse its encoding, stderr capture, and exit-status handling:

```powershell
$codexPrompt = @'
Continue the analysis and investigate the remaining risks.
'@
$codexArgs = @(
    'exec'
    '--skip-git-repo-check'
    'resume'
    '--last'
    '-m', 'gpt-6-astra'
    '-c', "model_reasoning_effort='medium'"
    '-'
)
```

Here Astra Medium must be the pair resolved by the resume policy, not an automatic switch to Astra. Replace `'--last'` with `$codexSessionId` for a known session. Validate flag placement against installed help. Wrap the invocation in `Push-Location -LiteralPath $codexWorkDir` and `try/finally { Pop-Location }` to scope session selection to the intended workspace without permanently changing the caller's directory. Preserve the authorized sandbox; do not pass unsupported resume flags.

## Background execution and failure diagnosis

- Prefer the direct foreground example when the host's timeout permits. If the tool yields a running-session handle, continue monitoring it instead of launching another Codex process.
- For an explicit process wrapper, redirect stdin, stdout, and stderr; write the prompt and close stdin. Drain stdout/stderr concurrently to avoid pipe-buffer deadlocks, retain a process handle, and wait with a finite timeout. Do not assume `.cmd` launchers behave like native executables under `System.Diagnostics.Process`.
- On cancellation, stop only the process tree created for this task. Do not kill every `node`, `codex`, or `powershell` process. Confirm termination before retrying.
- Diagnose failures in order: launcher resolution, prompt/argument delivery and EOF, exit status plus stderr, then the actual Codex authentication/network/sandbox error.
- Suppression syntax is `2>$null`, but capturing diagnostics is the default. A successful host login or HTTP request does not establish that a child sandbox has identical access. Report the specific failure without weakening execution policy or sandbox settings automatically.

## Acceptance checks for a Windows host

Before claiming a launcher/version is verified, exercise new sessions and resumes on Windows PowerShell 5.1 and PowerShell 7 as available, including through the actual VS Code extension tool. Check:

- Paths with spaces and prompts containing quotes, dollar signs, backticks, Unicode, and multiple lines reach the process intact.
- A finite prompt supplies EOF and does not leave a waiting child.
- Native stderr remains available; a deliberate nonzero exit is detected and preserved.
- Resume selects the intended session and applies the resolved permitted model/effort.
- Cancellation of a background invocation leaves no child belonging to that invocation running.

Use a harmless local argument/stdin probe first for shell mechanics; use an authorized read-only Codex task for end-to-end checks. Label untested platforms and live model availability honestly.

Sources: [PowerShell redirection](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_redirection), [native argument parsing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_parsing), [character encoding](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_character_encoding), [Claude Code Windows setup](https://code.claude.com/docs/en/setup#set-up-on-windows), [Codex CLI reference](https://developers.openai.com/codex/cli/reference).
