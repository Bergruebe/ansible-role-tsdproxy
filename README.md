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

It is mandatory to set following variables:

```yaml
tsdproxy_uid: '' # the uid the container runs as
tsdproxy_gid: '' # the gid the container runs as - see "Access to the container socket" below

tsdproxy_tailscale_authkey: '' # OR
tsdproxy_tailscale_authkeyfile: '' # use this to load authkey from file. If this is defined, Authkey is ignored
```

### Access to the container socket

TSDProxy discovers the services it should proxy by watching the container daemon, so it needs to be able to talk to it. By default the role bind-mounts `/var/run/docker.sock` into the container read-only. Point `tsdproxy_docker_endpoint` somewhere else to change that, and set `tsdproxy_docker_endpoint_is_unix_socket` to `false` if you are using a TCP endpoint (in which case nothing is mounted).

**`tsdproxy_gid` has to be the group that owns the socket.** The container runs as `tsdproxy_uid:tsdproxy_gid`, and the socket is usually `root:docker` mode `0660`, so a `tsdproxy_gid` that is not the socket's group cannot open it. TSDProxy then logs `permission denied`, panics and exits, and - because the systemd service restarts it - crash-loops. The `--privileged` flag the role passes does not help with this: a non-root uid gets no effective capabilities from it.

If you would rather not hand a container the real socket, put something like [ansible-role-container-socket-proxy](https://github.com/mother-of-all-self-hosting/ansible-role-container-socket-proxy) in front of it and point `tsdproxy_docker_endpoint` at that instead.

### Where the Tailscale authkey ends up

The role renders `tsdproxy_tailscale_authkey` into `{{ tsdproxy_config_path }}/tsdproxy.yaml` in clear text, owned by `tsdproxy_uid:tsdproxy_gid` with mode `0660`. It is not written to the systemd unit and not logged.

`tsdproxy_tailscale_authkeyfile` is the alternative: TSDProxy reads the key out of a file at startup, so the key itself never has to appear in your playbook's variables or in the rendered configuration. The file has to be readable from inside the container - use `tsdproxy_container_additional_mounts` to put it there.

### Add a new Service

This proxy creates for each service a own machine in the Tailscale network, without creating each time a sidecar container. To add a new service, you have to make sure that the service and proxy are in a same container network. You can do this by adding the proxy to the network of the service or the other way round.

```yaml
tsdproxy_container_additional_networks_custom:
  - YOUR-SERVICE-NETWORK
# OR
YOUR-SERVICE_container_additional_networks_custom:
  - "{{ tsdproxy_container_network }}"
```

The next step is to add the service to the proxy.

#### Via docker labels

```yaml
YOUR-SERVICE_container_labels_additional_labels: |
  tsdproxy.enable: "true"
  tsdproxy.container_port: 8080
```

Following labels are optional, please read the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/docker/) for more information.

```yaml
  tsdproxy.name: "my-service"
  tsdproxy.autodetect: "false"
  tsdproxy.proxyprovider: "providername"
  tsdproxy.ephemeral: "false"
  tsdproxy.funnel: "false"
```

#### Via Proxy list

An alternative way to add a service to the proxy is to use Proxy files. Please read the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/files/) for more information.

You will need to use the `tsdproxy_config_files` variable and add your proxy list file into the config folder, most likely `/mash/tsdproxy/config/`. This is possible manually or by using [AUX-Files](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/services/auxiliary.md).

Note that the role renders its own `tsdproxy.yaml` into that same folder, because that is the only path TSDProxy reads its configuration from. Do not edit that file by hand - it is overwritten on every run. Use the role's variables, or `tsdproxy_configuration_extension_yaml`, instead.

## Testing

This role has a [Molecule](https://ansible.readthedocs.io/projects/molecule/) test suite. See [`molecule/README.md`](molecule/README.md) for how to run it, and - importantly - for what it can and cannot tell you: exercising TSDProxy's actual purpose needs a real Tailscale authkey and control plane, which CI cannot be given, so the suite deliberately stops one step short of that and says so.

## Development

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```
