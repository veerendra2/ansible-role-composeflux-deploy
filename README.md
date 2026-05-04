# ansible-role-composeflux-deploy

Ansible role to deploy [ComposeFlux](https://github.com/veerendra2/composeflux) - a GitOps continuous deployment tool for Docker Compose.

## Requirements

- Docker with compose plugin installed on target servers
- Debian/Ubuntu target OS
- `community.docker` collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Role Variables

See [`defaults/main.yml`](defaults/main.yml) for all available variables with descriptions and examples.

### Minimal Example

```yaml
composeflux_git_repo_url: "git@github.com:user/infra-repo.git"
composeflux_stack_path: "servers/myhost"
```

### Full Example with Infisical

```yaml
composeflux_version: "v0.1.0"
composeflux_git_repo_url: "git@github.com:user/infra-repo.git"
composeflux_stack_path: "servers/myhost"
composeflux_git_clone_path: "/opt/docker/compose-stack"
composeflux_git_branch: "main"
composeflux_image_update_schedule: "0 4 * * 1"

composeflux_secrets_provider: "infisical"
composeflux_git_deploy_key_secret_ref: "SSH_PRIVATE_KEY"
composeflux_infisical_client_id: "{{ vault_infisical_client_id }}"
composeflux_infisical_client_secret: "{{ vault_infisical_client_secret }}"
composeflux_infisical_environment: "prod"
composeflux_infisical_project_id: "project-id"
composeflux_infisical_secret_path: "/home-servers/myhost"

composeflux_metrics_enabled: true
composeflux_external_networks:
  - monitoring
```

### Deploy Key from Disk

```yaml
composeflux_deploy_key_src: "~/.ssh/composeflux_deploy"
composeflux_git_clone_path: "/opt/docker/compose-stack"
```

## Usage

### Deploy / Upgrade

```bash
ansible-playbook site.yml
```

### Force Sync (re-fetch secrets without git changes)

```bash
ansible-playbook site.yml --tags sync
```

## Example Playbook

```yaml
- hosts: servers
  become: true
  roles:
    - role: composeflux_deploy
      vars:
        composeflux_git_repo_url: "git@github.com:user/infra-repo.git"
        composeflux_stack_path: "servers/myhost"
```

## License

MIT
