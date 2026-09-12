# ansible-role-composeflux-deploy

[![Lint](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/lint.yml/badge.svg)](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/lint.yml)
[![Release](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/release.yml/badge.svg)](https://github.com/veerendra2/ansible-role-composeflux-deploy/actions/workflows/release.yml)
[![Ansible Galaxy](https://img.shields.io/badge/Ansible%20Galaxy-composeflux__deploy-blue)](https://galaxy.ansible.com/ui/standalone/roles/veerendra/composeflux_deploy/)
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

| Variable                                | Default                      | Example                                    | Description                                                                 |
| --------------------------------------- | ---------------------------- | ------------------------------------------ | --------------------------------------------------------------------------- |
| `composeflux_git_repo_url`              | `""`                         | `"git@github.com:user/repo.git"`           | **Required.** Git repo SSH URL                                              |
| `composeflux_stack_path`                | `""`                         | `"servers/myhost"`                         | **Required.** Path to stacks dir relative to repo root                      |
| `composeflux_version`                   | `"latest"`                   | `"v0.5.0"`                                 | ComposeFlux image tag                                                       |
| `composeflux_deploy_path`               | `"/opt/docker/composeflux"`  | `"/srv/composeflux"`                       | Host directory for compose.yml and .env                                     |
| `composeflux_git_branch`                | `""`                         | `"main"`                                   | Git branch to track (ComposeFlux default: `main`)                           |
| `composeflux_git_interval`              | `""`                         | `"10m"`                                    | Git polling interval (ComposeFlux default: `5m`)                            |
| `composeflux_git_clone_path`            | `""`                         | `"/opt/compose-stack"`                     | Clone path inside container (ComposeFlux default: `/opt/compose-stack`)     |
| `composeflux_config_file`               | `""`                         | `"stack.yml"`                              | Stack config filename (ComposeFlux default: `stack.yml`)                    |
| `composeflux_deploy_key_src`            | `""`                         | `"~/.ssh/composeflux_deploy"`              | Local SSH key path to copy to host (Option A)                               |
| `composeflux_git_ssh_key_path`          | `"/.ssh/composeflux_id_rsa"` | `"/.ssh/id_rsa"`                           | Mount path of SSH key inside container                                      |
| `composeflux_git_deploy_key_secret_ref` | `""`                         | `"SSH_PRIVATE_KEY"`                        | Secret name/ID in secrets manager holding the SSH key (Option B)            |
| `composeflux_age_passphrase`            | `""`                         | `"{{ vault_age_passphrase }}"`             | Passphrase for Age-encrypted secret files in git                            |
| `composeflux_bitwarden_access_token`    | `""`                         | `"{{ vault_bw_token }}"`                   | Bitwarden machine account access token                                      |
| `composeflux_bitwarden_organization_id` | `""`                         | `"org-uuid"`                               | Bitwarden organization ID                                                   |
| `composeflux_bitwarden_project_id`      | `""`                         | `"project-uuid"`                           | Bitwarden project ID                                                        |
| `composeflux_bitwarden_api_url`         | `""`                         | `"https://bitwarden.example.com/api"`      | Bitwarden API URL (self-hosted only)                                        |
| `composeflux_bitwarden_identity_url`    | `""`                         | `"https://bitwarden.example.com/identity"` | Bitwarden identity URL (self-hosted only)                                   |
| `composeflux_infisical_client_id`       | `""`                         | `"{{ vault_infisical_id }}"`               | Infisical Universal Auth client ID                                          |
| `composeflux_infisical_client_secret`   | `""`                         | `"{{ vault_infisical_secret }}"`           | Infisical Universal Auth client secret                                      |
| `composeflux_infisical_environment`     | `""`                         | `"prod"`                                   | Infisical environment slug                                                  |
| `composeflux_infisical_project_id`      | `""`                         | `"project-id"`                             | Infisical project ID                                                        |
| `composeflux_infisical_secret_path`     | `""`                         | `"/home-servers/myhost"`                   | Infisical secret path (comma-separated paths supported)                     |
| `composeflux_infisical_site_url`        | `""`                         | `"https://infisical.example.com"`          | Infisical site URL (self-hosted only)                                       |
| `composeflux_image_update_schedule`     | `""`                         | `"0 4 * * 1"`                              | Cron for Docker image update checks (empty = disabled)                      |
| `composeflux_health_reconcile_interval` | `""`                         | `"5m"`                                     | Proactive stack health reconciliation interval (empty = disabled)           |
| `composeflux_prune_interval`            | `""`                         | `"24h"`                                    | Docker resource prune interval (`0` = disabled, ComposeFlux default: `24h`) |
| `composeflux_remove_orphans`            | `""`                         | `"false"`                                  | Remove orphan containers on deploy (ComposeFlux default: `true`)            |
| `composeflux_log_level`                 | `""`                         | `"debug"`                                  | Log level: `debug`, `info`, `warn`, `error`                                 |
| `composeflux_log_format`                | `""`                         | `"json"`                                   | Log format: `console` or `json`                                             |
| `composeflux_log_add_source`            | `""`                         | `"true"`                                   | Add source file/line to log records                                         |
| `composeflux_external_networks`         | `[]`                         | `["proxy", "monitoring"]`                  | Existing Docker networks to attach ComposeFlux to                           |
| `composeflux_extra_volumes`             | `[]`                         | `["./certs:/certs:ro"]`                    | Additional volume mounts                                                    |
| `composeflux_extra_env`                 | `{}`                         | `{MY_VAR: value}`                          | Additional environment variables                                            |

## Example Usage

```yaml
- hosts: servers
  become: true
  roles:
    - role: veerendra.composeflux_deploy
      vars:
        # Required
        composeflux_git_repo_url: "git@github.com:user/infra-repo.git"
        composeflux_stack_path: "servers/myhost"

        # Deploy key from local disk (Option A)
        composeflux_deploy_key_src: "~/.ssh/composeflux_deploy"

        # Age-encrypted secrets stored in the git repo
        composeflux_age_passphrase: "{{ vault_age_passphrase }}"

        # Or use Infisical for remote secrets (Option B)
        # composeflux_git_deploy_key_secret_ref: "SSH_PRIVATE_KEY"
        # composeflux_infisical_client_id: "{{ vault_infisical_client_id }}"
        # composeflux_infisical_client_secret: "{{ vault_infisical_client_secret }}"
        # composeflux_infisical_environment: "prod"
        # composeflux_infisical_project_id: "project-id"

        # Force sync (re-fetch secrets without git changes): --tags sync
```
