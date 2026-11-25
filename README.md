# UIAVM

**A modern control panel for LXC and Docker virtual servers, with multi-node clustering.**
Create and manage VPS across as many servers as you own, from one fast dashboard. Version 12.1.

*Made by reon*

---

## Contents

- [Features](#features)
- [How it fits together](#how-it-fits-together)
- [1. Install LXC (LXD) on your server](#1-install-lxc-lxd-on-your-server)
- [2. Install the UIAVM panel](#2-install-the-uiavm-panel)
- [3. Create your first VPS](#3-create-your-first-vps)
- [Docker or LXC?](#docker-or-lxc)
- [4. Add more nodes](#4-add-more-nodes)
- [Run the panel as a service](#run-the-panel-as-a-service)
- [HTTPS with nginx](#https-with-nginx)
- [Configuration](#configuration)
- [REST API](#rest-api)
- [Upgrading from HVM](#upgrading-from-hvm)
- [What's new in 12.1](#whats-new-in-121)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [Project layout](#project-layout)

---

## Features

- **Remote access links (Tmate and sshx).** From a VPS's Manage tab, get an SSH or web terminal link with one click. The tool is installed in the VPS automatically, and if the link expires you press Regenerate.
- **Real specs.** The VPS page reads the actual OS, kernel, CPU, memory limit and disk usage from inside the VPS (for Docker too).
- **Docker or LXC, your choice per VPS.** Pick a full-system LXC container or a Docker container from any Docker Hub image when you create a VPS. Both get live stats, a terminal, snapshots and port forwards.
- **Multi-node clusters.** Add a server with one command. New VPS go to the least-loaded online node, or pick one yourself.
- **Live dashboard.** Real CPU, memory, disk and network per VPS and per node, a cluster overview and an animated map of your nodes.
- **Browser terminal.** A real shell inside any VPS, including VPS on remote nodes.
- **VPS tools.** Start, stop, restart, freeze, reinstall (with live progress), snapshots with one-click restore, port forwards, notes, sharing with other users.
- **Accounts.** Public registration, per-user port quotas, 2FA with any authenticator app, API keys. Everyone can change their own username, email and password.
- **Admin control.** Create, edit, delete and promote any account, create more admins, and use "Sign in as" to see the panel as a user (logged).
- **Safety nets.** Auto-suspend for sustained high usage, node drain mode, offline alerts, hourly database backups, full activity log.
- **Self-contained.** Fonts, terminal and scripts are bundled, so the panel works without internet access to CDNs.

## How it fits together

```
                        ┌─────────────────────────┐
   your browser  ─────► │  UIAVM panel  (:5000)   │
                        │  Flask + Socket.IO + DB │
                        └───────┬─────────┬───────┘
                local lxc       │         │   HTTP :7071 (token auth)
                                ▼         ▼
                       ┌────────────┐  ┌────────────┐  ┌────────────┐
                       │ Local node │  │ Node: sg-01│  │ Node: de-01│  ...
                       │ LXD        │  │ agent + LXD│  │ agent + LXD│
                       └────────────┘  └────────────┘  └────────────┘
```

- The **panel** runs on one server. It manages its own LXD directly (the "Local Node").
- Every **extra node** runs a tiny agent (`uiavm_node.py`, Python standard library only) next to its own LXD and/or Docker.
- The panel talks to agents over HTTP on port **7071** using a per-node secret token. Agents only ever run `lxc` or `docker` commands as an argument list, never a shell.

---

## 1. Install LXC (LXD) on your server

UIAVM drives containers through **LXD**, the `lxc` command. You need it on the panel server (for the Local Node) and on every extra node. *(The node installer in step 4 installs it for you if it is missing, so on extra nodes you can skip this step.)*

> **Supported:** Ubuntu 22.04 / 24.04 (recommended) and other systemd Linux distributions that can run `snapd`. Use a real server or VM with root access. Containers share the host kernel and need features that OpenVZ-style VPS hosts usually don't allow.

### Ubuntu / Debian

```bash
# 1. snap (already present on Ubuntu)
sudo apt update && sudo apt install -y snapd

# 2. install LXD
sudo snap install lxd

# 3. initialise it with sane defaults (creates a storage pool called "default" and a NAT bridge "lxdbr0")
sudo lxd init --auto

# 4. check it works
lxc version
lxc storage list          # you should see a pool named "default"
```

Quick test (creates, lists and removes a container):

```bash
sudo lxc launch ubuntu:24.04 test
sudo lxc list
sudo lxc delete test --force
```

If `lxc` says *permission denied*, run it with `sudo`, or add your user to the group: `sudo usermod -aG lxd $USER` and log in again. The panel itself should run as **root** (or a user in the `lxd` group).

### Choose a storage backend (important for disk limits)

UIAVM sets a **disk size limit** on every VPS. The plain `dir` backend that `lxd init --auto` picks often **cannot enforce disk limits**, and creating a VPS then fails at the disk step. For real use, pick ZFS or btrfs:

```bash
sudo apt install -y zfsutils-linux

# Testing: a 100 GB file-backed pool
sudo lxd init --auto --storage-backend zfs --storage-create-loop 100 --storage-pool default

# Production: a dedicated disk (this ERASES /dev/sdb)
sudo lxd init --auto --storage-backend zfs --storage-create-device /dev/sdb --storage-pool default
```

Already ran `lxd init` with `dir`? Create a second pool and tell UIAVM to use it: `lxc storage create fast zfs size=100GiB`, then start the panel with `UIAVM_STORAGE_POOL=fast`.

### Optional: install Docker (only for Docker VPS)

Skip this if you only want LXC VPS. To let a node create **Docker VPS**, install Docker on it:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
docker version        # should print a Server version
```

On extra nodes you can let the install command do this for you, see [step 4](#4-add-more-nodes). UIAVM only manages containers it created (they carry the label `uiavm=1`), so your other Docker containers are left alone.

### LXC "Docker-ready" profile: load two kernel modules

The default **Docker-ready** container profile lets users run Docker inside their VPS. It needs these modules on the host, otherwise the container will not start:

```bash
sudo modprobe overlay br_netfilter
printf 'overlay\nbr_netfilter\n' | sudo tee /etc/modules-load.d/uiavm.conf
```

### Host already runs Docker?

Docker's firewall rules can block LXD containers from the internet. Allow the LXD bridge:

```bash
sudo iptables -I DOCKER-USER -i lxdbr0 -j ACCEPT
sudo iptables -I DOCKER-USER -o lxdbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

(Make these persistent with `iptables-persistent`, or your firewall tool.)

---

## 2. Install the UIAVM panel

You need **Python 3.9+** and **LXD** (step 1) on this server.

```bash
sudo apt install -y python3 python3-venv python3-pip git

git clone https://github.com/YOUR_USERNAME/uiavm.git /opt/uiavm-panel
cd /opt/uiavm-panel

python3 -m venv venv
. venv/bin/activate
pip install -r requirements.txt

sudo env "PATH=$PATH:/snap/bin" venv/bin/python uiavm.py
```

Open **`http://SERVER_IP:5000`** and sign in:

| | |
|---|---|
| Username | `admin` |
| Email | `admin@gmail.com` |
| Password | `admin@123` |

> **Change the password right after your first sign-in** (top-right menu, then *Profile and security*). Anyone who reads this README knows the default. To choose your own before the first run: `UIAVM_ADMIN_PASSWORD='something-strong' ...`.

Make sure port **5000** is reachable (`sudo ufw allow 5000/tcp`), or put the panel behind HTTPS, see [below](#https-with-nginx).

The first start creates `uiavm.db` (your data), `uiavm.log` and a `.uiavm_secret` file (session key). Keep all of them out of Git with a `.gitignore`:

```gitignore
uiavm.db*
uiavm.log
.uiavm_secret
backups/
venv/
__pycache__/
```

---

## 3. Create your first VPS

1. Sign in as `admin`.
2. Open **Create VPS** in the sidebar.
3. Choose how to **make the VPS**: **LXC** (full Linux system) or **Docker** (container from a Docker image). See [Docker or LXC?](#docker-or-lxc).
   Then choose an **owner**, a **node** (leave on *Automatic*), the operating system or image, and the resources (RAM, CPU, disk).
4. Press **Create VPS** and watch each step. The first creation of an image downloads it, which can take a minute.
5. Open the VPS: use the **Console** tab for a terminal, **Ports** to forward port 22 for SSH, **Snapshots** for restore points.

Other people can **register** from the login page (*Create account*). New accounts have no VPS until an admin creates one for them. Turn registration off in **Settings** if you do not want public sign-ups.

**Share a terminal with a link:** open the VPS, then **Manage → Remote access**. Press **Install and start** on **Tmate** (SSH command and web link) or **sshx** (one browser link). If a link stops working, press **Regenerate**. See [Remote access](#remote-access-tmate-and-sshx).

---

## Remote access: Tmate and sshx

Each VPS has two optional ways to open its terminal from anywhere without forwarding a port:

| | **Tmate** | **sshx** |
|---|---|---|
| You get | An SSH command (`ssh TOKEN@ssh.tmate.io`) and a web link | One link that opens a browser terminal ([sshx.io](https://sshx.io)) |
| Installed from | The VPS's package manager (apt, apk, dnf, yum, pacman) | `curl -sSf https://sshx.io/get \| sh` |
| Best for | Quick SSH access from your own terminal | Sharing a live terminal with other people |

How it works:
- Press **Install and start**. UIAVM installs the tool inside the VPS if it is missing (the VPS needs outbound internet), starts a session in the background and shows the link with copy and open buttons.
- A link works **while its session is running**. A session ends when the VPS restarts or its process stops. The card then shows **expired**: press **Regenerate** to get a fresh link. Regenerating also kills the old link and disconnects anyone using it.
- **Stop all sessions** ends both.
- Anyone holding a link gets a **root shell** in that VPS, so treat links like passwords. They are visible only to people who can open the VPS (owner, shared users, admins).
- The VPS must be **running**, and the image needs `sh` (all normal OS images have it).

---

## Docker or LXC?

| | **LXC** (system container) | **Docker** (application container) |
|---|---|---|
| What it is | A whole Linux system with its own init, package manager and SSH | One image running one main process |
| Best for | "A server": hosting, SSH, many services, running Docker inside | Apps, bots, web services, databases from an existing image |
| Images | Ubuntu, Debian, Alpine, Fedora, Rocky, Alma, CentOS, Arch, or any LXD image | Ubuntu, Debian, Alpine, Python, Node.js, Nginx, Redis, or **any Docker Hub image** you type in |
| Port forwards | TCP and UDP | TCP only |
| Disk limit | Yes (needs ZFS or btrfs) | Only if the node's Docker storage supports quotas, otherwise skipped with a notice |
| Snapshots | Full LXD snapshots, instant restore | Filesystem snapshots (`docker commit`). Restore recreates the container from the snapshot. Mounted volumes are not included |
| Resize RAM / CPU | Live | Live |
| Resize disk | Yes | No |
| Needs on the node | LXD | Docker |

**Notes for Docker VPS**
- Plain OS images (ubuntu, debian, alpine...) exit immediately, so UIAVM starts them with a placeholder process and you work in the **Console** tab. App images such as `nginx` keep their own command. You can change this with the *keep it running* switch when creating.
- Port forwards for Docker VPS are handled by the node agent (a small TCP proxy), so you can add and remove them while the VPS runs. They survive restarts of the agent and the panel.
- *Privileges: Privileged* runs the container with `--privileged`. Only for people you fully trust.

---

## 4. Add more nodes

Each extra server becomes a node in three steps: **register it in the panel, run one command on the server, done.**

**Before you start**

| Requirement | Detail |
|---|---|
| Node server | Linux with systemd, root access, internet access. LXD and/or Docker (the installer can add them) |
| Panel → node | TCP **7071** must be reachable *from the panel server* to the node |
| Node → panel | The node must be able to open `http://PANEL_IP:5000` |
| Panel address | If you browse the panel as `localhost`, set `UIAVM_PUBLIC_URL=http://PANEL_IP:5000` so generated commands work from other servers |

### Method A: in the web panel (one or many)

1. Go to **Nodes** and press **Add nodes**.
2. **One node:** fill in a name and location. **Several at once:** switch to *Several at once* and paste one node per line:

   ```text
   sg-01, Singapore, 203.0.113.20
   de-01, Frankfurt, 198.51.100.7
   us-01, Virginia, 192.0.2.44
   ```

   (`name, location, public IP`. Location and IP are optional.)
3. Check the **panel address** field is what the servers can reach, then copy the command for each node.
4. Tick **Install LXD** and/or **Install Docker** (only what is missing gets installed). Use Docker only for a Docker-only node, LXD only for LXC, both for either.
5. Run each command **as root on its server**:

   ```bash
   curl -fsSL http://PANEL_IP:5000/node/install/TOKEN | sudo bash
   # with Docker too:
   curl -fsSL "http://PANEL_IP:5000/node/install/TOKEN?docker=1" | sudo bash
   # Docker only (no LXD):
   curl -fsSL "http://PANEL_IP:5000/node/install/TOKEN?docker=1&lxd=0" | sudo bash
   ```

If you added public IPs, the panel also gives you **one script that installs every node over SSH** at once (needs SSH key access to each server).

### Method B: from the command line

On the panel server:

```bash
# one node
python3 uiavm.py node add --name sg-01 --location Singapore --panel http://PANEL_IP:5000

# many nodes from a file
cat > nodes.txt <<'EOF'
# name, location, public IP
sg-01, Singapore, 203.0.113.20
de-01, Frankfurt, 198.51.100.7
us-01, Virginia, 192.0.2.44
EOF

python3 uiavm.py node add --file nodes.txt --panel http://PANEL_IP:5000 --slots 50

# add --docker to also install Docker on each node, and --no-lxd for Docker-only nodes
python3 uiavm.py node add --file nodes.txt --panel http://PANEL_IP:5000 --docker
```

It prints an install command per node, and, for nodes that have an IP, a ready SSH script:

```bash
ssh -o StrictHostKeyChecking=accept-new root@203.0.113.20 'curl -fsSL http://PANEL_IP:5000/node/install/TOKEN | sudo bash' &
ssh -o StrictHostKeyChecking=accept-new root@198.51.100.7 'curl -fsSL http://PANEL_IP:5000/node/install/TOKEN | sudo bash' &
wait
```

Save that output as `install-nodes.sh` and run it: every server is set up in parallel.

Useful commands:

```bash
python3 uiavm.py node list                                          # all nodes and their state
python3 uiavm.py node token sg-01 --panel http://PANEL_IP:5000      # show a node's install command again
python3 uiavm.py node remove sg-01                                  # delete a node with no VPS on it
```

### What the install command does

On the node it will: install Python 3 and curl if missing, **install LXD and run `lxd init --auto` if missing** (unless you chose Docker only), **install Docker if you asked for it and it is missing**, download the agent from your panel, save its token to `/etc/uiavm/node.json` (readable by root only), create and start the `uiavm-node` systemd service, and open TCP 7071 in `ufw` or `firewalld` if one is active.

Within a few seconds the node appears **online** on the Nodes page. The agent registers its own address, so you never type a node URL.

> The installer uses `lxd init --auto` (the `dir` storage backend). For disk limits, set up ZFS or btrfs on the node first, as described in [step 1](#choose-a-storage-backend-important-for-disk-limits), **before** running the install command.

### Check a node

On the node:

```bash
systemctl status uiavm-node
journalctl -u uiavm-node -f        # live agent log
```

In the panel: **Nodes → Details** shows latency, kernel, LXD and Docker versions, storage pools and networks. Each node card lists which runtimes it has, and the Create VPS page only offers nodes that can run the runtime you picked.

**Updating older agents:** nodes installed with an earlier UIAVM only support LXC. Re-run their install command (add `?docker=1` for Docker) to upgrade the agent.

### Manage nodes

| Action | Where | What it does |
|---|---|---|
| **Drain** | Node card | Stops new VPS being placed there. Existing VPS keep running. Press again to resume. |
| **Edit** | Node card | Name, location, max VPS, tags, public IPs and domain aliases (shown to users in SSH commands and port-forward addresses). |
| **Install command** | Node card | Shows the install command again, plus the uninstall command. Re-running it **updates** the agent. |
| **Issue a new token** | Install command dialog | Revokes the old token at once. Reinstall the agent with the new command. |
| **Remove** | Node card | Disconnects the node. If it still hosts VPS you must confirm, and those VPS are destroyed. |

**How VPS are placed:** *Automatic* picks the online, non-draining node with room **that supports the chosen runtime** and has the lowest mix of VPS count, memory use and CPU use. Give nodes **tags** (for example `ssd`, `eu`) to target groups.

**Tighten the firewall** after installing: the installer opens 7071 to everyone. Restrict it to your panel:

```bash
sudo ufw delete allow 7071/tcp
sudo ufw allow from PANEL_IP to any port 7071 proto tcp
```

**Remove an agent** from a server:

```bash
systemctl disable --now uiavm-node && rm -rf /opt/uiavm /etc/uiavm /etc/systemd/system/uiavm-node.service
```

**Run the agent by hand** (no systemd, for testing):

```bash
python3 uiavm_node.py --panel http://PANEL_IP:5000 --token TOKEN --port 7071
```

**Use a different agent port:** `curl -fsSL .../node/install/TOKEN | sudo UIAVM_NODE_PORT=8443 bash`.

**Public ports:** a port forward listens on **that node's own public IP**, so users reach `NODE_IP:PORT`. Set each node's public IP in *Edit*.

---

## Run the panel as a service

Create `/etc/systemd/system/uiavm.service`:

```ini
[Unit]
Description=UIAVM panel
After=network-online.target snapd.service

[Service]
WorkingDirectory=/opt/uiavm-panel
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
Environment=UIAVM_PUBLIC_URL=http://PANEL_IP:5000
ExecStart=/opt/uiavm-panel/venv/bin/python /opt/uiavm-panel/uiavm.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now uiavm
sudo journalctl -u uiavm -f
```

**Update:** `cd /opt/uiavm-panel && git pull && venv/bin/pip install -r requirements.txt && sudo systemctl restart uiavm`
Your data lives in `uiavm.db`, which `git pull` never touches. Re-run a node's install command to update its agent.

---

## HTTPS with nginx

Run the panel on localhost (`UIAVM_HOST=127.0.0.1`), set `UIAVM_PUBLIC_URL=https://panel.example.com`, and proxy it. The `/socket.io/` block is **required** for the web terminal.

```nginx
server {
    listen 80;
    server_name panel.example.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /socket.io/ {
        proxy_pass http://127.0.0.1:5000/socket.io/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 3600s;
    }
}
```

Then `sudo certbot --nginx -d panel.example.com`. Nodes will use the HTTPS address in their install commands.

---

## Configuration

Set these as environment variables (for example in the service file).

| Variable | Default | Meaning |
|---|---|---|
| `UIAVM_HOST` / `UIAVM_PORT` | `0.0.0.0` / `5000` | Listen address |
| `UIAVM_PUBLIC_URL` | auto | Address nodes use to reach the panel |
| `UIAVM_STORAGE_POOL` | `default` | LXD storage pool for new VPS |
| `UIAVM_ADMIN_USER` | `admin` | First-run admin username |
| `UIAVM_ADMIN_EMAIL` | `admin@gmail.com` | First-run admin email |
| `UIAVM_ADMIN_PASSWORD` | `admin@123` | First-run admin password |
| `UIAVM_RESET_ADMIN` | `0` | `1` resets the main admin to the values above on start |
| `UIAVM_DB` | `./uiavm.db` | Database path |
| `UIAVM_POLL_INTERVAL` | `5` | Seconds between node polls |
| `UIAVM_BACKUP_INTERVAL` | `3600` | Seconds between database backups |
| `UIAVM_SECRET_KEY` | generated | Session signing key |
| `UIAVM_NODE_TLS_VERIFY` | `1` | `0` if agents use self-signed certificates |
| `UIAVM_DEBUG` | `0` | Flask debug mode (never in production) |

Everything else (branding, limits, auto-suspend thresholds, default container profile, registration, maintenance mode, backups) is in **Settings** in the panel and applies instantly.

**Container profiles** (chosen per VPS when creating):

| Profile | What it allows |
|---|---|
| `standard` | Isolated container, no nesting |
| `docker` (default) | Nesting and FUSE so Docker/Podman work, still unprivileged |
| `privileged` | Full access. Root in the VPS can be root on the host. Trusted users only. |

---

## REST API

Get your key under **Profile and security → API key**, then:

```bash
curl -H "X-API-Key: YOUR_KEY" http://PANEL_IP:5000/api/v1/vps

curl -X POST -H "X-API-Key: YOUR_KEY" -H "Content-Type: application/json" \
     -d '{"action":"restart"}' http://PANEL_IP:5000/api/v1/vps/1/power
```

| Endpoint | Method | Notes |
|---|---|---|
| `/api/v1/me` | GET | Who the key belongs to |
| `/api/v1/vps` | GET | Your VPS (all VPS for admins) |
| `/api/v1/vps/<id>` | GET | One VPS with live stats |
| `/api/v1/vps/<id>/power` | POST | `{"action": "start\|stop\|restart\|freeze\|unfreeze"}` |
| `/api/v1/nodes` | GET | Admin only |

---

## Upgrading from HVM

Put this folder next to your old `hvm.db` and start UIAVM. The database is **imported automatically** into `uiavm.db` (users, VPS, nodes, port forwards, settings) and the old file is left untouched. Your old admin password keeps working.

To set the admin back to the defaults once: `UIAVM_RESET_ADMIN=1 python3 uiavm.py`.
To reset any password: `python3 uiavm.py admin reset-password --user NAME --password NEWPASS`.

---

## What's new in 12.1

- **Remote access.** Tmate and sshx links per VPS, installed on demand and regenerable.
- **Real VPS specs.** A System details panel reads the real OS, kernel, architecture, processor, memory limit and disk usage from inside the VPS. Docker VPS now show their true disk usage instead of a dash.
- **One look everywhere.** The login page's aurora glow, drifting network, glass panels with a light that follows your cursor, animated panel edges, gradient headings, shimmering buttons and a rotating globe behind the dashboard hero now run through the whole panel.

## What's new in 12.0

- **Docker or LXC per VPS.** New runtime chooser on Create VPS. Docker VPS support live stats, console, power actions, snapshots, resizing and port forwards.
- **Smarter nodes.** The agent (v2) manages both runtimes, nodes report their LXD and Docker versions, the scheduler only picks nodes that support the chosen runtime, and the node installer can add Docker with `?docker=1`.
- **New login page.** Rotating globe of connected nodes, aurora background, live activity preview, a tilting glass card, Caps Lock warning and a password strength meter.
- **Better dashboard.** Animated cluster map, count-up numbers, time-aware greeting, always-visible cluster status in the top bar, entrance animations, runtime badges and a grid view for the VPS list.

---

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `lxc: command not found` | LXD is installed with snap. Add `/snap/bin` to `PATH` (the service file above does). |
| Node stuck on **pending** | The agent has not reached the panel. On the node: `journalctl -u uiavm-node -f`. Check the node can open `http://PANEL_IP:5000` and that you used the right panel address in the command. |
| Node shows **offline** | The panel cannot reach `NODE_IP:7071`. Check the firewall, and `systemctl status uiavm-node` on the node. |
| Install command points to `localhost` | You browsed the panel as `localhost`. Edit the address in the dialog, or set `UIAVM_PUBLIC_URL`. |
| LXC VPS creation fails at the disk step | Your storage pool cannot enforce disk limits (`dir` backend). Use ZFS or btrfs, see [step 1](#choose-a-storage-backend-important-for-disk-limits). |
| "Docker is not installed" / Docker option greyed out | The node has no Docker. Install it ([step 1](#optional-install-docker-only-for-docker-vps)) or re-run the node install command with `?docker=1`. |
| Docker VPS created without a disk limit | Normal on most Docker setups: quotas need the `overlay2` driver on XFS with `pquota`. A notice is shown while creating. |
| Docker VPS stops right after starting | The image has no long-running command. Enable *keep it running* when creating, or use an app image. |
| Tmate / sshx "could not be installed" or "could not reach" | The VPS has no outbound internet, or its image has no supported package manager. Test from the Console tab, for example `curl -I https://sshx.io`. |
| Docker port forward "cannot listen on port" | Something else on the node uses that port. UIAVM retries with another port automatically. |
| VPS will not start, mentions a kernel module | Run `sudo modprobe overlay br_netfilter` on the node, or use the `standard` profile. |
| VPS has no internet | Host Docker firewall rules, see [step 1](#host-already-runs-docker). Also check `lxc network list`. |
| Terminal stays blank | Behind a proxy? Add the `/socket.io/` websocket block shown above. The VPS must be running. |
| Forgot the admin password | `python3 uiavm.py admin reset-password --password NEWPASS` |
| "Too many attempts" on login | Five minutes cooldown after 8 wrong passwords from one address. Restart the panel to clear it. |

Logs: `uiavm.log` (panel), `journalctl -u uiavm-node` (agents), **Activity log** page (who did what).

---

## Security notes

- **Change the default admin password** immediately, and turn on 2FA for admin accounts.
- Node tokens are secrets. Anyone with a token can control that node's containers. Use *Issue a new token* if one leaks.
- Agents only run `lxc` or `docker` with an argument list, never a shell. Anyone holding a node token can still create privileged containers, which is root-equivalent on that node, so restrict port 7071 to the panel's IP and keep tokens secret.
- Serve the panel over HTTPS, and use `UIAVM_NODE_TLS_VERIFY` plus agent certificates (`--cert`, `--key`) if node traffic crosses untrusted networks.
- Avoid the `privileged` LXC profile and Docker's *Privileged* option for anyone you do not fully trust.
- Back up `uiavm.db`. Hourly copies are kept in `backups/`, and you can download them from **Settings**.

---

## Project layout

```
uiavm.py          the panel (Flask, Socket.IO, SQLite) and command-line tools
uiavm_node.py     the node agent (standard library only), also used by the panel's local node
uiavm-forwards.json  created at runtime: Docker port forwards of the panel's own node
templates/        pages
static/           CSS, JavaScript, bundled fonts and terminal (no CDN needed)
requirements.txt  panel dependencies
uiavm.service     example systemd unit
```

---

<sub>UIAVM, made by reon.</sub>
