Pritunl VPN Server Install (Ubuntu 24.04 + MongoDB 8.0)

Overview
- Installs Pritunl with MongoDB 8.0 on Ubuntu 24.04 (noble) following the official docs: https://docs.pritunl.com/docs/installation#other-providers-ubuntu-2404
- Includes manual steps and an Ansible playbook with robust key handling and idempotency.

Repository Setup
- Sensitive files such as real inventories (`ansible/inventory.ini`, `ansible/inventories/**/hosts.ini`), raw password files, and `ips.txt` are ignored via `.gitignore`.
- Copy the provided samples before running anything:
  - `cp inventory.sample.ini ansible/inventory.ini`
  - `cp ips.sample.txt ips.txt`
- Fill in the copied files with real host information and keep them out of version control.

Prerequisites
- Ubuntu 24.04 LTS (noble). Verify with `lsb_release -cs` → `noble`.
- Root SSH access.
- Outbound HTTPS to fetch repositories.

Manual Install (Ubuntu 24.04)
1) Base tools and keyring dir
   sudo apt-get update
   sudo apt-get install -y curl gnupg ca-certificates lsb-release apt-transport-https
   sudo install -d -m 0755 /etc/apt/keyrings

2) MongoDB 8.0 repo + key (noble)
   curl -fsSL https://pgp.mongodb.com/server-8.0.asc | \
     sudo gpg --dearmor -o /etc/apt/keyrings/mongodb-server-8.0.gpg
   echo "deb [ signed-by=/etc/apt/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
     sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list >/dev/null

3) Pritunl repo + key (noble)
   # Import the Pritunl repo signing key from Ubuntu keyserver
   TMPDIR=$(mktemp -d); GNUPGHOME="$TMPDIR"; export GNUPGHOME
   gpg --keyserver hkps://keyserver.ubuntu.com --recv-keys 7AE645C0CF8E292A
   gpg --armor --export 7AE645C0CF8E292A | \
     sudo gpg --dearmor -o /etc/apt/keyrings/pritunl.gpg
   rm -rf "$TMPDIR"
   echo "deb [signed-by=/etc/apt/keyrings/pritunl.gpg] https://repo.pritunl.com/stable/apt noble main" | \
     sudo tee /etc/apt/sources.list.d/pritunl.list >/dev/null

4) Install packages
   sudo apt-get update
   sudo apt-get install -y mongodb-org pritunl openvpn wireguard wireguard-tools

5) Enable services
   sudo systemctl enable --now mongod
   sudo systemctl enable --now pritunl

6) Get setup key (use in web UI)
   sudo pritunl setup-key

7) Firewall (optional if UFW is enabled)
   sudo ufw allow OpenSSH
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw allow 1194/udp

8) Web setup
- Open https://SERVER_PUBLIC_IP/ in a browser (self‑signed cert warning is expected)
- Paste the setup key, set the admin username/password
- Add an organization and a user
- Create a server, set port/protocol, attach to an organization, and start the server
- Download user profile and import into your VPN client (Pritunl/OpenVPN client)

Troubleshooting
- If `pritunl default-password` errors with a Mongo URI message during first run, proceed with the web setup using the setup key. The daemon runs and the UI works; you can set the admin there.

Ansible Playbook (24.04 + MongoDB 8.0)

File: ansible/playbooks/install_pritunl_ubuntu2404.yml

Run:
- Using inventory alias `vpn_server`:
  ansible-playbook -i ansible/inventory.ini ansible/playbooks/install_pritunl_ubuntu2404.yml

- Or with the production inventory file:
  ansible-playbook -i ansible/inventories/production/hosts.ini ansible/playbooks/install_pritunl_ubuntu2404.yml

- Run as `ubuntu` user with sudo (become root):
  ansible-playbook -i ansible/inventory.ini -u ubuntu -b ansible/playbooks/install_pritunl_ubuntu2404.yml

Outputs (captured by Ansible):
- Pritunl setup key (for first‑time web setup)

Notes
- This playbook asserts the OS release is Ubuntu 24.04 (noble). If your host is on Ubuntu 22.04 (jammy), upgrade to 24.04 or use a 22.04 playbook with MongoDB 7.0.

K3s Cluster Install (3 masters + 3 workers)

Overview
- Best-practice Ansible layout with roles, group_vars, host_vars.
- Uses per-node K3s config templates (provided under `templates/`) converted to Jinja2 and parameterized with Vault-managed token.
- Installs K3s `{{ k3s_version }}` binary, runs installer with `INSTALL_K3S_SKIP_DOWNLOAD=true`.

Inventory
- `ansible/inventory.ini` and `ansible/inventories/production/hosts.ini` include:
  - `k3s_masters`: 10.10.0.26, 10.10.0.133, 10.10.0.7 (aliases: k3s-master-01..03)
  - `k3s_workers`: 10.10.0.98, 10.10.0.15, 10.10.0.202 (aliases: k3s-worker-01..03)

Variables
- Global defaults: `ansible/group_vars/all.yml` (version, paths, defaults).
- Secret token: `ansible/group_vars/all/vault.yml` (encrypted with Ansible Vault).
- Per-host template mapping: `ansible/host_vars/<host>.yml` via `k3s_config_template`.

Vault Setup
1) Create and encrypt the Vault file (replace with your UUID):
   cp ansible/group_vars/all/vault.yml.sample ansible/group_vars/all/vault.yml
   $EDITOR ansible/group_vars/all/vault.yml   # set vault_k3s_token: <uuid>
   ansible-vault encrypt ansible/group_vars/all/vault.yml

2) To run with Vault:
   - Ask for password each run:
     ansible-playbook -i ansible/inventory.ini ansible/playbooks/install_k3s.yml --ask-vault-pass
   - Or use a vault password file:
     ansible-playbook -i ansible/inventory.ini ansible/playbooks/install_k3s.yml --vault-password-file ~/.vault_pass.txt

Run K3s Install
- Activate venv if using fish:
  source .venv/bin/activate.fish

- Install masters then workers (single command installs both by groups):
  ansible-playbook -i ansible/inventory.ini ansible/playbooks/install_k3s.yml --ask-vault-pass

What it does
- Downloads K3s binary to `/usr/local/bin/k3s` for the specified version.
- Downloads `get.k3s.sh` and installs server/agent services if not already present.
- Renders `/etc/rancher/k3s/config.yaml` from node-specific templates with `token` injected from Vault.
- Enables and starts `k3s` (masters) and `k3s-agent` (workers) systemd services.

Notes
- Templates replicated under `ansible/roles/k3s_common/templates/` from your `templates/` dir and now accept `{{ k3s_token }}`.
- There’s a discrepancy in the Worker IPs in AGENTS.md vs templates; inventory follows the templates/ips.txt (10.10.0.98/15/202).
- To fetch kubeconfig after cluster is up (optional):
  sudo cat /etc/rancher/k3s/k3s.yaml   # on a master (e.g., 10.10.0.26)
  # then set KUBECONFIG locally
