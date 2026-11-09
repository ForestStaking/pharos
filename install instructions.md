# Pharos Pacific Mainnet Node Installation Guide

> **Required before startup.** The Docker host must have an open-file ceiling of at least **10,000,000** (`fs.nr_open`). The node container cannot start with the configured open-file limit if the host ceiling is lower. [Set the open-file ceiling](#set-the-open-file-ceiling).

## Prerequisites

### Hardware

| Component | Specification |
| --- | --- |
| CPU | 32 cores at 2.8 GHz or faster, AMD EPYC Milan or Intel Xeon Platinum |
| Memory | 256 GB |
| Storage | 5 TB SSD with at least 350 MiB/s throughput and 30,000 IOPS |
| Network bandwidth | 0.5 Gbps |
| Open file limit | 10,000,000 |

[Pharos Docker deployment requirements](https://docs.pharos.xyz/node-and-validator-guide/validator-node-deployment/using-docker-deployment#prerequisites)

### Software

- x86-64 Linux
- Docker Engine 20.10 or later
- Docker Compose v2 or later
- `curl`, `wget`, `aria2`, `pv`, `pigz`, `jq`, `tree`, and `screen`

#### Update the system

```bash
sudo apt update &&
sudo apt upgrade -y
```

Reboot the machine if required, for example if the upgrade prints **`Pending kernel upgrade!`**.

#### Install on Ubuntu

Install the command-line tools and add Docker's repository signing key.

```bash
sudo apt-get update &&
sudo apt-get install -y ca-certificates curl wget aria2 pv pigz jq tree screen &&
sudo install -d -m 0755 /etc/apt/keyrings &&
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc &&
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker's APT repository for your Ubuntu release.

```bash
. /etc/os-release
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: ${VERSION_CODENAME}
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Install Docker Engine and the Compose plugin, then check both versions.

```bash
sudo apt-get update &&
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin &&
sudo docker version &&
sudo docker compose version
```

#### Install Foundry (optional)

Install Foundry if you want to use `cast` to query the node's RPC. Run the installer as your regular user. [Foundry installation](https://getfoundry.sh/)

```bash
curl -fsSL https://getfoundry.sh/install | bash &&
"$HOME/.foundry/bin/foundryup" &&
"$HOME/.foundry/bin/cast" --version
```

If you use Bash on Ubuntu, run these commands to make `cast` and `forge` available in this terminal and in future Bash sessions.

```bash
echo 'export PATH="$HOME/.foundry/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.foundry/bin:$PATH"
```

### Set the open-file ceiling

Check the open-file ceiling on the Docker host before starting the container. [Linux kernel documentation](https://docs.kernel.org/admin-guide/sysctl/fs.html#nr-open)

```bash
sysctl fs.nr_open
```

If the value is below `10000000`, save the new ceiling in `/etc/sysctl.d/` and apply it immediately. The saved setting is loaded at boot.

```bash
printf '%s\n' 'fs.nr_open = 10000000' |
sudo tee /etc/sysctl.d/99-pharos-nofile.conf >/dev/null &&
sudo sysctl -p /etc/sysctl.d/99-pharos-nofile.conf
```

#### Set the open-file limit for host logins

If you run `./ops health-check` on the Ubuntu host, set the login limits after raising `fs.nr_open`. This gives new `root` and `pharos` login sessions the required soft and hard limits. [Ubuntu limits.conf manual](https://manpages.ubuntu.com/manpages/bionic/man5/limits.conf.5.html)

```bash
sudo tee /etc/security/limits.d/99-pharos-nofile.conf >/dev/null <<'EOF'
root   soft nofile 10000000
root   hard nofile 10000000
pharos soft nofile 10000000
pharos hard nofile 10000000
EOF
```

Disconnect and start a new SSH login as the account you use to run the health check. Both commands should print `10000000`.

```bash
ulimit -Sn
ulimit -Hn
```

If a fresh SSH login still shows `1024`, check that SSH uses PAM and that its PAM session configuration loads `pam_limits.so`.

### Enable CPU performance mode

On the Ubuntu host, install `cpupower` and enable a systemd service that sets the performance governor for all CPUs now and at each boot. [Ubuntu cpupower documentation](https://ubuntu.com/server/docs/explanation/performance/perf-tune-cpupower/)

```bash
sudo apt-get install -y linux-tools-common "linux-tools-$(uname -r)" &&
sudo tee /etc/systemd/system/pharos-cpu-performance.service >/dev/null <<'EOF'
[Unit]
Description=Set the CPU frequency governor to performance

[Service]
Type=oneshot
ExecStart=/usr/bin/cpupower -c all frequency-set -g performance
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload &&
sudo systemctl enable --now pharos-cpu-performance.service
```

After a kernel upgrade, reboot into the new kernel before running the commands below. The service stays enabled across upgrades, but `cpupower` needs tools that match the running kernel. [Ubuntu kernel tools documentation](https://ubuntu.com/real-time/docs/how-to/reduce-latency/)

```bash
sudo apt-get install -y "linux-tools-$(uname -r)" &&
sudo systemctl restart pharos-cpu-performance.service
```

### Create Pharos user

Create a user named **pharos** with a home directory and Bash as its login shell.

```bash
sudo useradd -m -s /bin/bash pharos
```

### Grant sudo and Docker access

After installing Docker, run these commands from your existing administrator account. On Ubuntu, the `sudo` group grants administrative access. The `docker` group grants root-level access through Docker. Set the `pharos` password when prompted so the account can authenticate with `sudo`. [Docker permissions on Linux](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)

```bash
sudo usermod -aG sudo,docker pharos && sudo passwd pharos
```

Start a fresh login shell to load the new group memberships.

```bash
sudo -iu pharos
```

In that shell, confirm that `id` lists both groups, then check sudo, Docker Engine, and Compose.

```bash
id
sudo -v && docker info && docker compose version
```

## **1. Create working directory**

From the `pharos` shell, create a fresh working directory owned by `pharos`.

```bash
export WORKSPACE=pharos
sudo install -d -m 0755 -o pharos "/data/$WORKSPACE" && cd "/data/$WORKSPACE"
```

## **2. Download configuration files**

Download the genesis configuration and `bin/VERSION` for the network you plan to run. Run only one of the blocks below. Choose Atlantic Testnet or Mainnet to match the network you plan to run.

**For Atlantic Testnet:**

```bash
mkdir -p bin &&
wget -O genesis.conf https://raw.githubusercontent.com/PharosNetwork/resources/main/atlantic.genesis &&
wget -O bin/VERSION https://raw.githubusercontent.com/PharosNetwork/resources/main/atlantic.version
```

**For Mainnet:**

```bash
mkdir -p bin &&
wget -O genesis.conf https://raw.githubusercontent.com/PharosNetwork/resources/main/mainnet.genesis &&
wget -O bin/VERSION https://raw.githubusercontent.com/PharosNetwork/resources/main/mainnet.version
```

Download `pharos.conf` for the full node.

```bash
wget -O pharos.conf https://raw.githubusercontent.com/PharosNetwork/resources/main/conf/full.conf
```

The remaining steps use the Pacific Mainnet Docker image and chain ID.

### Download and extract the Mainnet snapshot

This guide uses the Mainnet snapshot available at the time of writing. If you use a newer snapshot, update the download URL, archive filename, and SHA-256 checksum together. [Pharos Mainnet snapshots](https://docs.pharos.xyz/node-and-validator-guide/pharos-network-snapshots#mainnet-node-snapshot)

> **Tip:** Pharos snapshots can take a while to download. An extracted Atlantic Testnet snapshot can take up a few terabytes of disk space, and decompression can take several hours. If you are connected over SSH, run `screen -S pharos-snapshot` before downloading and use the same session for extraction. Press **Ctrl+A**, then **D**, to detach. After reconnecting, run `screen -r pharos-snapshot` to return to the session.

```bash
aria2c -c -x 12 -s 12 https://snapshot.dplabs-internal.com/mainnet/mainnet-snapshot-2026-11-01-03.tar.gz
```

Verify the downloaded archive against the published SHA-256 checksum. Continue to extraction only if the check reports `OK`. [Pharos Mainnet snapshot checksums](https://docs.pharos.xyz/node-and-validator-guide/pharos-network-snapshots#mainnet-node-snapshot)

```bash
(
  set -o pipefail
  pv mainnet-snapshot-2026-11-01-03.tar.gz |
    sha256sum -c <(printf '%s  -\n' 'a8dc0a0888730285a7afcd2865b18e28dd73b098194ea682237034f50542b5a1')
)
```

After the download finishes, extract the archive in the same directory. The command creates `public/` and shows progress with `pv`.

```bash
set -o pipefail
pv mainnet-snapshot-2026-11-01-03.tar.gz | pigz -dc -p 16 | tar -xf -
```

> Snapshot extraction can take several hours.

Leave `public/` in the working directory until the node has started and its block height has advanced. Then follow [Restore the Mainnet snapshot](#restore-the-mainnet-snapshot). [Pharos Network Snapshots](https://docs.pharos.xyz/node-and-validator-guide/pharos-network-snapshots#mainnet-node-snapshot)

## **3. Create `docker-compose.yml`**

Create `docker-compose.yml` in `/data/pharos`. This example publishes the HTTP RPC, WebSocket, P2P TCP, and node RPC ports on the Docker host. Review the port mappings and firewall rules for your deployment. [Pharos port mappings](https://github.com/PharosNetwork/resources/blob/main/docker-compose.yml) · [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)

The Compose example hardcodes the host directory as `/data/pharos`. If you changed `WORKSPACE` in step 1, update the host path in `volumes` to match your working directory. Keep the container path on the right as `/data`.

```bash
cat > docker-compose.yml <<EOF
services:
  pharos:
    image: public.ecr.aws/k2g7b7g1/pharos:pharos_community_v0.16.0_3cf35988_0914
    container_name: pharos-node
    environment:
      - CONSENSUS_KEY_PWD=YOUR_PASSWORD_HERE  # Change this to your password
      - PHAROS_CONF=/data/pharos.conf
      - GENESIS_CONF=/data/genesis.conf
      - KEYS_DIR=/data/keys
    volumes:
      # Mount data directory
      - /data/pharos:/data
    ports:
      - "18100:18100"  # HTTP RPC
      - "18200:18200"  # WebSocket
      - "19000:19000"  # P2P TCP
      - "20000:20000"  # RPC
    restart: unless-stopped
    ulimits:
      nofile:
        soft: 10000000
        hard: 10000000
    healthcheck:
      test: ["CMD", "curl", "-sf", "-X", "POST", "-H", "Content-Type: application/json", "-d", "{\"jsonrpc\":\"2.0\",\"method\":\"eth_blockNumber\",\"params\":[],\"id\":1}", "http://localhost:18100"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
EOF
```

### Choose which ports to expose

- **19000 (P2P):** Publish this TCP port so other Pharos nodes can connect to yours. It is the only port that needs to be public for peer traffic.
- **18100 (HTTP API):** Bind this port to `127.0.0.1` so you can check the block height from the Docker host.
- **18200 (WebSocket) and 20000 (additional RPC):** Leave these ports unpublished unless an application needs them. Bind them to localhost for local clients or restrict remote access to trusted clients.

[Pharos port mappings](https://docs.pharos.xyz/node-and-validator-guide/validator-node-deployment/using-docker-deployment#3-create-docker-composeyml) · [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)

For public P2P and local HTTP RPC, replace the example’s `ports` block with:

```yaml
    ports:
      - "127.0.0.1:18100:18100"  # Local HTTP RPC
      - "19000:19000"            # P2P TCP
```

## **4. Set the node password**

Open `docker-compose.yml` in your preferred text editor, such as Nano or Vim.

```bash
nano docker-compose.yml
```

Find this line under `environment`.

```yaml
- CONSENSUS_KEY_PWD=YOUR_PASSWORD_HERE
```

Replace the placeholder with your actual password.

```yaml
- CONSENSUS_KEY_PWD=your_secure_password
```

Docker Compose treats `$` as interpolation syntax in environment values.[^compose-password]

[^compose-password]: When editing `docker-compose.yml`, write `$$` for each literal `$` in the password. For example, enter `abc$$123` to pass `abc$123` to the container. [Docker Compose interpolation](https://docs.docker.com/reference/compose-file/interpolation/)

Save the file. Keep the same password across restarts and upgrades so the node can open its keys.

## **5. Verify required files**

Before starting the container, list the working directory. This command skips the snapshot archive and its extracted files.

```bash
tree -I 'public|*.tar.gz|*.aria2' .
```

Check that these files are present.

```text
.
├── bin/
│   └── VERSION
├── docker-compose.yml
├── genesis.conf
└── pharos.conf
```

- `genesis.conf` contains the network genesis configuration.
- `pharos.conf` contains the full node configuration. Step 2 downloads `full.conf` and saves it locally as `pharos.conf`.
- `docker-compose.yml` contains the container settings and node password.
- `bin/VERSION` contains the network version information.

If a file is missing, return to steps 2–4 to download or create it.

## **6. Start the node**

Start the Pharos service in the background.

```bash
docker compose up -d
```

### Verify the domain public key after first boot

From `/data/pharos`, check `keys/domain.pub` after the node generates its keys. On some first boots, a bug produces a key that is only four characters long. Display the file to check for that symptom.

```bash
sudo cat keys/domain.pub
```

To print the key only if it is exactly 134 hexadecimal characters, run:

```bash
sudo grep -xE '[[:xdigit:]]{134}' keys/domain.pub
```

> **Warning:** Run this recovery only on a newly initialized, unregistered node whose `keys/domain.pub` contains exactly four characters. The procedure permanently deletes `keys/` and `data/`, removing the current node identity and local chain data. Skip to the next section on any existing node or registered validator.

Install `xxd` inside the running container.

```bash
docker compose exec -u root pharos sh -c \
  'apt-get update && apt-get install -y xxd && command -v xxd'
```

After installing `xxd`, stop the node and delete the incomplete keys and generated data.

```bash
cd /data/pharos &&
docker compose stop pharos &&
sudo rm -rf -- ./keys ./data
```

Once the directories are removed, bring the node back up with the same Compose configuration so Docker reuses the container with `xxd` installed. [Pharos container entrypoint](https://github.com/PharosNetwork/resources/blob/main/docker-entrypoint.sh)

```bash
docker compose up -d
```

Once the node regenerates its keys, rerun the public key check before continuing.

## **7. Check node status**

View startup logs.

```bash
docker compose logs --tail=100 -f pharos
```

In another terminal, run the following command and look for `pharos-node` under `NAMES`.

```bash
docker ps
```

Once the container passes its healthcheck, the status includes `(healthy)`. The example below shows selected columns.

```text
CONTAINER ID   STATUS               NAMES
16c3f029215b   Up 2 days (healthy)  pharos-node
```

### Restore the Mainnet snapshot

**Required: Start the node with `docker compose up -d` before applying the snapshot.** The first startup creates the node keys and initializes the `/data/pharos/data/` directory. Complete the first-boot key check and wait for the local block height to increase. Then stop the node before moving the extracted `public/` directory into `/data/pharos/data/`.

Run this command on the Docker host a few times and wait for the local block height to increase.

```bash
height=$(
  curl -fsS --max-time 10 http://127.0.0.1:18100 \
    -H 'Content-Type: application/json' \
    --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' |
    jq -er '.result'
) &&
printf '%d\n' "$height"
```

Once the height has advanced, stop the node and replace its existing `data/public/` directory with the extracted snapshot.

Run the commands from `/data/pharos`. The snapshot’s destination is `/data/pharos/data/public`.

```bash
cd /data/pharos &&
test -d public && test -d data &&
docker compose stop pharos &&
sudo rm -rf -- data/public &&
sudo mv -T -- public data/public &&
docker compose up -d
```

Check the logs and run the block-height command again to confirm the node resumes syncing.

```bash
docker compose logs --tail=100 -f pharos
```

### Remove the downloaded snapshot archive

Once the node is running and its block height is increasing after the restore, remove the downloaded archive to free disk space. Keep `/data/pharos/data/public/`, which the node uses.

```bash
rm -- /data/pharos/mainnet-snapshot-2026-11-01-03.tar.gz
```

### Run the host health check

After the node is running and the snapshot has been restored, run the health check from the working directory. The node creates `ops` in that directory during startup.

```bash
cd /data/pharos && ./ops health-check
```

## Key Management

### Back Up Node Keys

The container creates these files in `/data/pharos/keys/` on first boot.

- `domain.key` and `domain.pub`
- `stabilizing.key` and `stabilizing.pub`

## Becoming a Validator

Once your full node is synced, review the validator requirements and confirm the Pacific Mainnet onboarding process with the Pharos team. [Pharos validator requirements](https://docs.pharos.xyz/node-and-validator-guide/validator-requirements)

## Troubleshooting

### Test larger TCP buffers (optional)

On the host used for this guide, increasing TCP buffers to about 16 MB helped reduce dropped connections. If you see similar drops, record the current settings and compare the drop rate and socket memory use before and after tuning under a similar load. Keep the change only if it helps. [Linux TCP sysctls](https://docs.kernel.org/networking/ip-sysctl.html)

For a host-side trial, inspect the current TCP receive and send settings.

```bash
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

Each setting lists the minimum, default, and maximum in bytes. The example below raises each maximum to 16 MiB (16,777,216 bytes) if it is lower, preserving the other values. The change lasts until reboot. To undo it sooner, restore the values you recorded with `sudo sysctl -w`.

```bash
for setting in net.ipv4.tcp_rmem net.ipv4.tcp_wmem; do
  read -r min default max < <(sysctl -n "$setting") || break
  if [ "$max" -lt 16777216 ]; then
    sudo sysctl -w "$setting=$min $default 16777216" || break
  fi
done
```

| Symptom | Check |
| --- | --- |
| Container fails to start with a `nofile` error | Check the Docker host's `fs.nr_open` ceiling and the Compose `nofile` values. |
| Node starts but block height stays flat | Inspect startup and peer logs, check P2P connectivity, and confirm the genesis file, version file, and image are for Pacific Mainnet. |
| RPC reports a different chain ID | Check what is listening on local port `18100` and confirm the mounted data directory belongs to Pacific Mainnet. |
| Key generation fails | Check the mounted directory's permissions and confirm `CONSENSUS_KEY_PWD` is set in `docker-compose.yml`. |

## Managing the node

Run these commands from `/data/pharos`.

### Stop the node

```bash
docker compose stop pharos
```

### Restart the node

```bash
docker compose restart pharos
```

### Stop and remove the container

```bash
docker compose down
```

The bind mount keeps node files and chain data on the host in `/data/pharos/`.

### Follow the logs

```bash
docker compose logs -f pharos
```

### Open a shell in the container

```bash
docker compose exec pharos bash
```

## Directory structure

After the first boot and snapshot restore, the Mainnet working directory contains the following files. Log filenames may vary by release.

```text
/data/pharos/
├── bin/                   # Binaries copied from the image at startup
│   ├── pharos_light
│   ├── pharos_cli
│   └── VERSION            # Downloaded in step 2
├── ops                    # Copied from the image at startup
├── keys/                  # Generated on first boot
│   ├── domain.key
│   ├── domain.pub
│   ├── stabilizing.key
│   └── stabilizing.pub
├── data/                  # Initialized on first boot
│   ├── meta_store/
│   └── public/            # Restored from the snapshot
├── log/
│   ├── aldaba.log
│   ├── aldaba_cli.log
│   └── pamir.log
├── genesis.conf           # Downloaded in step 2
├── pharos.conf            # Downloaded in step 2
└── docker-compose.yml     # Created in step 3 and edited in step 4
```

## Security recommendations

- Set a strong password for `CONSENSUS_KEY_PWD` and restrict access to `docker-compose.yml`.
- Limit access to the private keys in `/data/pharos/keys/`.
- Expose only the ports your deployment needs. Docker forwards traffic to published container ports before ordinary UFW deny rules can filter it, so those ports can remain reachable even when UFW shows them as denied. [Docker and UFW](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw)
- Back up `keys/`, `pharos.conf`, and `docker-compose.yml` together in encrypted storage.
- Check Pharos release notes before updating the image and network version file.
- Review the node logs regularly for errors and warnings.
