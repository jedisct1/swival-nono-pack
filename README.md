# swival nono pack

Run [Swival](https://github.com/Swival/swival) inside a [nono](https://nono.sh) sandbox. When Swival hits a permission denial, it diagnoses it cleanly instead of retrying in a loop, and offers you a narrow fix.

## Install

```sh
nono pull <namespace>/swival
```

## Run

```sh
nono run --profile swival --allow-cwd -- swival "review this repository"
```

`--allow-cwd` grants the current directory read+write for the session. Without it, nono prompts interactively (and denies non-interactively, which trips up scripted runs).

Local model server:

```sh
nono run --profile swival --allow-cwd -- \
  swival --provider llamacpp "fix the failing tests"
```

Make sure the local server is already running on the host before you launch the sandboxed Swival. Loopback is open in the profile so Swival can reach it.

OpenRouter or any hosted provider:

```sh
nono run --profile swival --allow-cwd -- \
  swival --provider openrouter --model z-ai/glm-5 "summarise this patch"
```

## What you get

The pack adds a `swival` nono profile (shown in `nono profile list` as `from <namespace>/swival`) and a Swival skill called `nono-sandbox`. The profile gives Swival the access it needs to run normally: the working directory, `~/.config/swival`, language runtimes, Homebrew, git config, outbound network, and loopback. It blocks credentials, keychains, browser data, and shell history.

The skill is registered in Swival's global skill catalog as soon as it's installed. When Swival hits a permission error inside the sandbox, the model should load the skill and call `nono why --self` to find out exactly what is blocked and why, then tell you in plain language. If you actually need that access, it offers a one-shot grant for this session or drafts a `swival-local` profile you can promote to make the change persistent.

## When you need extra access

For a single session, add the path on the launch command:

```sh
nono run --profile swival --allow-cwd --allow ~/some/other/project -- swival "..."
```

Use `--read` for read-only, `--write` for write-only, `--allow-file` for a single file. Swival itself will suggest the right flag when it hits the denial.

For repeated access, accept Swival's offer to draft a `swival-local` profile. Drafts land in `~/.config/nono/profile-drafts/`. Review and apply with:

```sh
nono profile promote swival-local
nono run --profile swival-local --allow-cwd -- swival "..."
```

## Provider auth files

API keys passed through environment variables work without any extra setup. Providers that cache credentials on disk need a one-time grant on first login. The most common case is the `chatgpt` provider:

```sh
nono run --profile swival --allow-cwd --allow $HOME/.config/litellm -- swival --provider chatgpt "..."
```

Once tokens are written, you can usually drop the flag unless the provider refreshes them often.

## Do not stack with `--sandbox agentfs`

Swival has its own optional OS sandbox flag (`--sandbox agentfs`). Don't combine it with `nono run` — two stacked sandboxes will either fail or double-isolate in confusing ways. nono is the boundary here.

## Uninstall

```sh
nono remove <namespace>/swival
```

The built-in `swival` profile that ships with nono becomes active again automatically.
