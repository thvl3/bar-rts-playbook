# Ansible Playbook for BAR Replay Server

> [!WARNING]
> This repo is work in progress, not finished yet.

This is an [Ansible](https://en.wikipedia.org/wiki/Ansible_(software)) playbook for setting up the BAR Replay Server (bar-rts.com).

## Architecture

The playbook deploys the following containerized services using Podman Quadlet:

- **PostgreSQL** - Database for replay metadata, maps, and user data
- **Redis** - Caching and session storage
- **bar-db** - REST API backend serving replay data
- **bar-db-processor** - Background service processing replays and maps
- **bar-live-services** - Nuxt.js frontend application
- **Caddy** - Reverse proxy with automatic HTTPS

## Prerequisites

- Ansible 2.15+
- Target server running Debian 12+ (Trixie recommended)
- SSH access to target server

## Usage

### Production Deployment

1. Configure inventory in `inventory/` directory
2. Create and encrypt vault secrets:
   ```bash
   cp group_vars/prod/vault.yml.example group_vars/prod/vault.yml
   # Edit vault.yml with your secrets
   ansible-vault encrypt group_vars/prod/vault.yml
   ```
3. Run the playbook:
   ```bash
   ansible-playbook -l prod play.yml --ask-vault-pass
   ```

### Local Testing

We use [Incus](https://linuxcontainers.org/incus/) for local testing. Make sure you have it installed and initialized following [the official getting started docs](https://linuxcontainers.org/incus/docs/main/tutorial/first_steps/).

#### Setup

Create a new container and initialize it via cloud-init:

```bash
touch .incus-integration-on && \
chmod 0600 test.ssh.key && \
incus launch images:debian/trixie/cloud bar-replay-test < test.incus.yml && \
incus exec bar-replay-test -- cloud-init status --wait
```

Test that ansible can connect:

```bash
ansible dev -m shell -a 'uname -a'
```

Add local domain to `/etc/hosts`:

```bash
ansible ,localhost -b -K -m lineinfile -a "path=/etc/hosts regexp='.*bar-mon.*' line='$(ansible-inventory --host test | jq -r '.ansible_host') replay.bar-mon.local'"
```

#### Run Playbook

```bash
ansible-playbook -l dev play.yml
```

#### Access Services

- Frontend: https://replay.bar-mon.local
- API: https://replay.bar-mon.local/api/

#### SSH Access

```bash
ssh -i test.ssh.key ansible@$(ansible-inventory --host test | jq -r '.ansible_host')
```

Or enter container shell directly:

```bash
incus exec bar-replay-test -- /bin/bash
```

#### Cleanup

```bash
incus stop bar-replay-test && incus delete bar-replay-test
```

## Configuration

### Required Secrets (Production)

| Variable | Description |
|----------|-------------|
| `vault_postgres_password` | PostgreSQL database password |
| `vault_bar_db_github_token` | GitHub token for balance change processing |
| `vault_bar_db_lobby_username` | Teiserver bot username |
| `vault_bar_db_lobby_password` | Teiserver bot password |
| `vault_bar_db_google_sheets_id` | Google Sheets ID for map lists |
| `vault_bar_db_google_sheets_api_key` | Google Sheets API key |

### Configurable Variables

See `group_vars/all.yml` for all available configuration options.

## Related Repositories

- [bar-db](https://github.com/beyond-all-reason/bar-db) - Backend source code
- [bar-live-services](https://github.com/beyond-all-reason/bar-live-services) - Frontend source code
- [ansible-monitoring](https://github.com/beyond-all-reason/ansible-monitoring) - Reference architecture
