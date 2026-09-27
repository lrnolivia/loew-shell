# loew-shell implementation plan

## 1. Purpose

Build a small, deterministic execution transport that lets **normal ChatGPT** run shell commands on Lauren's Mac and Linux machines through the existing Composio -> GitHub connection.

loew-shell is not a general autonomous agent system and does not replace loew-runner. It is the narrow remote execution plane that lets ChatGPT perform terminal work directly.

## 2. Architectural decision

Use:

- a private GitHub repository
- GitHub Actions `workflow_dispatch`
- one self-hosted runner on macOS
- one self-hosted runner on Linux
- Composio's connected GitHub toolkit as the ChatGPT control path

Do **not** require:

- Cloudflare
- public SSH
- Cloudflare Tunnel
- Composio Custom MCP
- a public API
- an inbound listener on either machine
- the standalone ChatGPT GitHub plugin

The runners maintain outbound connections to GitHub. ChatGPT talks to GitHub through Composio.

## 3. Control flow

```text
normal ChatGPT
      |
      | Composio
      v
GitHub toolkit
      |
      | workflow_dispatch(host, cwd, command)
      v
lrnolivia/loew-shell
      |
      v
GitHub Actions scheduler
      |
      +-------------------------+
      |                         |
      v                         v
[self-hosted,              [self-hosted,
 loew-shell,                loew-shell,
 loew-mac]                  loew-linux]
      |                         |
      v                         v
macOS zsh                  Linux bash
```

## 4. Non-goals

The first version will not provide:

- a live interactive PTY
- terminal screen streaming
- arbitrary execution from pull requests
- arbitrary execution from issue comments
- a public command API
- remote desktop
- SSH replacement for human administration
- cross-machine file sync beyond what explicit commands perform

Command-by-command execution is acceptable because the primary workloads are diagnostics, Git, curl, builds, tests, deployment commands, configuration edits, and log inspection.

## 5. Phase 1 — workflow contract

Create:

```text
.github/workflows/loew-shell.yml
```

Trigger:

```yaml
on:
  workflow_dispatch:
```

No other event may execute arbitrary commands.

### Inputs

`host`
- required
- choice: `mac` or `linux`

`cwd`
- optional string
- empty means the runner's configured default working directory

`command`
- required string
- the exact shell command to execute

`timeout_minutes`
- optional
- conservative bounded default
- must not allow unbounded jobs

### Runner selection

Resolve host deterministically:

```text
mac   -> [self-hosted, loew-shell, loew-mac]
linux -> [self-hosted, loew-shell, loew-linux]
```

Do not accept a free-form runner label from the caller.

### Workflow permissions

Start with:

```yaml
permissions:
  contents: read
```

Grant additional GitHub permissions only when a concrete loew-shell feature requires them.

## 6. Phase 2 — execution wrapper

Create:

```text
scripts/loew-shell-exec
```

Responsibilities:

1. Resolve the requested working directory safely.
2. Refuse an invalid/nonexistent cwd.
3. Print a compact execution envelope:
   - timestamp
   - hostname
   - OS
   - user
   - cwd
   - selected shell
4. Execute exactly one supplied command.
5. Preserve stdout and stderr.
6. Preserve the real exit code.
7. Emit a completion envelope with elapsed time and exit code.
8. Never print environment variables wholesale.
9. Never print runner registration credentials.
10. Never silently retry a failed shell command.

### Shell choice

macOS:
- prefer `/bin/zsh -lc`

Linux:
- prefer `/bin/bash -lc`
- allow a future configurable shell only if a real need appears

The command contract should remain identical across both platforms.

## 7. Phase 3 — macOS runner bootstrap

Create:

```text
scripts/install-runner-macos.sh
```

Responsibilities:

- verify macOS
- verify required command-line tools
- create a dedicated loew-shell runner directory
- download the current GitHub Actions runner using GitHub's official runner setup flow
- configure labels:
  - `loew-shell`
  - `loew-mac`
- install/start the runner using the supported macOS service mechanism
- verify the runner is online

The installer must not contain a registration token in source control.

