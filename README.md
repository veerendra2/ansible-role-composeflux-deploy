# ansible-role-composeflux-deploy

[![Lint](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/lint.yml/badge.svg)](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/lint.yml)
[![Release](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/release.yml/badge.svg)](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/release.yml)
[![Ansible Galaxy](https://img.shields.io/badge/Ansible%20Galaxy-composeflux__deploy-blue)](https://galaxy.ansible.com/ui/standalone/roles/veerendra2/composeflux_deploy/)
[![GitHub stars](https://img.shields.io/github/stars/veerendra2/ansible-role-composeflux-deploy)](https://github.com/veerendra2/ansible-role-composeflux-deploy/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/veerendra2/ansible-role-composeflux-deploy)](https://github.com/veerendra2/ansible-role-composeflux-deploy/issues)
[![GitHub last commit](https://img.shields.io/github/last-commit/veerendra2/ansible-role-composeflux-deploy)](https://github.com/veerendra2/ansible-role-composeflux-deploy/commits/main)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Ansible role to deploy [ComposeFlux](https://github.com/veerendra2/composeflux) — a GitOps continuous deployment tool for Docker Compose.

## Requirements

- Docker with Compose plugin installed on target hosts
- Debian/Ubuntu target OS
- [`community.docker`](https://galaxy.ansible.com/community/docker) collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Role Variables

See [defaults/main.yml](./defaults/main.yml)

## Example Usage

```yaml
- hosts: servers
  become: true
  roles:
    - role: veerendra2.composeflux_deploy
      vars:
        # Required
        composeflux_git_repo_url: "git@github.com:user/infra-repo.git"
        composeflux_stack_path: "servers/myhost"

        # SSH key — required to clone the repo; pick one
        composeflux_ssh_key_local_path: "~/.ssh/composeflux_deploy"
        # composeflux_ssh_key_secret_ref: "SSH_PRIVATE_KEY"

        # Age-encrypted secrets stored in the git repo
        composeflux_age_passphrase: "{{ vault_age_passphrase }}"

        # Infisical remote secrets
        # composeflux_infisical_client_id: "{{ vault_infisical_client_id }}"
        # composeflux_infisical_client_secret: "{{ vault_infisical_client_secret }}"
        # composeflux_infisical_environment: "prod"
        # composeflux_infisical_project_id: "project-id"

        # Force sync (re-fetch secrets without git changes): --tags sync
```
