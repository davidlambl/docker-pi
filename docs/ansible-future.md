# Ansible Conversion Notes

This document captures how the current manual runbook (README.md) maps to Ansible roles
and tasks for future automation.

## Inventory

```ini
[pi]
pi5 ansible_host=<PI5_IP> ansible_user=<your_user>

# Cold spare (uncomment when a second Pi is available)
; pi5-spare ansible_host=<SPARE_IP> ansible_user=<your_user>
```

## Role mapping

Each Phase 3 bootstrap step becomes a discrete role:

| README section              | Ansible role          | Key modules                              |
|-----------------------------|-----------------------|------------------------------------------|
| System update + timezone    | `base`                | `apt`, `timezone`, `unattended_upgrades` |
| Install Docker              | `docker`              | `get_url` + `shell` (get.docker.com)     |
| User + docker group         | `docker` (continued)  | `user`                                   |
| Disable systemd-resolved    | `resolved`            | `lineinfile`, `file`, `systemd`          |
| UFW rules                   | `ufw`                 | `ufw` module                             |
| Clone repo                  | `deploy`              | `git`, `template`, `docker_compose`      |

## Templatized files

- **`docker-compose.yml`** stays as-is (Compose reads `.env`; no Jinja needed).
- **`.env`** rendered from `templates/env.j2` using `host_vars` or `group_vars`:
  ```jinja2
  PUID={{ puid }}
  PGID={{ pgid }}
  TZ={{ timezone }}
  MONGO_INITDB_ROOT_USERNAME={{ mongo_root_user }}
  MONGO_INITDB_ROOT_PASSWORD={{ mongo_root_pass }}
  MONGO_USER={{ mongo_user }}
  MONGO_PASS={{ mongo_pass }}
  MONGO_DBNAME={{ mongo_dbname }}
  MONGO_AUTHSOURCE={{ mongo_authsource }}
  MEM_LIMIT={{ unifi_mem_limit }}
  MEM_STARTUP={{ unifi_mem_startup }}
  ```
- **`AdGuardHome.yaml`** rendered from `templates/AdGuardHome.yaml.j2` with variables
  for `bind_host`, `bind_port`, upstream DNS servers, and filtering lists.

## Backup role

The cron-based backup from Phase 7 becomes a `backup` role:
- `cron` module for the nightly tar job
- `template` for the backup script
- Optional `rsync` task to push archives to a cold-spare host

## Playbook outline

```yaml
- hosts: pi
  become: true
  roles:
    - base
    - docker
    - resolved
    - ufw
    - deploy
    - backup
```

## Vault

Sensitive values (`MONGO_PASS`, `MONGO_INITDB_ROOT_PASSWORD`) stored in
`group_vars/pi/vault.yml`, encrypted with `ansible-vault`.

## Testing

- Use Molecule + Docker driver for role unit tests (run against a Debian arm64 image).
- Integration test: spin up a full Vagrant/UTM VM, run the playbook, verify ports are
  listening and containers are healthy.
