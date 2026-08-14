# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Working rules

- **Never edit tracked files in this repo without explicit user confirmation first.** This role is applied to real production servers, so propose the change (what files, what content) and wait for the user to say go-ahead before using Edit/Write. This applies even to small-looking fixes.

## Overview

This is an Ansible Galaxy role (`pastir.haproxy`) that installs and configures HAProxy on Ubuntu (jammy/noble/resolute) from the official HAProxy Technologies APT repository (https://www.haproxy.com/downloads). It is a standalone role repo, not a playbook project — it gets consumed by including `role: pastir.haproxy` from other playbooks.

## Commands

Testing is done with Molecule + Docker:

```bash
cd molecule/default  # molecule commands run from repo root, config lives in molecule/default/

molecule test         # full lifecycle: dependency, create, converge, verify, destroy
molecule converge     # apply the role to the running test containers (fast iteration)
molecule verify       # run the verifier only
molecule destroy      # tear down test containers
```

Linting (invoked automatically by molecule as the `provisioner.lint` step, per `molecule/default/molecule.yml`):

```bash
ansible-lint
yamllint .
```

Molecule targets two Docker platforms in parallel: `ubuntu-jammy` (22.04) and `ubuntu-noble` (24.04), defined in `molecule/default/molecule.yml`. Resolute (26.04) is supported by the role but not currently covered by Molecule (no stable `geerlingguy/docker-ubuntu*-ansible` image for it yet).

There is also a standalone playbook at `tests/test.yml` (runs the role against `localhost` using `tests/inventory`) for a quick non-Molecule smoke test.

## Architecture

Task flow (`tasks/main.yml`):
1. `include_tasks: "{{ ansible_facts['distribution'] }}.yml"` (tag `install`) — dispatches to a distro-specific install file. Currently only `tasks/Ubuntu.yml` exists; adding another OS means adding a new `tasks/<Distribution>.yml`.
2. `include_tasks: configure.yml` (tag `configure`) — renders/uploads the HAProxy config and wires up validate+restart handlers.

`tasks/Ubuntu.yml` enforces a support matrix before installing anything, driven by `haproxy_supported_versions` in `vars/main.yml` (a dict of `distribution_release -> [allowed haproxy_major_version...]`):
- Fails if `ansible_facts['distribution_release']` isn't a key in `haproxy_supported_versions` (jammy/noble/resolute only — focal is not supported).
- Fails if `haproxy_major_version` isn't in that release's allowed-versions list (e.g. jammy only gets 3.3/3.4, noble also gets 3.2, resolute only gets 3.3/3.4).
- Downloads the HAProxy Technologies signing key (`haproxy_gpg_key_url`) to `haproxy_gpg_key_path` (`/usr/share/keyrings/HAPROXY-key-community.asc`) and adds a `deb [signed-by=...]` line pointing at `https://www.haproxy.com/download/haproxy/performance/ubuntu/ha<major-no-dot>` for the current distro release (e.g. `ha34 jammy`), then installs the pinned `haproxy_version` (e.g. `3.4.*`) of `haproxy-awslc`.
- That repo is HAProxy Technologies' "performance" build, linked against AWS-LC instead of OpenSSL — it only ships `haproxy-awslc` (there is no plain `haproxy` package in it). `haproxy-awslc` provides/conflicts with `haproxy`, so it's a drop-in replacement, but any task/doc referring to the installed package should say `haproxy-awslc`, not `haproxy`.
- This replaced an earlier approach that added the `vbernat` PPA via `apt_repository` + a keyserver GPG fetch; that broke in practice (`Unable to get required signing key` from the keyserver, and "does not have a Release file" for combos the PPA doesn't actually publish). The `signed-by` + official-repo approach avoids both failure modes.

`tasks/configure.yml` has two mutually exclusive paths, selected by whether `haproxy_custom_config_file` is set:
- **Custom file mode**: `ansible.builtin.copy` a user-supplied config file straight to `haproxy_cfg_file`.
- **Templated mode** (default): sets `merged_haproxy_conf` from `haproxy_config` and renders `templates/haproxy.cfg.j2` to `haproxy_cfg_file`.

Both paths notify the same two handlers (`handlers/main.yml`), in order: `Validate HAProxy configuration` (`haproxy -c -f ...`, prints stderr if any) then `Restart HAProxy`.

`templates/haproxy.cfg.j2` is a generic renderer over `merged_haproxy_conf`, a list of section dicts (each with a `section` key like `global`/`defaults` plus arbitrary directive keys). It handles three value shapes per directive:
- falsy value → bare keyword on its own line (e.g. `log global:` with no value)
- list/iterable (non-string) → repeats the key once per list item (used for `log`, `stats`, `timeout`, `errorfile`, multi-value directives)
- scalar → `key value` on one line

This means the config structure in `defaults/main.yml` / a caller's `haproxy_config` override directly drives the rendered `haproxy.cfg` — there's no per-directive validation, so typos in keys are rendered verbatim into the config and only caught by the `haproxy -c` validation handler at apply time.

## Key variables

- `haproxy_major_version` / `haproxy_version` (`defaults/main.yml`) — control which repo suite is added and the pinned apt package version; must stay in sync (e.g. `"3.4"` / `"3.4.*"`), and `haproxy_major_version` must be one of the versions allowed for the target release in `haproxy_supported_versions` (`vars/main.yml`).
- `haproxy_gpg_key_url` / `haproxy_gpg_key_path` (`defaults/main.yml`) — where the HAProxy Technologies signing key is fetched from and stored.
- `haproxy_cfg_file` — destination path, default `/etc/haproxy/haproxy.cfg`.
- `haproxy_custom_config_file` — when non-empty, bypasses the template entirely (see above).
- `haproxy_config` — the structured config consumed by `haproxy.cfg.j2`; overriding this in a playbook fully replaces the default global/defaults sections rather than merging.
