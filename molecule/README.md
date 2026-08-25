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

TSDProxy discovers a service by watching Docker for containers labelled `tsdproxy.enable=true`. Everything up to *registering the resulting node with the control plane* works without a tailnet, and was checked by hand against `docker.io/almeidapaulopt/tsdproxy:1.4.7`: given a labelled container, TSDProxy finds it through the socket, auto-detects its target URL, connects to it on the container network, creates the proxy and starts a `tsnet` server for it. Only the final step - authenticating to `controlurl` with the authkey - needs the real thing.

That last step cannot be made to fail gracefully, either. With an unusable authkey, TSDProxy 1.4.7 does not log an error and carry on; the `tsnet` server never produces a listener and TSDProxy panics on a nil listener inside `proxymanager.(*Proxy).start`, exits 2, and - under the role's `Restart=always` - crash-loops. A scenario that started a labelled container would therefore be testing a panic.

**So no container in this scenario carries the `tsdproxy.enable=true` label.** The suite stops one step short of proxying, and the dashboard's proxy listing is asserted to be served, not to be populated.

Wiring up a local [Headscale](https://headscale.net/) as the control plane would move the ceiling, at the cost of turning this into a two-application scenario. That has not been attempted.

### What a green run does prove

- **That the role's configuration file is the one TSDProxy reads.** This is the important one. TSDProxy loads `/config/tsdproxy.yaml`, and when it finds nothing there it *generates a default and runs with it*. That generated default is near-identical to this role's template - the template was originally copied from it - so a scenario using the role's own default values could not tell the two apart. Every value this scenario sets is therefore deliberately different from both the role default and TSDProxy's fallback: log level `trace` rather than `info`, HTTP port `8099` rather than `8080`, `proxyaccesslog` false rather than true, `log.json` true rather than false. The verification asserts TSDProxy logged `loading configuration from: /config/tsdproxy.yaml` and did *not* log `Generating default configuration`, and then asserts on values that only the role's file could have supplied.
- **That `tsdproxy_container_http_port` reaches the process**, not just the `-p` flag: the readiness endpoint is probed on the published port, and nothing would answer there if TSDProxy were listening on its compiled-in 8080.
- **That `tsdproxy_configuration_extension_yaml` survives into the running process.** The extension turns on JSON logging, which the role's template hardcodes to false; structured log lines cannot come from anywhere else.
- **That `tsdproxy_container_additional_mounts` renders as a real `--mount`.** `tsdproxy_tailscale_authkeyfile` points at a file that only exists inside the container because of one, and TSDProxy reads that file while loading its configuration and refuses to start when it cannot.
- **That TSDProxy really has the Docker socket.** `Default Network found` is logged only after a successful network listing through the socket. Without access, TSDProxy logs `permission denied`, panics and exits 2 - which is also why `tsdproxy_gid` is set to the gid that owns the socket rather than to anything convenient (see below).
- **That the running TSDProxy is the version `defaults/main.yml` pins**, taken from the `Starting server` log line of the live process and cross-checked against the image's `org.opencontainers.image.version` label.
- **That the role's other plumbing lands**: the `--label-file` label, the `--env-file` variable, the container's uid:gid, the read-only Docker socket mount, and the removal of the stale configuration file that older versions of this role left in the data path (`prepare.yml` seeds one).

### Things worth knowing before editing

- **`Restart=always` makes `systemctl is-active` nearly worthless here**, because the failure mode this role most plausibly has - TSDProxy unable to use the Docker socket - is a crash loop, and a crash loop reports `active`. The verification asserts `NRestarts` is 0, records the systemd `InvocationID`, reads the journal scoped to *that invocation* (`journalctl _SYSTEMD_INVOCATION_ID=...`, not `--unit`, which would also return starts that have since been replaced), and re-checks at the end that the invocation has not changed.
- **`tsdproxy_gid` must be the gid of the group that owns the Docker socket.** The socket is `root:docker` mode 0660 and the container runs as a non-root uid. `--privileged`, which the role passes, does *not* help: a non-root uid gets no effective capabilities from it, so `CAP_DAC_OVERRIDE` is not in play. This was checked directly - privileged with a non-`docker` gid is denied; unprivileged with the `docker` gid works.
- **TSDProxy reads no environment variables at all.** It has no viper or envconfig layer - just a `-config` flag defaulting to `/config/tsdproxy.yaml` and the YAML file. The `env` file this role writes is therefore inert as far as the application is concerned; the scenario asserts it reaches Docker, and nothing more.
- **The image's built-in `HEALTHCHECK` is a compiled-in probe of `127.0.0.1:8080`**, so it reports `unhealthy` for any other `tsdproxy_container_http_port`. Nothing in this role or this suite consults it; the readiness endpoint is probed directly instead.

## Scenarios

Currently these testing scenarios are available:

### `default`

A single TSDProxy instance with no proxied services, configured entirely through the role's variables and with every value chosen to be distinguishable from what TSDProxy would pick for itself.

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
