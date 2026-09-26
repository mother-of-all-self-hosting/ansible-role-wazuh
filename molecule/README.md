<!--
SPDX-FileCopyrightText: 2018-2026 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
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

Here is a synopsis of what a successful run proves. Check the scenario itself for details about what are exactly checked.

- The the running stack is the version the role pins
- The indexer is running the configuration this role rendered
- The indexer's security database is the one this role rendered
- The manager is running the configuration this role rendered
- `API_PASSWORD` reached the manager process
- The dashboard reached the indexer
- `wazuh_dashboard_http_port` reaches the process
- The Traefik labels, the additional volumes and the extra container arguments are real
- The manager-side features are installed

What it does not prove:

- Upgrade works
- An agent can actually enrol
- The rules and integrations do anything

## Scenarios

Currently one testing scenario is available.

### `default`

The whole role: manager, indexer, dashboard and the certificate generator, with Traefik labels, agent enrollment, a remote agent configuration, a custom rule, custom integrations, additional volumes, extra container arguments and a custom `ossec.conf` XPath replacement.

Nearly every value the scenario sets differs from the role's own default, so that a value observed on the running stack can be told apart from one the container image or the role default produced.

## Running

By default it is configured to run the scenario on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default
```

## Note about the case where idempotence step fails

If the scenario is executed locally on a workstation running several privileged systemd containers in parallel, the idempotence step may fail due to a `vm.max_map_count` value on the computer.

To avoid it, run `molecule converge` followed by `molecule verify` instead of `molecule test`, or run this scenario on its own.

If you run this suite locally, note the `vm.max_map_count` value first and restore it afterwards:

```bash
sudo sysctl -n vm.max_map_count                  # before
molecule test --scenario-name default
sudo sysctl -w vm.max_map_count=<the old value>  # after
```

Please check the section below if you are interested in the details about why the idempotence step fails.

### Details

`tasks/install_indexer.yml` raises `vm.max_map_count` to 262144, because OpenSearch refuses to start below the value. Please note that `vm.max_map_count` is **not a namespaced kernel parameter**, and also that the scenario's container is privileged, so that write reaches the kernel of the machine running the suite and nothing puts it back.

On a GitHub Actions runner that is both necessary (the default is too low for the indexer) and harmless (the runner is thrown away). On the other hand, on a workstation it is a permanent change to a machine-wide setting, and it can *lower* the value if your distribution already sets a higher one.

`prepare.yml` records the value it found and `verify.yml` asserts the value the role left is at least 262144, reporting both. That is the honest boundary: the suite proves the parameter ended up high enough for OpenSearch, and it tells you what it was before, but it cannot prove the role was the one that raised it on a machine that was already above the threshold.

The same shared-kernel property has a second consequence. `geerlingguy/docker-ubuntu2604-ansible` ships systemd 259, which ships `/usr/lib/sysctl.d/55-map-count.conf` setting `vm.max_map_count=1048576`. `systemd-sysctl.service` applies that at container boot, and because the parameter is not namespaced, it lands on the shared kernel.

On a GitHub Actions runner that happens once during `create` and the role overwrites it during `converge`, so `molecule test`'s idempotence step passes. On the other hand, on a workstation running several privileged systemd containers in parallel, every one of them that boots resets the parameter to 1048576, so `ansible.posix.sysctl` finds a value it did not set, and the idempotence step fails.
