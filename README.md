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

Instead of an authkey, you can also let TSDProxy authenticate with an [OAuth client](https://almeidapaulopt.github.io/tsdproxy/docs/advanced/tailscale/) by setting `tsdproxy_tailscale_client_id` and `tsdproxy_tailscale_client_secret` (and usually `tsdproxy_tailscale_tags`).

### Upgrading from TSDProxy v1

This role installs TSDProxy v2, whose configuration file format is incompatible with v1. The role takes care of the file it renders, but anything you add to it yourself has to follow the new format, and so do your services' labels and proxy lists. See the [upstream upgrade guide](https://almeidapaulopt.github.io/tsdproxy/docs/upgrading/from-v1/) for the details. In short:

- **Configuration keys are case-sensitive and mostly camelCase now** (`authKey`, `authKeyFile`, `controlUrl`, `targetHostname`, `defaultProxyProvider`, `proxyAccessLog`, `dataDir`, ...), and TSDProxy refuses to start when it finds a key it does not know - including the old all-lowercase ones. Update anything you pass in through `tsdproxy_additional_providers`, `tsdproxy_extra_config` or `tsdproxy_configuration_extension_yaml`. The role fails early, before touching the running instance, when it finds a v1 key there.
- **`files` was renamed to `lists`**, so `tsdproxy_config_files` is now `tsdproxy_config_lists`, and the list files themselves have a new format (see [Via Proxy list](#via-proxy-list)).
- **Port labels changed.** `tsdproxy.container_port`, `tsdproxy.scheme`, `tsdproxy.tlsvalidate` and `tsdproxy.funnel` still work, but have been superseded by `tsdproxy.port.<index>` (see [Via container labels](#via-container-labels)).
- **The dashboard requires authentication** - see [The dashboard](#the-dashboard).

### Access to the container socket

TSDProxy discovers the services it should proxy by watching the container daemon, so it needs to be able to talk to it. By default the role bind-mounts `/var/run/docker.sock` into the container read-only. Point `tsdproxy_docker_endpoint` somewhere else to change that, and set `tsdproxy_docker_endpoint_is_unix_socket` to `false` if you are using a TCP endpoint (in which case nothing is mounted).

**`tsdproxy_gid` has to be the group that owns the socket.** The container runs as `tsdproxy_uid:tsdproxy_gid`, and the socket is usually `root:docker` mode `0660`, so a `tsdproxy_gid` that is not the socket's group cannot open it. TSDProxy then logs `permission denied` and cannot discover any service. The `--privileged` flag the role passes does not help with this: a non-root uid gets no effective capabilities from it.

If you would rather not hand a container the real socket, put something like [ansible-role-container-socket-proxy](https://github.com/mother-of-all-self-hosting/ansible-role-container-socket-proxy) in front of it and point `tsdproxy_docker_endpoint` at that instead.

### Where the Tailscale authkey ends up

The role renders `tsdproxy_tailscale_authkey` into `{{ tsdproxy_config_path }}/tsdproxy.yaml` in clear text, owned by `tsdproxy_uid:tsdproxy_gid` with mode `0660`. It is not written to the systemd unit and not logged.

`tsdproxy_tailscale_authkeyfile` is the alternative: TSDProxy reads the key out of a file at startup, so the key itself never has to appear in your playbook's variables or in the rendered configuration. The file has to be readable from inside the container - use `tsdproxy_container_additional_mounts` to put it there.

### The dashboard

TSDProxy serves a dashboard and an API on `tsdproxy_container_http_port` (default `8080`) inside the container. The role does not publish that port on the host unless you set `tsdproxy_container_http_host_bind_port` (e.g. `127.0.0.1:8080`).

Since TSDProxy v2.2.0 the dashboard and the API require authentication: a Tailscale identity, or an API key (`apiKey` / `apiKeyFile`, which you can set via `tsdproxy_configuration_extension_yaml`). Note, however, that **when TSDProxy runs in a container it always lets requests from loopback and private (RFC 1918) addresses in** (`adminAllowLocalhost`), with full admin rights. Any container that shares a network with TSDProxy, and anything that can reach the published port, can therefore use the dashboard and the API. Publish the port on a loopback address only, if at all.

### Add a new Service

This proxy creates for each service a own machine in the Tailscale network, without creating each time a sidecar container. To add a new service, you have to make sure that the service and proxy are in a same container network. You can do this by adding the proxy to the network of the service or the other way round.

TSDProxy then finds the service's address on that network by probing it, because the role enables `tryDockerInternalNetwork` (`tsdproxy_docker_try_internal_network`). Without that, TSDProxy v2 would try to reach the service through the host (`tsdproxy_docker_target_hostname`) on the service's port, which does not work for services that do not publish their ports.

```yaml
tsdproxy_container_additional_networks_custom:
  - YOUR-SERVICE-NETWORK
# OR
YOUR-SERVICE_container_additional_networks_custom:
  - "{{ tsdproxy_container_network }}"
```

The next step is to add the service to the proxy.

#### Via container labels

Container labels are passed to Docker as a label file, so they are written as `key=value` lines:

```yaml
YOUR-SERVICE_container_labels_additional_labels: |
  tsdproxy.enable=true
  tsdproxy.port.1=443/https:8080/http
```

`tsdproxy.port.1` makes the service available via HTTPS on port 443 of its Tailscale machine, and proxies to port 8080 of the container via HTTP. Without it, TSDProxy uses the first port the container exposes.

The following labels are optional. Please read the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/providers/docker/) for more information, e.g. about options such as `tailscale_funnel` or `no_tlsvalidate`.

```yaml
  tsdproxy.name=my-service
  tsdproxy.autodetect=false
  tsdproxy.proxyprovider=providername
  tsdproxy.ephemeral=false
  tsdproxy.port.2=80/http->https://my-service.your-tailnet.ts.net
```

#### Via Proxy list

An alternative way to add a service to the proxy is to use proxy lists. Please read the [official TSDProxy documentation](https://almeidapaulopt.github.io/tsdproxy/docs/providers/lists/) for more information.

Define the lists with the `tsdproxy_config_lists` variable:

```yaml
tsdproxy_config_lists:
  critical:
    filename: /config/critical.yaml
    defaultProxyProvider: default
    defaultProxyAccessLog: true
```

and put the list file into the config folder (`tsdproxy_config_path`, most likely `/mash/tsdproxy/config/`), which is mounted at `/config` in the container. This is possible manually or by using [AUX-Files](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/services/auxiliary.md). The list file has to exist before TSDProxy starts, or TSDProxy refuses to start. A list file looks like this:

```yaml
nas:
  ports:
    443/https:
      targets:
        - http://192.168.1.2:5000
```

Note that the role renders its own `tsdproxy.yaml` into that same folder, because that is the only path TSDProxy reads its configuration from. Do not edit that file by hand - it is overwritten on every run. Use the role's variables, or `tsdproxy_configuration_extension_yaml`, instead.

## Testing

This role has a [Molecule](https://ansible.readthedocs.io/projects/molecule/) test suite. See [`molecule/README.md`](molecule/README.md) for how to run it, and - importantly - for what it can and cannot tell you: exercising TSDProxy's actual purpose needs a real Tailscale authkey and control plane, which CI cannot be given, so the suite deliberately stops one step short of that and says so.

## Development

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```
