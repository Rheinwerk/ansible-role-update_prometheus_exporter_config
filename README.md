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
| `listen_port` | yes | - | Port the exporter listens on |
| `listen_address` | no | `127.0.0.1` | Address the exporter binds to |
| `extra_args` | no | `[]` | List of additional command-line arguments |
| `binary_dir` | no | - | Path to the binary directory (omit for default location) |

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
```

## How It Works

For each configured exporter, the role:

1. Creates a systemd drop-in directory at `/etc/systemd/system/<name>.service.d/`
2. Deploys an `override.conf` that redefines the `ExecStart` directive
3. Reloads systemd if the configuration changed

## License

MIT

## Author

Rheinwerk
