# Socket-proxy Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Optionally run `wollomatic/socket-proxy` beside ComposeFlux and let ComposeFlux talk to Docker over `tcp://socket-proxy:2375` with a minimal allowlist.

**Architecture:** One extra compose service and one private bridge network, both rendered by `templates/compose.yml.j2`. ComposeFlux loses the raw socket mount and gets `DOCKER_HOST` from `.env`. The sync task is narrowed to the `composeflux` service so the proxy stays up during a one-shot sync.

**Tech Stack:** Ansible (Jinja2 templates), Docker Compose v2, `wollomatic/socket-proxy:1`.

**Spec:** short design approved in session 2026-11-21 (this repo, no separate spec file).

## Global Constraints

- New vars: `composeflux_socket_proxy_enabled: false`, `composeflux_socket_proxy_version: "1"`.
- Proxy is opt-in only; default render must stay byte-compatible in behaviour with today (direct socket mount, no extra service/network).
- No published proxy ports; proxy reachable only on the project's private network.
- Allowed API set is fixed (verified against ComposeFlux v0.20.1): HEAD `/_ping`; GET `/v1..{1,2}/(version|_ping|containers/.*|images/.*|networks.*|volumes.*|distribution/.*)` and `/_ping`; POST `containers/create`, `containers/{id}/(start|stop|restart|kill|rename|wait)`, `images/create`, `networks/create`, `volumes/create`, `volumes/prune`, `images/prune`, `build/prune`, `build`; DELETE `containers/{id}`, `networks/{id}`.
- Proxy service hardening: `read_only: true`, `cap_drop: [ALL]`, `no-new-privileges`, `mem_limit: 64M`, `user: "0:0"`, `-allowfrom=composeflux`, socket mounted `:ro`.
- `yamllint .` and `ansible-lint` must pass (line-length 120).

## Review Focus

- A user enables the proxy but also configures `composeflux_external_networks`: ComposeFlux must end up on both the proxy network and the external ones.
- `--tags sync` with the proxy enabled: stopping the whole project would kill the proxy, so the one-shot sync would have no Docker socket. Stop/start must target only `composeflux`.
- Proxy disabled: `compose.yml` must still mount `/var/run/docker.sock` and define no `socketproxy` network.
- GET allowlist must not accidentally allow unrelated paths; every alternative stays inside the grouped `(/v1..{1,2}/...)` pattern or the exact `/_ping`.

---

### Task 1: Variables and `DOCKER_HOST`

**Files:**
- Modify: `defaults/main.yml` (new section after "Networking")
- Modify: `templates/env.j2`

**Interfaces:**
- Produces: `composeflux_socket_proxy_enabled` (bool), `composeflux_socket_proxy_version` (str), `DOCKER_HOST` env line used by Tasks 2 and 4.

- [ ] **Step 1: Add vars to `defaults/main.yml`**

```yaml
# ---------------------------------------------------------------------------
# Docker Socket Proxy
# ---------------------------------------------------------------------------

# Run wollomatic/socket-proxy beside ComposeFlux and connect over TCP instead of
# mounting /var/run/docker.sock into ComposeFlux. Only the Docker API calls
# ComposeFlux needs are allowed. No ports are published.
composeflux_socket_proxy_enabled: false

# socket-proxy image tag. Upstream publishes "1" (rolling major) and exact
# versions such as "1.14.0"; there is no "latest" tag.
composeflux_socket_proxy_version: "1"
```

- [ ] **Step 2: Add `DOCKER_HOST` to `templates/env.j2`**

Insert after the `CONFIG_FILE` line:

```jinja
{% if composeflux_socket_proxy_enabled %}
{{ env_line("DOCKER_HOST", "tcp://socket-proxy:2375") -}}
{% endif %}
```

- [ ] **Step 3: Verify default render has no `DOCKER_HOST` and no syntax error**

Run:
```bash
ansible localhost -c local -m ansible.builtin.template \
  -a "src=templates/env.j2 dest=/tmp/env-default" \
  -e composeflux_git_repo_url=x -e composeflux_stack_path=x
grep -c DOCKER_HOST /tmp/env-default
```
Expected: `0`

- [ ] **Step 4: Verify enabled render has the line**

Run the same command with `-e composeflux_socket_proxy_enabled=true` and `dest=/tmp/env-proxy`, then:
```bash
grep -n 'DOCKER_HOST' /tmp/env-proxy
```
Expected: `DOCKER_HOST="tcp://socket-proxy:2375"`

- [ ] **Step 5: Commit**

```bash
git add defaults/main.yml templates/env.j2
git commit -m "feat: add socket-proxy variables and DOCKER_HOST"
```

### Task 2: Compose template

**Files:**
- Modify: `templates/compose.yml.j2`

**Interfaces:**
- Consumes: both vars from Task 1.
- Produces: services `composeflux` and `socket-proxy`, network `socketproxy` — exercised by Task 4's end-to-end run.

- [ ] **Step 1: Replace the whole template**

Order matters: both service blocks, then the top-level `networks:` key. Write:

