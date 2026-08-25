# DGX Spark Codex sandbox runbook

Last verified: 2026-08-25

Status: resolved

## Environment

- Hardware: NVIDIA DGX Spark, NVIDIA GB10
- OS: Ubuntu 24.04.4 LTS
- Kernel: `6.17.0-1031-nvidia`
- Architecture: `aarch64`
- Codex CLI: `0.149.1`
- Bubblewrap: `0.9.0`
- `uv`: `0.12.5`
- System Python: CPython `3.12.3`

## Original failure

Every command launched inside the Codex sandbox failed before the command
itself started:

```text
bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted
```

Standalone namespace tests also failed:

```text
unshare: write failed /proc/self/uid_map: Operation not permitted
bwrap: setting up uid map: Permission denied
```

The relevant host settings were:

```text
kernel.unprivileged_userns_clone = 1
user.max_user_namespaces = 511827
kernel.apparmor_restrict_unprivileged_userns = 1
```

Kernel audit logs showed AppArmor transitioning `bwrap` into the generic
`unprivileged_userns` profile and denying `setpcap`, `net_admin`, `sys_admin`,
and writes to `/proc/.../uid_map`.

## Root cause

Ubuntu 24.04 restricts unprivileged user namespaces through AppArmor. The
machine had Bubblewrap installed, but it did not have a loaded Bubblewrap
AppArmor profile. Codex therefore could not create its Linux sandbox.

This is a known Ubuntu 24.04 and Codex compatibility issue, not a Python,
`uv`, ARM, or DGX Spark failure:

- [OpenAI sandbox documentation](https://learn.chatgpt.com/docs/sandboxing)
- [OpenAI Codex issue #14919](https://github.com/openai/codex/issues/14919)
- [Ubuntu AppArmor issue #2046477](https://bugs.launchpad.net/ubuntu/+source/apparmor/+bug/2046477)

## Resolution used on this machine

Ubuntu 24.04 provides the preferred restrictive profile through the
`apparmor-profiles` package rather than the `bubblewrap` package.

First check that another Bubblewrap profile is not already loaded:

```bash
sudo aa-status | grep -i bwrap
```

On this machine the command returned no output. The documented Ubuntu 24.04
repair was then applied:

```bash
sudo apt update
sudo apt install apparmor-profiles apparmor-utils

sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict

sudo apparmor_parser -r /etc/apparmor.d/bwrap-userns-restrict
sudo systemctl reload apparmor.service
```

Do not globally disable
`kernel.apparmor_restrict_unprivileged_userns` unless the security tradeoff is
explicitly intended. The targeted profile keeps the global mitigation enabled.

## Verification

Run the Bubblewrap probe as the normal user, not with `sudo`:

```bash
bwrap --unshare-user --unshare-net \
  --ro-bind / / /bin/true &&
echo bwrap-ok
```

After restarting Codex, sandboxed `pwd` completed successfully.

The Python smoke test also passed when the `uv` cache was placed in writable
`/tmp`:

```bash
UV_CACHE_DIR=/tmp/codex-uv-cache \
  uv run --no-sync python -c \
  "import platform; print(platform.python_version(), platform.machine())"
```

Expected output includes:

```text
3.12.3 aarch64
```

## `uv` cache note

The Codex sandbox mounts the home-directory cache as read-only. Using the
default `uv` cache can therefore fail with:

```text
Could not acquire lock
Could not create temporary file
Read-only file system at /home/combacsa/.cache/uv/...
```

For commands run inside Codex, set a writable cache location:

```bash
UV_CACHE_DIR=/tmp/codex-uv-cache uv run <command>
```

The project `.venv` is inside the writable workspace and works normally.

## If the failure returns

Check the profile and recent AppArmor denials:

```bash
sudo aa-status | grep -i bwrap

sudo journalctl -k -b --no-pager |
  grep -Ei 'apparmor.*(DENIED|userns|bwrap)' |
  tail -50
```

Then rerun the Bubblewrap verification command above. If Bubblewrap succeeds
but Codex fails, record `codex --version` and check the OpenAI Codex issue
tracker for a sandbox regression.
