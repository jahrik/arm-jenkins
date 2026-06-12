# AGENTS.md

Ansible playbooks bootstrapping the 2018 Jenkins-on-SBC CI node and cluster hosts. Historical infra repo: the `arm-*` repos now build on GitHub Actions, not this Jenkins. Not an image repo.

## Commands

```bash
ansible-playbook --syntax-check playbook.yml
ansible-playbook playbook.yml --tags jenkins   # java, docker, ansible, cluster
```

## CI

`build.yml`: syntax-check on PR and main. No release job.

## Quirks

- Tasks target Ubuntu 18.04-era hosts (python2, openjdk-8, apt_key) — kept as-is; bar is "parses and documented", not a rewrite.
- `docs/BUILD.md` is the original long-form writeup; keep it.
- `inventory.ini` names real cluster hosts, grouped by arch.
