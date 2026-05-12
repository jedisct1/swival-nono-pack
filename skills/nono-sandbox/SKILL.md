---
name: nono-sandbox
description: Understands nono security sandbox constraints. Use when Swival is running inside a nono sandbox, when read_file, write_file, edit_file, run_command, run_shell_command, or fetch_url returns a permission error, or when the user asks about sandbox capabilities.
---

# nono Sandbox Awareness

You are running inside a **nono security sandbox**. nono enforces OS-level capability restrictions using Landlock on Linux and Seatbelt on macOS. These are kernel-enforced boundaries that the sandboxed process cannot bypass from within.

## How the sandbox works

nono applies an allow-list of filesystem paths and network rules before Swival starts. Everything not explicitly allowed is blocked at the kernel level.

- **Filesystem.** Only paths granted by the active profile are accessible. The current working directory normally has read-write access. `~/.config/swival` is granted so Swival's own configuration, skills, and commands work.
- **Network.** May be fully allowed, filtered to specific domains, or completely blocked depending on the profile. The default `swival` profile leaves outbound network open so hosted LLM providers and local servers on `127.0.0.1` both work.
- **No escalation.** There is no `sudo`, no `chmod`, no alternate path that grows the allow-list. Capabilities are fixed at session start.

## Recognising a sandbox denial in Swival tool results

A nono denial usually surfaces in Swival as a tool result starting with `error:` and containing one of:

- `Permission denied` or `Operation not permitted`
- `EACCES` or `EPERM`
- `Read-only file system` on a path you expected to be writable
- network calls that fail with connection refused or DNS errors when the profile blocks them

Treat these as **policy boundaries, not bugs in the code**. Do not:

1. Retry the same operation in a loop.
2. Try cosmetic variations of the same path (`./foo` then `foo` then absolute `/abs/foo`).
3. Suggest the user run `sudo`, `chmod`, or rebuild filesystem state.
4. Move data to a different location to work around the boundary.
5. Apologise repeatedly or claim you will "try another approach". There is no other approach from inside.

## Diagnosing the denial

Run `nono why --self` to ask the running sandbox why a specific path or network operation is blocked. Use the `run_command` tool:

```text
nono why --self --path <absolute-path> --op <read|write|readwrite> --json
nono why --self --host <hostname> --port <port> --json
```

The JSON response includes:

- `status`: `allowed` or `denied`.
- `reason`: e.g. `granted_path`, `sensitive_path`, `path_not_granted`, `network_allowed`, `network_blocked`.
- `policy_source` or `source`: which profile rule or group covers the path (e.g. `group:deny_credentials`, `profile`, `user`).
- For allows: `granted_path` and `access`.

Always call `nono why --self` once before telling the user what is happening. Do not guess. The diagnosis is one tool call away.

If `nono why --self` itself fails or returns `not_sandboxed`, you are not actually inside a nono session and this skill does not apply. Treat the error as a real code or filesystem failure.

## Talking to the user about denials

After diagnosing, present a clear short summary:

1. What you tried.
2. What `nono why --self` reported (group or rule name, in one line).
3. The user's realistic options:
   - **Quick fix.** Restart Swival with the path granted on the command line:
     ```sh
     nono run --profile swival --allow <path> -- swival "..."
     ```
     Use `--read` for read-only or `--write` for write-only access. For a single file rather than a directory, use `--allow-file`, `--read-file`, or `--write-file`.
   - **Persistent fix.** Draft a profile so the access is granted by default in future sessions.
   - **Stop.** If the denial is from a deny group like `deny_credentials`, the user probably does not want to grant access at all. Confirm before recommending an override.

Never propose `--allow $HOME` or `--allow /`. If the user needs broad access for a specific task, recommend the narrowest grant that covers it.

## Writing a profile draft

Active profiles live at `~/.config/nono/profiles/<name>.json`. That directory is intentionally not writable from inside the sandbox. Drafts go to `~/.config/nono/profile-drafts/<name>.json`, and the user promotes them out-of-band with `nono profile promote <name>`.

Before drafting:

- Run `nono profile guide` to get the current schema.
- Run `nono profile show swival --json` to see the active profile.

If the active profile is built-in or pack-provided (the case for `swival`), **do not draft a same-named replacement.** Draft a derived profile, e.g. `swival-local`, that extends the existing one and adds only the extra access:

```json
{
  "extends": "swival",
  "meta": { "name": "swival-local", "description": "Local extensions to the swival profile" },
  "filesystem": { "read": ["/path/needed/for/this/repo"] }
}
```

Write that JSON to `~/.config/nono/profile-drafts/swival-local.json` and tell the user:

> Drafted `swival-local`. Run `nono profile promote swival-local` to review and apply, then start Swival with `nono run --profile swival-local -- swival "..."`.

If a user-managed profile of the same name already exists at `~/.config/nono/profiles/<name>.json`, read it first, compute its SHA-256, base your changes on it, and write the hash to `~/.config/nono/profile-drafts/<name>.base`. This lets `nono profile promote` detect concurrent edits.

If `~/.config/nono/profile-drafts/` does not exist or is not writable, tell the user to upgrade nono rather than trying to bypass the boundary.

## Checking what is allowed

If the `NONO_CAP_FILE` environment variable is set, it points to a JSON document listing the active capabilities. Read it to enumerate accessible paths and the network policy without one `nono why` call per path:

```sh
cat "$NONO_CAP_FILE"
```

Fields you can rely on:

- `fs`: array of filesystem capabilities with `path`, `resolved`, and `access`.
- `net_blocked`: boolean.

## Common scenarios

**Swival blocked from reading a config file outside the workspace.** Run `nono why --self --path <abs> --op read --json`. If the path is sensitive (SSH keys, cloud credentials), explain that it is intentionally denied and do not work around it. If it is a legitimate workspace input, suggest a narrow `--read` grant or a `swival-local` profile draft.

**Network request failed.** Check the profile with `nono why --self --host <host> --port <port> --json`. If the profile blocks network, advise the user to switch profiles or pick a provider that does not require network (e.g. `--provider llamacpp` against a process that is started outside the sandbox).

**Local LLM server unreachable.** Loopback is allowed by the default `swival` profile, but the server itself must already be running on the host. Suggest starting it before launching `nono run`.

**Writing to a `node_modules`, `.next`, `__pycache__`, `.swival`, or `*.pyc` path.** These are excluded from rollback snapshots by the `swival` profile. Writes still work, but those paths will not be captured in `nono rollback`. Tell the user once if it matters.

**A skill or command file is unreadable.** Skills and commands live under `~/.config/swival/`, which the `swival` profile grants r+w. If a path under that directory still fails, run `nono why --self` to confirm; otherwise the file may simply not exist.

## Reminders for yourself

- The Swival application-layer flags `--files`, `--commands`, `--add-dir`, and `--yolo` still apply inside the sandbox. They are an additional agent-level guardrail, not the boundary. nono is the boundary.
- Do not pass `--sandbox agentfs` to Swival when already inside `nono run`. AgentFS is a second OS sandbox; stacking them is unsupported and will fail or behave unpredictably.
- Tool results in Swival begin with `error:` on failure. Repeated identical errors trigger the loop's own guardrail; do not wait for that, diagnose with `nono why --self` after the first denial.
