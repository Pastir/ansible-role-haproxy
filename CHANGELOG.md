# Changelog

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