```jinja
---
services:
  composeflux:
    image: ghcr.io/veerendra2/composeflux:{{ composeflux_version }}
    container_name: composeflux
    env_file:
      - .env
    volumes:
{% if not composeflux_socket_proxy_enabled %}
      - /var/run/docker.sock:/var/run/docker.sock:ro
{% endif %}
{% if composeflux_git_clone_path | length > 0 %}
      - {{ composeflux_git_clone_path }}:{{ composeflux_git_clone_path }}
{% endif %}
{% if composeflux_ssh_key_local_path | length > 0 %}
      - ./deploy_key:{{ composeflux_ssh_key_container_path }}:ro
{% endif %}
{% if composeflux_ssh_known_hosts | length > 0 or composeflux_ssh_known_hosts_local_path | length > 0 %}
      - ./known_hosts:/etc/composeflux/known_hosts:ro
{% endif %}
{% for vol in composeflux_extra_volumes %}
      - {{ vol }}
{% endfor %}
    restart: unless-stopped
{% if composeflux_external_networks | length > 0 or composeflux_socket_proxy_enabled %}
    networks:
{% for net in composeflux_external_networks %}
      - {{ net }}
{% endfor %}
{% if composeflux_socket_proxy_enabled %}
      - socketproxy
{% endif %}
{% endif %}
{% if composeflux_socket_proxy_enabled %}

  socket-proxy:
    image: wollomatic/socket-proxy:{{ composeflux_socket_proxy_version }}
    container_name: socket-proxy
    restart: unless-stopped
    read_only: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges
    mem_limit: 64M
    # The socket is root:docker; root avoids hardcoding a host-specific GID.
    user: "0:0"
    command:
      - "-listenip=0.0.0.0"
      - "-allowfrom=composeflux"
      - '-allowHEAD=/_ping'
      - '-allowGET=(/v1\..{1,2}/(version|_ping|containers/.*|images/.*|networks.*|volumes.*|distribution/.*)|/_ping)'
      - '-allowPOST=(/v1\..{1,2}/(containers/create|containers/[a-zA-Z0-9_.-]+/(start|stop|restart|kill|rename|wait)|images/create|networks/create|volumes/create|volumes/prune|images/prune|build/prune|build))'
      - '-allowDELETE=(/v1\..{1,2}/(containers/[a-zA-Z0-9_.-]+|networks/[a-zA-Z0-9_.-]+))'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - socketproxy
{% endif %}

{% if composeflux_external_networks | length > 0 or composeflux_socket_proxy_enabled %}
networks:
{% for net in composeflux_external_networks %}
  {{ net }}:
    external: true
{% endfor %}
{% if composeflux_socket_proxy_enabled %}
  socketproxy:
    driver: bridge
{% endif %}
{% endif %}
```

- [ ] **Step 2: Confirm the disabled render is unchanged**

Before editing, render the current template:
```bash
ansible localhost -c local -m ansible.builtin.template \
  -a "src=templates/compose.yml.j2 dest=/tmp/compose-before.yml" \
  -e composeflux_git_repo_url=x -e composeflux_stack_path=x \
  -e '{"composeflux_ssh_key_local_path":""}'
```
After editing, render again to `/tmp/compose-after.yml` with the same args, then `diff /tmp/compose-before.yml /tmp/compose-after.yml`.
Expected: no differences.

- [ ] **Step 3: Verify all four renders**

Run for each combination (disabled; enabled; enabled + `composeflux_external_networks='["proxy"]'`; disabled + external networks):

```bash
ansible localhost -c local -m ansible.builtin.template \
  -a "src=templates/compose.yml.j2 dest=/tmp/compose-<case>.yml" \
  -e composeflux_git_repo_url=x -e composeflux_stack_path=x \
  [-e composeflux_socket_proxy_enabled=true] \
  [-e '{"composeflux_external_networks":["proxy"]}']
python3 -c 'import sys,yaml;d=yaml.safe_load(open(sys.argv[1]));print(sorted(d["services"]), sorted(d.get("networks",{})))' /tmp/compose-<case>.yml
```
Expected: disabled → `['composeflux']` and no networks key; enabled → both services and `socketproxy`; enabled+external → both and both networks; disabled+external → composeflux only, external network only. Also `grep -c docker.sock` must be `1` when enabled and `1` when disabled (mounted on exactly one service), and the proxy service must have no `ports:`.

- [ ] **Step 4: Commit**

```bash
git add templates/compose.yml.j2
git commit -m "feat: add optional socket-proxy service to compose template"
```

### Task 3: Keep the proxy up during `--tags sync`

**Files:**
- Modify: `tasks/sync.yml`

**Interfaces:**
- Consumes: rendered project from Task 2.
- Produces: nothing for later tasks.

- [ ] **Step 1: Scope stop and start to `composeflux`**

Add `services: [composeflux]` under the module args of both `community.docker.docker_compose_v2` tasks (`state: stopped` and `state: present`):

```yaml
        services:
          - composeflux
```

- [ ] **Step 2: Static check**

Run: `ansible-lint tasks/sync.yml`
Expected: exit 0, no findings.

- [ ] **Step 3: Commit**

```bash
git add tasks/sync.yml
git commit -m "fix: stop only composeflux during sync so socket-proxy stays up"
```

### Task 4: README and end-to-end verification

**Files:**
- Modify: `README.md` (new section before "Tags")

**Interfaces:**
- Consumes: everything above.

- [ ] **Step 1: Document the feature**

```markdown
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
```

- [ ] **Step 2: Run repo lint**

Run: `yamllint . && ansible-lint`
Expected: both exit 0.

- [ ] **Step 3: End-to-end run of the rendered project**

Render the enabled compose file (Task 2 Step 4) into a temp dir, replace the ComposeFlux image with a local build of ComposeFlux v0.20.1, point `GIT_REPO_URL` at a local `git daemon` repo with one stack, then:

```bash
docker compose -f /tmp/e2e/compose.yml --env-file /tmp/e2e/.env up -d
docker compose -f /tmp/e2e/compose.yml logs --tail 20
docker compose -f /tmp/e2e/compose.yml stop composeflux
docker compose -f /tmp/e2e/compose.yml run --rm composeflux sync
docker compose -f /tmp/e2e/compose.yml start composeflux
docker logs socket-proxy | grep -i denied
```
Expected: stack deployed on first `up`; one-shot `sync` succeeds while socket-proxy stays running; no denied requests in proxy logs.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: document optional socket-proxy integration"
```
