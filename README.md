# Ansible Role: update_prometheus_exporter_config

Configure Prometheus exporters via systemd drop-in overrides.

## Description

This role creates systemd drop-in configuration files to customize Prometheus exporter services. It allows you to configure the listen address, port, and additional arguments for each exporter without modifying the original service files.

## Requirements

- Ansible >= 2.12
- Target systems: Debian (Bookworm) or Ubuntu (Jammy)
- Prometheus exporter services must already be installed

## Role Variables

### `_prometheus_exporter_configs`

A list of exporter configurations. Default: `[]`

Each item supports the following properties:

| Property | Required | Default | Description |
|----------|----------|---------|-------------|
| `name` | yes | - | Service name (e.g., `systemd_exporter`) |
| `listen_port` | no | - | Port the exporter listens on. If set, `--web.listen-address` is added automatically. Omit for exporters that don't support this flag. |
| `listen_address` | no | `127.0.0.1` | Address the exporter binds to (only used with `listen_port`) |
| `extra_args` | no | `[]` | List of additional command-line arguments |
| `env_vars` | no | `{}` | Dict of environment variables |
| `binary_dir` | no | - | Path to the binary directory (omit for default location) |
| `config_content` | no | - | Content for `/etc/<name>/config.yml` config file |

## Example Playbook

```yaml
- hosts: servers
  roles:
    - role: update_prometheus_exporter_config
      vars:
        _prometheus_exporter_configs:
          - name: node_exporter
            listen_port: "9100"

          - name: systemd_exporter
            listen_port: "9558"
            extra_args:
              - "--systemd.collector.unit-include=ssh\\.service"

          - name: custom_exporter
            listen_address: "0.0.0.0"
            listen_port: "9999"
            binary_dir: /opt/custom/bin
            extra_args:
              - "--log.level=debug"

          - name: process-exporter
            listen_port: "9256"
            extra_args:
              - "--config.path=/etc/process-exporter/config.yml"
            config_content: |
              process_names:
                - name: "{{.Comm}}"
                  comm:
                    - nginx
                    - postgres

          # No listen_port — openvpn_exporter does not support --web.listen-address
          - name: openvpn_exporter
            extra_args:
              - "--openvpn.listen-address=127.0.0.1:9176"
              - "--openvpn.status-files=/etc/openvpn/openvpn-status.log"
```

## How It Works

For each configured exporter, the role:

1. If `config_content` is defined, creates `/etc/<name>/config.yml` with the specified content
2. Creates a systemd drop-in directory at `/etc/systemd/system/<name>.service.d/`
3. Deploys an `override.conf` that redefines the `ExecStart` directive
4. Reloads systemd if the configuration changed

## x509-certificate-exporter

This role always appends a config entry for
[x509-certificate-exporter](https://github.com/enix/x509-certificate-exporter)
(cert-expiry metrics) to `_prometheus_exporter_configs`, ahead of the main
loop -- there's no need to list it yourself. It's expected to already be
installed, disabled, by the baseimage on every consumer of this role.

The rendered `config_content` is a single `kind: file` source whose
`paths` are read from a cert-glob file (default `/etc/cert_exp_time_globs`,
one glob pattern per line, `#`-comments and blank lines ignored) --
computed fresh each run, so it always reflects whatever cert paths other
roles have contributed to that file by the time this role runs. Lines
ending in `.properties` are skipped: those wrap a PKCS#12 path + passphrase
(e.g. APNS certs) that the exporter can't parse directly from a Java
properties file.

Set `update_prometheus_exporter_config_manage_x509_certificate_exporter:
false` to opt out (e.g. a consumer of this role that doesn't install the
exporter). Other tunables:

```yaml
x509_certificate_exporter_listen_address: "127.0.0.1:9793"
x509_certificate_exporter_refresh_interval: "1h"
x509_certificate_exporter_cert_glob_file: "/etc/cert_exp_time_globs"
```

## License

MIT

## Author

Rheinwerk
