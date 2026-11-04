# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Working rules

- **Never edit tracked files in this repo without explicit user confirmation first.** This role is applied to real production servers, so propose the change (what files, what content) and wait for the user to say go-ahead before using Edit/Write. This applies even to small-looking fixes.

## Overview

This is an Ansible Galaxy role (`pastir.haproxy`) that installs and configures HAProxy on Ubuntu (jammy/noble/resolute) from the official HAProxy Technologies APT repository (https://www.haproxy.com/downloads). It is a standalone role repo, not a playbook project — it gets consumed by including `role: pastir.haproxy` from other playbooks.

## Commands

Testing is done with Molecule + podman (systemd-enabled Ubuntu images built from `molecule/default/Dockerfile.j2`):

```bash
molecule test         # destroy, create, converge, idempotence, side_effect, verify, destroy
molecule converge     # fast iteration
yamllint . && ansible-lint
```

Platforms: `haproxy-noble` (24.04) and `haproxy-resolute` (26.04). `side_effect.yml` checks that a config change is a
hitless reload (master PID unchanged) and that an invalid config is rejected without touching the running file.

## Architecture

`tasks/main.yml`: `tasks/<Distribution>.yml` (install) → `tasks/configure.yml` → ensure the service is enabled and started.

`tasks/Ubuntu.yml`: asserts the support matrix (`haproxy_supported_versions` in `vars/main.yml`, and
`haproxy_version` must belong to `haproxy_major_version`). It downloads the signing key, removes a legacy
`haproxy.list`, adds `haproxy.sources` (deb822) for
`https://www.haproxy.com/download/haproxy/performance/ubuntu/ha<major-no-dot>`, then installs
`haproxy-awslc=<haproxy_version>-*` and holds it. The repository ships only `haproxy-awslc` (no plain `haproxy`).

`tasks/configure.yml`: either copies `haproxy_custom_config_file` or renders `templates/haproxy.cfg.j2`. Both use
`validate: haproxy -c -f %s` and notify the single handler `Reload HAProxy` (`state: reloaded`).

`templates/haproxy.cfg.j2` renders `haproxy_config` (a list of sections):
- scalar → `key value`;
- list → one line per item, and list items that are dicts → `key sub value`;
- dict → `key sub value` per entry;
- null or empty → bare `key`;
- `raw` → lines verbatim.

## Key variables

- `haproxy_major_version` / `haproxy_version`: repository suite and exact pinned version (e.g. `"3.4"` / `"3.4.7"`).
- `haproxy_hold`: `apt-mark hold` the package.
- `haproxy_cfg_file`, `haproxy_custom_config_file`, `haproxy_config` (overriding it replaces the default
  global/defaults sections).
