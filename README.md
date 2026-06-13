# arm-jenkins

[![Build](https://github.com/jahrik/arm-jenkins/actions/workflows/build.yml/badge.svg)](https://github.com/jahrik/arm-jenkins/actions/workflows/build.yml)

Ansible playbooks that bootstrapped Jenkins CI on a Libre Renegade SBC and prepared the Pi/Odroid swarm cluster (java, docker, jenkins user, ntp). The `arm-*` image repos have since moved from this Jenkins to GitHub Actions.

Full build writeup with photos: [docs/BUILD.md](docs/BUILD.md).

## Usage

```bash
ansible-playbook playbook.yml                   # everything
ansible-playbook playbook.yml --tags jenkins    # or java, docker, ansible, cluster
```

Hosts live in `inventory.ini` (grouped by arch); settings in `group_vars/`. DockerHub login comes from `DOCKER_USER`/`DOCKER_PASS`/`DOCKER_EMAIL` env vars.
