<!--
SPDX-FileCopyrightText: 2018-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://ansible.readthedocs.io/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems:
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## What the suite can and cannot tell you

Read this before changing anything under `molecule/`.

TSDProxy's entire purpose is to put a service on a [Tailscale](https://tailscale.com/) tailnet. Doing that for real needs a Tailscale pre-authentication key and a reachable control plane, neither of which CI can be given: an authkey is a credential that joins a network, and a test that used one would enrol throwaway machines into somebody's real tailnet on every run. So there is a hard ceiling on what this suite can assert, and it is worth being precise about where it sits.

### Where the ceiling is

TSDProxy discovers a service by watching Docker for containers labelled `tsdproxy.enable=true`. Everything up to *registering the resulting node with the control plane* works without a tailnet: given a labelled container, TSDProxy finds it through the socket, auto-detects its target URL on the container network, creates the proxy and starts a `tsnet` server for it. Only the final step - authenticating to `controlUrl` with the authkey - needs the real thing.

TSDProxy 1.x panicked at that last step when the authkey was unusable, so the suite used to leave labelled containers out altogether. TSDProxy 2.x fails it gracefully: the proxy's status becomes `Error` and the process carries on. **So the scenario now starts one labelled service** (`traefik/whoami`, on a network of its own, publishing no port) and asserts that TSDProxy discovered it and resolved its target to the service's address on the shared network. The proxy's status is shown in the summary, but not asserted on.

To keep that final step local, the scenario points `tsdproxy_controlurl` at a port nothing listens on, and sets `TS_NO_LOGS_NO_SUPPORT` so that the `tsnet` node does not upload its logs to `log.tailscale.com`. The fake authkey never leaves the machine.

Wiring up a local [Headscale](https://headscale.net/) as the control plane would move the ceiling, at the cost of turning this into a two-application scenario. That has not been attempted.

### What a green run does prove

- **That the role's configuration file is the one TSDProxy reads, and that TSDProxy accepts it.** This is the important one. TSDProxy loads `/config/tsdproxy.yaml`, and when it finds nothing there it *generates a default and runs with it*. That generated default is near-identical to this role's template - the template was originally copied from it - so a scenario using the role's own default values could not tell the two apart. Every value this scenario sets is therefore deliberately different from both the role default and TSDProxy's fallback: log level `trace` rather than `info`, HTTP port `8099` rather than `8080`, `proxyAccessLog` false rather than true, `log.json` true rather than false. The verification asserts TSDProxy logged `loading configuration` for `/config/tsdproxy.yaml` and did *not* log `generating default configuration`, and then asserts on values that only the role's file could have supplied. Since TSDProxy 2.x rejects unknown (e.g. TSDProxy v1-style) keys and exits, getting that far also shows the rendered file is in the format TSDProxy expects.
- **That `tsdproxy_container_http_port` reaches the process**, not just the `-p` flag: the readiness endpoint is probed on the published port, and nothing would answer there if TSDProxy were listening on its compiled-in 8080.
- **That `tsdproxy_configuration_extension_yaml` survives into the running process.** The extension turns on JSON logging, which the role's template hardcodes to false; structured log lines cannot come from anywhere else.
- **That `tsdproxy_container_additional_mounts` renders as a real `--mount`.** `tsdproxy_tailscale_authkeyfile` points at a file that only exists inside the container because of one, and TSDProxy reads that file while loading its configuration and refuses to start when it cannot.
- **That TSDProxy really has the Docker socket.** `Default Network found` is logged only after a successful network listing through the socket. Without access, TSDProxy logs `permission denied` - which is also why `tsdproxy_gid` is set to the gid that owns the socket rather than to anything convenient (see below).
- **That TSDProxy can reach a service the way this role sets it up.** The labelled service is only reachable on its own network, which TSDProxy joins via `tsdproxy_container_additional_networks_custom`, and only found there because of the role's `tsdproxy_docker_try_internal_network` default. Without either, the proxy's target would not be the service's address on that network (TSDProxy's own default would target `targetHostname`, which the scenario sets to a dummy `10.99.0.7`).
- **That the image's Docker `HEALTHCHECK` agrees**, on a port other than `8080`.
- **That the running TSDProxy is the version `defaults/main.yml` pins**, taken from the `Starting server` log line of the live process and cross-checked against the image's `org.opencontainers.image.version` label.
- **That the role's other plumbing lands**: the `--label-file` label, the `--env-file` variable, the container's uid:gid, the read-only Docker socket mount, and the removal of the stale configuration file that older versions of this role left in the data path (`prepare.yml` seeds one).

### Things worth knowing before editing

- **`Restart=always` makes `systemctl is-active` nearly worthless here**, because TSDProxy exits on a configuration it rejects (an unknown key, an unreadable authkey or list file), and the resulting crash loop reports `active`. The verification asserts `NRestarts` is 0, records the systemd `InvocationID`, reads the journal scoped to *that invocation* (`journalctl _SYSTEMD_INVOCATION_ID=...`, not `--unit`, which would also return starts that have since been replaced), and re-checks at the end that the invocation has not changed.
- **`tsdproxy_gid` must be the gid of the group that owns the Docker socket.** The socket is `root:docker` mode 0660 and the container runs as a non-root uid. `--privileged`, which the role passes, does *not* help: a non-root uid gets no effective capabilities from it, so `CAP_DAC_OVERRIDE` is not in play. This was checked directly - privileged with a non-`docker` gid is denied; unprivileged with the `docker` gid works.
- **TSDProxy configures itself from its YAML file, not from the environment.** It has a `-config` flag defaulting to `/config/tsdproxy.yaml`, and reads a few legacy environment variables only while generating a default configuration, which never happens with this role. The `env` file this role writes is therefore inert as far as TSDProxy's own settings are concerned (the scenario asserts it reaches Docker); only the Tailscale library TSDProxy embeds reads `TS_*` variables from it.
- **The image's built-in `HEALTHCHECK` has to wait for its 60-second interval** before it reports anything, which is why its assertion comes last. It probes the readiness endpoint on the port TSDProxy writes into `/data/.http-port`, so it works with any `tsdproxy_container_http_port`.

## Scenarios

Currently these testing scenarios are available:

### `default`

A single TSDProxy instance with one labelled service to discover (but no tailnet to register it with), configured entirely through the role's variables and with every value chosen to be distinguishable from what TSDProxy would pick for itself.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default
```
