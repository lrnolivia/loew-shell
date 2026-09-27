# loew-shell

Private control plane for running approved shell commands on Lauren's macOS and Linux machines from normal ChatGPT through GitHub Actions self-hosted runners.

## Status

**Planning only.** No self-hosted runner or executable workflow has been installed yet.

## Goal

Make normal ChatGPT able to execute shell commands on either local machine without exposing SSH, opening inbound ports, or depending on Composio Custom MCP.

```text
normal ChatGPT
    |
    v
Composio
    |
    v
GitHub toolkit
    |
    v
workflow_dispatch
    |
    v
GitHub Actions
   / \
  v   v
loew-mac   loew-linux
  |          |
 zsh        bash
```

ChatGPT dispatches a workflow with a target host, optional working directory, and command. The selected self-hosted runner executes it, and ChatGPT reads the workflow status and logs.

## Why this exists

The current loew-inspector work proved that Cloudflare and the inspector gateway are healthy, while Composio Custom MCP sync continues to fail with `1606 Upstream_Forbidden`.

loew-shell avoids that experimental Custom MCP path entirely.

## Does loew-shell need Cloudflare?

**No.**

GitHub self-hosted runners initiate outbound HTTPS connections to GitHub. The Mac and Linux machines do not need:

- public SSH
- inbound firewall holes
- Cloudflare Tunnel
- a public API
- a public MCP endpoint

Cloudflare remains relevant to loew-inspector and loew-runner themselves, but it is not part of the loew-shell transport.

## Does normal ChatGPT need the standalone GitHub plugin?

**No.**

The intended path is normal ChatGPT -> Composio -> Composio's GitHub toolkit. The standalone GitHub plugin can remain uninstalled so it does not compete for unrelated requests.

The required GitHub operations are already available through the connected Composio GitHub account:

- dispatch a workflow
- inspect workflow runs
- list jobs
- retrieve logs

## Planned repository shape

```text
loew-shell/
├── .github/
│   └── workflows/
│       └── loew-shell.yml
├── scripts/
│   ├── install-runner-macos.sh
│   ├── install-runner-linux.sh
│   └── loew-shell-exec
├── IMPLEMENTATION_PLAN.md
└── README.md
```

No executable workflow is committed during the planning phase.

## Host labels

macOS runner:

```text
self-hosted
loew-shell
loew-mac
```

Linux runner:

```text
self-hosted
loew-shell
loew-linux
```

## Intended ChatGPT workflow

1. ChatGPT intentionally routes through Composio's GitHub toolkit.
2. ChatGPT dispatches `loew-shell.yml` with `host`, `cwd`, and `command`.
3. GitHub schedules the job only onto the explicitly labeled host.
4. The runner executes the command with that machine's normal shell.
5. stdout, stderr, exit status, hostname, cwd, and timing are written to the Actions log.
6. ChatGPT reads the run and continues iteratively.

## Security principles

- Repository stays private.
- Arbitrary shell execution is exposed only through `workflow_dispatch`.
- No `pull_request`, `pull_request_target`, public webhook, or issue-comment shell trigger.
- Jobs run only on explicit `loew-mac` or `loew-linux` labels.
- Workflow permissions default to `contents: read`.
- Runner accounts should not run as root.
- Every command has a hard timeout.
- Secrets must never be echoed into Actions logs.
- Commands and output remain visible in GitHub Actions history.
- Repository ownership and GitHub account security are part of the trust boundary.

## First validation after implementation

```sh
pwd
uname -a
whoami
printf 'loew-shell ok\n'
```

Once that works on both hosts, loew-shell can be used to finish loew-inspector and loew-runner without manual terminal copy/paste.