Prefer a normal user context. Do not run the runner as root.

## 8. Phase 4 — Linux runner bootstrap

Create:

```text
scripts/install-runner-linux.sh
```

Responsibilities mirror macOS:

- verify supported Linux environment
- create the runner directory
- download the official GitHub Actions runner
- configure labels:
  - `loew-shell`
  - `loew-linux`
- install/start using the supported service mechanism
- verify online state

Do not assume a specific desktop environment or distro unless required by GitHub's runner dependencies.

## 9. Phase 5 — ChatGPT control path

Normal ChatGPT should use **Composio intentionally** for loew-shell.

Required Composio GitHub operations:

1. `GITHUB_CREATE_A_WORKFLOW_DISPATCH_EVENT`
2. list/find the newly created run
3. `GITHUB_GET_A_WORKFLOW_RUN`
4. list jobs when needed
5. download/read workflow or job logs

The standalone GitHub plugin is intentionally not required.

### Routing rule

When the user asks ChatGPT to run something on:
- "my Mac"
- "lobook-pro"
- "my Linux machine"
- "Bazzite"
- "my terminal"

and loew-shell is available, route through:

```text
Composio -> GitHub -> loew-shell workflow
```

Do not let generic GitHub repository questions automatically imply loew-shell execution.

## 10. Phase 6 — correlation and result retrieval

A dispatch request should include or generate a correlation identifier so ChatGPT can reliably identify the correct run when several commands are close together.

Preferred approach:
- add a `request_id` workflow input
- include it in the run name
- print it at the beginning and end of the job

Example:

```text
loew-shell / mac / req-20260927-001
```

This prevents ChatGPT from accidentally reading an older run.

## 11. Phase 7 — security hardening

Required before routine use:

- private repository
- no arbitrary-command PR triggers
- no arbitrary-command issue/comment triggers
- no fork-triggered execution
- explicit host allowlist
- explicit runner labels
- minimal workflow token permissions
- hard execution timeout
- no root runner
- no broad environment dump
- no secrets echoed into logs
- runner software kept current
- repository account protected with strong authentication
- command history retained in Actions for auditability

### Important trust boundary

A self-hosted runner executes code on the host. Anyone who can alter or dispatch the arbitrary-command workflow may effectively gain command execution on that machine.

Repository access is therefore equivalent to privileged machine access for this subsystem.

## 12. Optional v2 safeguards

Do not block v1 on these, but design so they can be added:

- read-only mode
- command risk classification
- explicit approval for destructive commands
- allowed cwd roots
- maximum output size
- redaction patterns for known secret formats
- kill/cancel operation
- structured JSON result artifact
- file upload/download helpers
- host health/status workflow
- queued-command view

## 13. Acceptance criteria

Implementation is complete only when all of the following are verified.

### macOS

ChatGPT can dispatch:

```sh
pwd
uname -a
whoami
printf 'loew-shell ok\n'
```

and accurately read the output and exit code.

### Linux

The same command contract works without changing ChatGPT behavior.

### cwd

A valid cwd executes there.

An invalid cwd fails clearly before running the supplied command.

### exit status

```sh
exit 7
```

must cause the workflow result/log contract to preserve exit code 7.

### stdout/stderr

A command producing both streams must preserve both.

### security

A pull request cannot execute an arbitrary shell command on either self-hosted runner.

### normal ChatGPT

The workflow can be dispatched and inspected through Composio's GitHub toolkit without installing the standalone GitHub plugin.

## 14. First production use

Once acceptance criteria pass, use loew-shell to resume the current systems work:

1. inspect loew-inspector source/deployment state
2. reproduce the Composio Custom MCP sync failure from the machine when useful
3. verify Cloudflare endpoints
4. inspect loew-runner local/repository state
5. fix and test runner/inspector
6. update canonical documentation from verified implementation truth

## 15. Future architecture

If normal ChatGPT later gains full write-capable private MCP support on the user's plan, loew-shell may gain a direct MCP transport.

That would be an additional transport, not a reason to discard the deterministic GitHub Actions path unless the direct path proves equally auditable and reliable.
