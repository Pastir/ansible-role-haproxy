# Ansible Role: pastir.haproxy

Installs HAProxy from the official HAProxy Technologies "performance" APT repository and manages `haproxy.cfg`.

- The package is `haproxy-awslc` (built against AWS-LC). It provides and conflicts with the plain `haproxy` package.
- The exact version is pinned and held (`haproxy_version`).
- The config is **validated before it is written** (`haproxy -c`), so a broken config never replaces the
  running one.
- Changes are applied with a **hitless reload**, so existing connections are not dropped.

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
| `haproxy_repo_enabled` | `true` | Add the repository |
| `haproxy_gpg_key_url` / `haproxy_gpg_key_path` | HAProxy key | |
| `haproxy_cfg_file` | `/etc/haproxy/haproxy.cfg` | |
| `haproxy_custom_config_file` | `""` | Complete config file from the controller. If set, the template is not used |
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
PID, and rejection of an invalid config.

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
