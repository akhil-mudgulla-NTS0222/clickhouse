# ClickHouse stress-test deployment

Ansible provisions an Ubuntu VM with Docker Engine and the Compose plugin, then deploys **only ClickHouse** as a container. The two plays can be rerun independently. Intended for a disposable lab VM: HTTP/native ports are exposed without TLS or firewall rules, and credentials are plaintext. Do not use this configuration on an untrusted network.

## Requirements

- Ubuntu VM with password SSH, sudo, internet access to Docker's Ubuntu APT repository and Docker Hub, and sufficient disk at `clickhouse_data_path`.
- Ansible and `sshpass` on the control machine (`sudo apt-get install -y ansible sshpass` on Ubuntu).
- `8123` and `9000` free on the VM. If a native ClickHouse service was installed separately, stop/disable it before deploying; this play does not uninstall it or remove its data.
- Review the pinned image tag and settings in `group_vars/clickhouse.yml` before deploying.

## Run in order

1. Edit `inventory.ini` with your VM IP, SSH user, SSH password and sudo password. Keep this plaintext file out of shared repositories.
2. From `ansible/`, check reachability: `ansible clickhouse -m ping`.
3. Run `ansible-playbook site.yml --syntax-check`, then `ansible-playbook site.yml` (not `--check` for the first install: check mode cannot install Docker or start a container).
4. Verify from the control machine: `curl --user 'stress:stress_pass' 'http://<VM_IP>:8123/?query=SELECT%201'`.

To rerun only one phase use `ansible-playbook site.yml --tags host-prep` or `--tags deploy`. The host play installs Docker; the deploy play renders `/opt/clickhouse-stress/compose.yml` and config overrides, starts the pinned image, then checks `SELECT 1`. Both protocols use **host networking** (no Docker NAT); the host data directory `/var/lib/clickhouse-stress` is bind-mounted into the container. The data survives container restarts and recreation. The bind-mounted directory is owned by the container's ClickHouse UID/GID, configured in group vars. Container `nofile`/`nproc` limits are set by Compose, not PAM limits on the host.

On the VM: `sudo docker compose -f /opt/clickhouse-stress/compose.yml ps`, `sudo docker compose -f /opt/clickhouse-stress/compose.yml logs --tail=100`, and `sudo docker compose -f /opt/clickhouse-stress/compose.yml restart` are useful for inspection. `down` removes the container **but not the host bind-mounted data**; restarting also does not reset state. For reproducible test runs, explicitly reset the test tables with SQL between runs, or intentionally clear the lab data directory only after a backup and with ClickHouse stopped. Do not treat `down -v` as a data reset for bind mounts.

The image is pinned to `24.8.14.39` for reproducible version selection; this is a tag, not an immutable digest. Image updates require changing the pin and redeploying. This setup does not provision the ingestion API.
