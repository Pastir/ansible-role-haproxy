# Changelog

## 2.1.0 — 2026-11

### Added
- `haproxy_allow_upgrade` (default `false`): an installed `haproxy-awslc` of another version, or the plain
  `haproxy` package, is not changed implicitly. The guard runs before the key, the apt source, `python3-debian`
  or packages are touched, so a refused run leaves the host as it was. A fresh install is not affected.
- `haproxy_config_template`: a project can supply its own full `haproxy.cfg.j2` (still validated with `haproxy -c`).
- `haproxy_config_diff` (default `true`): set to `false` to keep secrets (`stats auth`, userlists) out of
  `--diff` output.
- Molecule: `prepare` installs an older held version from a legacy `haproxy.list` on noble; converge checks the
  guard and the upgrade, `side_effect` renders a project template with the diff disabled.

### Changed
- A package version change (with `haproxy_allow_upgrade: true`) notifies `Reload HAProxy`: the master re-executes
  the new binary without dropping connections. The pinned version may also be a downgrade in that mode.
- `galaxy.yml` version follows the changelog (2.1.0).

## 2.0.0 — 2026-11

### BREAKING
- `haproxy_version` is an exact version (`"3.4.7"`), not a pattern (`"3.4.*"`). The package is held.
- The repository is added as `/etc/apt/sources.list.d/haproxy.sources` (deb822). The old `haproxy.list` is removed.
- Tags `haproxy_install` / `haproxy_config` were added (`install` / `configure` still work).

### Fixed
- Validation ran in a handler **after** the file was written. A broken config stayed on disk and was
  restarted. Now `validate:` checks the file before it replaces the current one.
- `restarted` dropped connections. Now the role reloads.
- The default `ssl-default-bind-ciphers` was rendered over several lines, which made the config invalid.
- `stats: [{socket: ...}]` was rendered as a Python dict. List items that are dicts are now rendered as
  `key sub value`.

### Added
- `raw` lines in a section, dict values, `daemon`/`ca-base`/`crt-base` in the default global section.
- Molecule (podman) for noble and resolute: reload keeps the master PID, an invalid config is rejected.
