# Chapter 4 — Execution Isolation and Checkpoints

A coding agent runs commands chosen by a model on a machine you care about. Path checks and allowlists inside the harness help, but they run in the same process as the agent and they inspect commands as text. A model can get around them, by accident or because a prompt injection in a file it read told it to. This chapter covers moving the real boundary to the operating system, and making every change undoable, so a bad turn costs a rewind rather than a recovery.

## Threat model

Before choosing a sandbox, be clear about what you're protecting against. For a coding harness, the realistic risks are:

| Risk | Example | What stops it |
|---|---|---|
| Accidental damage | `rm -rf build /` with a stray space; `git clean -fdx` in the wrong directory | Filesystem limited to the workspace; checkpoints |
| Secret exposure | The agent runs `env` or reads `~/.aws/credentials` and the output lands in a log or a PR | Scrubbed environment; no access to the home directory |
| Data exfiltration | A prompt injection in a README tells the agent to `curl` a file to an attacker's server | No network, or an egress allowlist |
| Resource exhaustion | A test with an infinite loop; a fork bomb; filling the disk | CPU, memory, process, and disk limits; timeouts |
| Persistence | The agent installs something that outlives the session | Throwaway environment per session |

The first two happen often, even with a well-behaved model. The third is why untrusted content (web pages, issues, dependency READMEs) plus network access plus secrets is the combination to avoid. If you can only remove one of the three, remove the network.

## Layers

From weakest to strongest. A serious harness uses several at once.

**1. In-process checks.** Path containment, program allowlists, permission rules, hooks. Fast, and they give the model clear feedback. They aren't a boundary: `python -c`, a script file, or a symlink created by one command and used by the next gets past them.

**2. Process controls.** Run commands as a child process with a scrubbed environment, a working directory inside the workspace, `resource` limits (CPU time, memory, open files, process count), and a timeout that kills the whole process group, not just the parent. This stops accidents and runaway processes, not a determined escape.

**3. OS sandboxes.** Kernel features that limit what a process can touch:
- **macOS:** Seatbelt profiles (`sandbox-exec`). Deprecated as a public interface, but still what several coding agents use.
- **Linux:** bubblewrap (`bwrap`) builds a private mount namespace: bind the workspace read-write, system directories read-only, nothing else, optionally no network. Landlock and seccomp add filesystem and syscall restrictions.

Claude Code and Codex both ship an OS-level sandbox built on these. They start fast (milliseconds) and let commands run against the real workspace.

**4. Containers.** Docker or Podman with the workspace mounted and everything else locked down:

```bash
docker run --rm \
  --network none \
  --read-only --tmpfs /tmp:rw,size=256m \
  --cpus 2 --memory 2g --pids-limit 256 \
  --user "$(id -u):$(id -g)" \
  --cap-drop ALL --security-opt no-new-privileges \
  -v "$WORKSPACE:/workspace:rw" -w /workspace \
  harness-runner:latest \
  pytest -q
```

Each flag closes a gap: no network, an immutable root filesystem, bounded resources, a non-root user, no extra kernel capabilities. A new container per command is simple and stateless. A long-lived container per session is faster and keeps installed dependencies, at the cost of state carrying over between commands.

**5. VMs and remote sandboxes.** Firecracker microVMs, gVisor, or a hosted sandbox service give a stronger boundary than containers, which share the host kernel. Use them for untrusted code or multi-tenant services where one user's agent must never reach another's.

### Network egress

"No network" breaks package installs and documentation lookups. The usual middle ground is an **egress allowlist**: route the sandbox's traffic through a proxy that permits only specific hosts (your package registry, your git host) and logs everything else. Default to no network, open specific hosts when a task needs them, and never allow general internet access in a session that also has secrets.

### Credentials

The sandbox should hold only the credentials the task needs, scoped as narrowly as possible: a read-only token, a short-lived credential, a token for one repository. Better still, keep credentials out entirely and give the agent a tool that performs the privileged action, like "open a pull request", outside the sandbox, where the harness can check and log it.

## The runner interface

Keep isolation behind one interface so the rest of the harness doesn't care where commands run:

```python
@dataclass(frozen=True)
class CommandResult:
    exit_code: int
    stdout: str
    stderr: str
    timed_out: bool
    duration_s: float

class CommandRunner(Protocol):
    def run(self, argv: Sequence[str], cwd: Path, timeout: float, env: Mapping[str, str]) -> CommandResult: ...
```

`LocalRunner` (process controls only) for development and tests, `DockerRunner` for real use, a remote runner for a hosted service. Pick one from config. The `bash` tool, the stop gate, and any hook that runs commands all go through the same runner, so they all get the same isolation.

One subtlety: paths. With a container, the workspace is `/workspace` inside but `/home/you/project` outside. Decide which paths the model sees (usually the inside ones, so commands and file tools agree) and translate at the boundary.

## Checkpoints: making every turn undoable

Isolation limits what a bad command can reach. Checkpoints handle the damage it can still do inside the workspace, which is where all the valuable work is.

The idea: before each turn that may change files, snapshot the workspace. If the turn goes wrong, restore the snapshot. Claude Code offers this as rewind; users can undo the agent's last changes without touching git history.

Git makes a good snapshot store, but you must not disturb the user's own git state: their branch, their staging area, their `HEAD`. The trick is a private index and private refs:

```bash
export GIT_INDEX_FILE="$(mktemp)"                   # private staging area
git add -A                                          # stage everything into it
tree=$(git write-tree)                              # store the tree object
commit=$(git commit-tree "$tree" -m "checkpoint 7") # commit with no branch
git update-ref refs/harness/checkpoints/7 "$commit" # private ref keeps it alive
rm "$GIT_INDEX_FILE"
```

The user's branch, index, and `HEAD` are untouched. `git log` doesn't show checkpoints, but they're safe from garbage collection because a ref points to them. To rewind, check out that tree into the working directory and remove files that exist now but weren't in the snapshot.

Decisions to make and document:

- **Ignored files.** `git add -A` skips `.gitignore`d files, so build outputs and `.env` aren't snapshotted. That's usually right: you don't want secrets in checkpoint objects. But rewinding then won't restore an ignored file the agent deleted.
- **When to checkpoint.** Before each model turn that has write-capable tools available is simple and cheap. Git deduplicates identical trees, so unchanged turns cost almost nothing.
- **Cleanup.** Delete old checkpoint refs at the end of the session, or keep the last N.
- **Not a repo?** Disable checkpoints with a warning, or fall back to copying the workspace to a temp directory for small projects.

### Worktrees for parallel agents

When several agents work on one repository at the same time, as parallel subagents or eval trials do, give each its own **git worktree**: a separate working directory on its own branch, sharing the object store. They can't overwrite each other's files, and merging their results is a normal git operation. mod-208's exercise-03 used fresh copies per trial; worktrees are the faster version for real repositories.

## Key takeaways

- Write down the threat model: accidents, secret exposure, exfiltration, resource exhaustion, persistence.
- In-process checks give feedback, not protection. The boundary is the OS: sandboxes, containers, or VMs.
- Default to no network; open specific hosts through an egress proxy; never combine untrusted content, network access, and secrets.
- Put isolation behind a `CommandRunner` interface so every command (tools, hooks, gates) gets the same treatment.
- Checkpoint the workspace before each turn using a private index and private refs, without touching the user's git state.
- Give parallel agents separate worktrees.
