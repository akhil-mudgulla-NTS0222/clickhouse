# ClickHouse Stress-Test Deployment (Ansible)

Ansible project that dynamically provisions an Ubuntu host and deploys a pinned,
stable ClickHouse server for stress / concurrency testing.

> **Scope note:** This is intentionally scoped for a disposable stress-test box
> — **no firewall scoping** and **no Ansible Vault** (plain credentials). Do not
> reuse as-is for a production / internet-facing deployment.

## Layout

```
ansible/
├── ansible.cfg                       # inventory path + lab-friendly defaults
├── inventory.ini                     # target host (fill in; password auth)
├── site.yml                          # runs both plays in order
├── group_vars/
│   └── clickhouse.yml                # version pin, ports, app user, limits
├── playbooks/
│   ├── 01-host-prep.yml              # Play 1 — host provisioning
│   └── 02-clickhouse-deploy.yml      # Play 2 — ClickHouse deployment
└── templates/
    ├── config.d/network.xml.j2       # listen host + ports + data path
    └── users.d/app-user.xml.j2       # non-default application user
```

## Design decisions

- **Two plays, cleanly separated.** Host prep (apt, prerequisites, ulimits) is
  independent and re-runnable without touching the deployment.
- **Password-based SSH, hardcoded in `inventory.ini`.** For a disposable lab
  box, `ansible_password` and `ansible_become_password` live in the inventory
  (no Vault, by design). Requires `sshpass` on the control node. Because the
  password is plaintext, keep the file out of version control.
- **Deterministic version pin.** `clickhouse_version` in `group_vars` installs
  an exact apt version (`24.8` LTS line) — re-runs are reproducible. Bump that
  one value to upgrade.
- **Override files, not shipped config.** All customization lands in `config.d/`
  and `users.d/` drop-ins; the packaged `config.xml`/`users.xml` stay untouched.
- **Idempotent restarts.** Config templates `notify` a restart handler, so the
  service restarts only when a config file actually changes.
- **Host-port exposure.** Both HTTP (8123) and native (9000) are exposed on the
  host for stress-tooling flexibility.

## Usage

0. One-time on the control node: `sudo apt-get install -y sshpass`
   (needed for Ansible password auth).
1. Edit `inventory.ini`:
   - `ansible_host` → target IP
   - `ansible_user` → SSH user with sudo rights
   - `ansible_password` → SSH login password
   - `ansible_become_password` → sudo password (same as login if applicable;
     delete the line if the user has passwordless sudo)
2. Review `group_vars/clickhouse.yml` (version, ports, app user/password).
3. Run everything (no password prompts — credentials come from the inventory):

   ```
   cd ansible
   ansible-playbook site.yml
   ```

   Or a single phase:

   ```
   ansible-playbook site.yml --tags host-prep
   ansible-playbook site.yml --tags deploy
   ```

## Validation

The deploy play finishes with a built-in health check: it waits for the HTTP
port, then runs `SELECT 1` as the application user and fails the run unless the
server returns `1`.

Manual check from any client afterwards:

```
curl "http://<host>:8123/?query=SELECT%20version()" \
  --user "stress:stress_pass"
```
