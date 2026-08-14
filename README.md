# Ansible Role: HAProxy

[![Build Status](https://travis-ci.org/pastir/haproxy.svg?branch=master)](https://travis-ci.org/pastir/haproxy)
[![License](https://img.shields.io/badge/license-MIT%20License-brightgreen.svg)](https://opensource.org/licenses/MIT)

Ansible role for installing and configuring HAProxy - a high-performance load balancer and proxy server.

## Requirements

- Ansible 2.9 or higher
- Ubuntu 22.04 (Jammy), 24.04 (Noble), 26.04 (Resolute)

Packages are installed from the official HAProxy Technologies "performance"
APT repository (https://www.haproxy.com/downloads), not the vbernat PPA.
That repository ships the `haproxy-awslc` build (linked against AWS-LC
instead of OpenSSL), which provides/conflicts with the plain `haproxy`
package, so this role installs `haproxy-awslc`.

## Supported HAProxy Versions

- Jammy (22.04): 3.3.x, 3.4.x
- Noble (24.04): 3.2.x, 3.3.x, 3.4.x
- Resolute (26.04): 3.3.x, 3.4.x

## Role Variables

Main role variables are defined in `defaults/main.yml`:

```yaml
# HAProxy version
haproxy_major_version: "3.4"
haproxy_version: "3.4.*"

# Path to a custom HAProxy configuration file
haproxy_custom_config_file: ""

# HAProxy configuration
haproxy_config:
  - section: global
    log:
      - /dev/log    local0
      - /dev/log    local1 notice
    chroot: /var/lib/haproxy
    stats:
      - socket: /run/haproxy/admin.sock mode 660 level admin
      - timeout: 30s
    user: haproxy
    group: haproxy
    # SSL settings
    ssl-default-bind-ciphers: |
      ECDHE-ECDSA-AES128-GCM-SHA256:
      ECDHE-RSA-AES128-GCM-SHA256:
      ECDHE-ECDSA-AES256-GCM-SHA384:
      ECDHE-RSA-AES256-GCM-SHA384:
      ECDHE-ECDSA-CHACHA20-POLY1305:
      ECDHE-RSA-CHACHA20-POLY1305:
      DHE-RSA-AES128-GCM-SHA256:
      DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites: TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options: ssl-min-ver TLSv1.2 no-tls-tickets
```

## Usage

### Basic Installation

```yaml
- hosts: haproxy_servers
  roles:
    - role: pastir.haproxy
```

### Installation with Custom Configuration

```yaml
- hosts: haproxy_servers
  roles:
    - role: pastir.haproxy
      vars:
        haproxy_major_version: "3.2"
        haproxy_version: 3.2.*
        haproxy_config:
          - section: global
            log:
              - /dev/log    local0
              - /dev/log    local1 notice
            chroot: /var/lib/haproxy
            stats:
              - socket: /run/haproxy/admin.sock mode 660 level admin
              - timeout: 30s
            user: haproxy
            group: haproxy
```

### Installation with Custom Configuration File

```yaml
- hosts: haproxy_servers
  roles:
    - role: pastir.haproxy
      vars:
        haproxy_custom_config_file: "files/my_custom_haproxy.cfg"
```

## Tags

- `install` - HAProxy installation
- `configure` - HAProxy configuration

## License

MIT

## Author

Role created and maintained by [pastir](https://github.com/pastir)
