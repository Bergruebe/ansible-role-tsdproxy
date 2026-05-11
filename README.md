<!--
SPDX-FileCopyrightText: 2023 Slavi Pantaleev
SPDX-FileCopyrightText: 2024 Bergrübe
SPDX-FileCopyrightText: 2025, 2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# TSDProxy Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [TSDProxy](https://almeidapaulopt.github.io/tsdproxy/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook/).

## Usage

At least one authentication method should be configured:

```yaml
# AuthKey-based auth
tsdproxy_tailscale_authkey: "" # OR
tsdproxy_tailscale_authkeyfile: "" # if set, authkey is ignored

# OAuth-based auth
tsdproxy_tailscale_clientid: ""
tsdproxy_tailscale_clientsecret: ""
```

The role mounts the Docker socket by default and uses `tsdproxy_docker_endpoint: unix:///var/run/docker.sock`.
By usage of the **MASH-Playbook** with [com.devture.ansible.role.container_socket_proxy](https://github.com/devture/com.devture.ansible.role.container_socket_proxy) (default), the container will use the proxy. 
If you use a TCP endpoint, set:

```yaml
tsdproxy_docker_endpoint: "tcp://DOCKER_HOST:2376"
tsdproxy_docker_endpoint_is_unix_socket: false
```

### Add a new Service

TSDProxy creates one Tailscale node per service, without a per-service sidecar.
To expose a service, TSDProxy and the service must be able to reach each other over Docker networking.
Usually, both containers should share at least one network:

```yaml
tsdproxy_container_additional_networks_custom:
  - YOUR-SERVICE-NETWORK
# OR
YOUR-SERVICE_container_additional_networks_custom:
  - "{{ tsdproxy_container_network }}"
```

If you proxy via Docker host published ports instead of a shared network, set:

```yaml
tsdproxy_docker_target_hostname: host.docker.internal
```

The role will automatically add `--add-host=host.docker.internal:host-gateway` to the TSDProxy container in this mode.

Then add labels to the target service.

#### Via Docker labels (v2)
See the official [documentation](https://almeidapaulopt.github.io/tsdproxy/docs/providers/docker-reference/) for more details.
```yaml
YOUR-SERVICE_container_labels_additional_labels: |
  tsdproxy.enable: "true"
  tsdproxy.port.1: "443/https:8080/http"
```

Optional labels:

```yaml
  tsdproxy.name: "my-service"
  tsdproxy.autodetect: "false"
  tsdproxy.proxyprovider: "providername"
  tsdproxy.ephemeral: "false"
  tsdproxy.port.2: "80/http->https://my-service.example.ts.net"
  tsdproxy.port.3: "8443/https:8443/https, no_tlsvalidate"
  tsdproxy.port.4: "443/https:8080/http, tailscale_funnel"
```

Deprecated v1 labels (`tsdproxy.container_port`, `tsdproxy.scheme`, `tsdproxy.tlsvalidate`, `tsdproxy.funnel`) should not be used anymore.

#### Via Lists provider (v2)

Configure lists in `tsdproxy.yaml` via `tsdproxy_config_lists`:

```yaml
tsdproxy_config_lists:
  critical:
    filename: /config/critical.yaml
    defaultProxyProvider: default
    defaultProxyAccessLog: true
```

Example `critical.yaml`:

```yaml
nas:
  ports:
    443/https:
      targets:
        - http://192.168.1.10:5001

redirect-home:
  ports:
    80/http:
      targets:
        - https://example.com
      isRedirect: true
```

The old role variable `tsdproxy_config_files` is still accepted as a backward-compatible alias to `tsdproxy_config_lists`, but new setups should use `tsdproxy_config_lists`.

For details, see the official v2 docs:

- [Getting Started](https://almeidapaulopt.github.io/tsdproxy/docs/getting-started/)
- [Docker Labels Reference](https://almeidapaulopt.github.io/tsdproxy/docs/providers/docker-reference/)
- [Lists](https://almeidapaulopt.github.io/tsdproxy/docs/providers/lists/)
- [Changelog](https://almeidapaulopt.github.io/tsdproxy/docs/changelog/)

## Development

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```
