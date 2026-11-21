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

## Self-hosted Git hosts

The ComposeFlux image only trusts `github.com` and `gitlab.com`. To clone from any other SSH host (Gitea, Forgejo, self-hosted GitLab, or a server on a non-default port), give the role the host keys:

```bash
ssh-keyscan git.example.com > ~/.ssh/composeflux_known_hosts
```

```yaml
composeflux_ssh_known_hosts_local_path: "~/.ssh/composeflux_known_hosts"
# or inline:
# composeflux_ssh_known_hosts:
#   - "git.example.com ssh-ed25519 AAAA..."
#   - "git.example.com ssh-rsa AAAA..."
```

Do not pass `-t` to `ssh-keyscan`: a single key type can fail with `knownhosts: key mismatch`, because go-git negotiates its own host key algorithm. For a non-default port, scan with `-p 2222` and use an `ssh://git@git.example.com:2222/group/repo.git` URL.

The role mounts the file at `/etc/composeflux/known_hosts` and sets `SSH_KNOWN_HOSTS=/etc/ssh/ssh_known_hosts:/etc/composeflux/known_hosts`, so the built-in GitHub/GitLab trust is kept.

Verify the host fingerprint out of band before trusting a scanned key.

## Docker socket proxy

By default ComposeFlux mounts `/var/run/docker.sock`. To avoid handing it the raw socket, run
[`wollomatic/socket-proxy`](https://github.com/wollomatic/socket-proxy) alongside it:

```yaml
composeflux_socket_proxy_enabled: true
```

The role then starts a hardened `socket-proxy` container on a private bridge network with no
published ports, and ComposeFlux gets `DOCKER_HOST=tcp://socket-proxy:2375`. The allowlist is the
minimum ComposeFlux needs (containers, images, networks, volumes, build cache, image update checks,
pruning). Pick an image tag with `composeflux_socket_proxy_version` (default `1`; exact pins such as
`1.14.0` also work — upstream has no `latest` tag).

## Tags

| Tag      | Behaviour                                                                                  |
| -------- | ------------------------------------------------------------------------------------------ |
| `deploy` | Templates `.env` and `compose.yml`, then deploys or updates the ComposeFlux daemon.         |
| `sync`   | Forces a one-shot reconciliation **with the daemon stopped**, then starts the daemon again. |

