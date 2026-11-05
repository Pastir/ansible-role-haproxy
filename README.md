# Ansible Role: pastir.haproxy

Installs HAProxy from the official HAProxy Technologies "performance" APT repository and manages `haproxy.cfg`.

- The package is `haproxy-awslc` (built against AWS-LC). It provides and conflicts with the plain `haproxy` package.
- The exact version is pinned and held (`haproxy_version`).
- The config is **validated before it is written** (`haproxy -c`), so a broken config never replaces the
  running one.
- Changes are applied with a **hitless reload**, so existing connections are not dropped.
- **A version change of an installed package** requires `haproxy_allow_upgrade: true`. Without it a host whose
  installed `haproxy-awslc` differs from `haproxy_version` (or that has the plain `haproxy` package) fails before
  the role writes anything: no key, apt source, `python3-debian` or package is touched. A fresh install works
  with the default. With `true` the pinned version is installed over the hold (an upgrade or a downgrade), held
  again and picked up by a hitless reload (the master re-executes the new binary). Do it one node at a time:
  `-e haproxy_allow_upgrade=true`.

## Requirements

- ansible-core >= 2.16
- Ubuntu 22.04 (jammy), 24.04 (noble), 26.04 (resolute)

| Release | Major versions |
|---|---|
| jammy | 3.3, 3.4 |
| noble | 3.2, 3.3, 3.4 |
| resolute | 3.3, 3.4 |

## Variables

| Variable | Default | Description |
|---|---|---|
| `haproxy_major_version` | `"3.4"` | Selects the repository suite |
| `haproxy_version` | `"3.4.7"` | Exact version. Empty means the latest of the major version |
| `haproxy_package` | `haproxy-awslc` | |
| `haproxy_hold` | `true` | `apt-mark hold` |
| `haproxy_allow_upgrade` | `false` | Allow changing the version of an installed package (see above) |
| `haproxy_repo_enabled` | `true` | Add the repository |
| `haproxy_gpg_key_url` / `haproxy_gpg_key_path` | HAProxy key | |
| `haproxy_cfg_file` | `/etc/haproxy/haproxy.cfg` | |
| `haproxy_custom_config_file` | `""` | Complete config file from the controller. If set, the template is not used |
| `haproxy_config_template` | `haproxy.cfg.j2` | Template for `haproxy.cfg`. Set it to your own full template (absolute path or a file in the playbook's `templates/`). Validated with `haproxy -c` before it is written |
| `haproxy_config_diff` | `true` | Show the `haproxy.cfg` diff in `--diff` mode. **Set to `false` when the config contains secrets** (`stats auth`, `userlist` passwords), otherwise they are printed |
| `haproxy_service_enabled` | `true` | |
| `haproxy_config` | global + defaults | Structured config, see below |

### `haproxy_config` format

A list of sections. `section` is the header line. Every other key is a directive:

```yaml
haproxy_config:
  - section: global
    log: [/dev/log local0, /dev/log local1 notice]      # list   -> one line per item
    stats:
      - socket: /run/haproxy/admin.sock mode 660 level admin   # dict item -> "stats socket ..."
      - timeout: 30s
    daemon:                                             # null   -> bare keyword
  - section: listen pg_primary
    bind: 127.0.0.1:7432
    option: httpchk OPTIONS /
    server:
      - pg1 10.0.0.1:5432 check port 8008
      - pg2 10.0.0.2:5432 check port 8008
    raw:                                                # copied as is
      - "# comment"
```

Overriding `haproxy_config` replaces the default `global` and `defaults` sections entirely. Repeat them in your
value if you need them. Keys inside a section are rendered in the order you write them.

## Example

```yaml
- hosts: haproxy
  become: true
  roles:
    - role: pastir.haproxy
      vars:
        haproxy_version: "3.4.7"
```

## Tags

`haproxy`, `install` / `haproxy_install`, `configure` / `haproxy_config`

## Testing

`molecule test` (podman): Ubuntu 24.04 and 26.04. The test covers idempotence, a reload that keeps the master
PID, and rejection of an invalid config. On 24.04 `prepare` installs an older held `haproxy-awslc` from a legacy
`haproxy.list`: the default run must fail and leave apt sources, keys and packages untouched, and
`haproxy_allow_upgrade: true` must upgrade and reload. 26.04 is a fresh install with defaults. A project template
with `haproxy_config_diff: false` is rendered in `side_effect`.

## Installation

```yaml
# requirements.yml
roles:
  - name: pastir.haproxy
    src: git+git@github.com:Pastir/ansible-role-haproxy.git
    version: main  # or a tag
```

```bash
ansible-galaxy install -r requirements.yml
```

## License

MIT
