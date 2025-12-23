# OpenAirInterface (OAI) O1 Adapter

This repository contains the [OAI O1 Adapter](https://gitlab.eurecom.fr/oai/o1-adapter) deployment guide for integrating OAI gNB with an O-RAN-SC OAM components via the O1 interface.

![OAI O1 Adapter Architecture](./images/oai-o1-adapter.jpg)

## Pre-requisites

Install Docker:

```bash
# 1. Update and install basic tools
sudo apt update
sudo apt install -y git net-tools putty unzip ca-certificates curl make

# 2. Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. Add the repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker packages
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose

# 5. Add current user to docker group (avoids using sudo for docker)
sudo usermod -a -G docker $(whoami)
```

> [!TIP]
> Log out and back in to apply group changes or run `su - $USER` to start a new session.

## 1. Repository Preparation

```bash
git clone https://gitlab.eurecom.fr/oai/o1-adapter.git
cd o1-adapter
```

## 2. Update Configuration

Update configuration before building as this is common for deployment methods — **docker**, **source**, or **Makefile**.

### For docker, source and makefile-based builds:

Modify `docker/config/config.json`:

1. Set host IP address (`host` under `network` section) and telnet server IP address (`host` under `telnet` section) to host IP.
2. Update the `ves.url` parameter to `http://<HOST_IP>:<VES_SVC_PORT>/eventListener/v7` to connect with the SMO VES Collector.

### For Makefile-based builds only:

1. Update the `integration/.env` file as the values got further propagated to `integration/config/config.json`.

   - Rename the incorrect variable from `OAI_OAI_TELNET_HOST` to `OAI_TELNET_HOST`
   - Add a new variable `VES_PORT` and set the value to <VES_SVC_PORT> from OSC OAM deployment.
     - `VES_PORT=<VES_SVC_PORT>`
   - Change `VES_COLLECTOR_URL` value from `https://${VES_FQDN}/eventListener/v7` to `http://${VES_IP}:${VES_PORT}/eventListener/v7`
   - Update `OAI_TELNET_PORT` value to `9091`

> [!NOTE]
> You can get the value of `<VES_SVC_PORT>` from the [ngkore/OSC-OAM](https://github.com/ngkore/OSC-OAM?tab=readme-ov-file#exposing-ves-collector-for-oai-o1-adapter).

## 3. Deployment Method 1: Docker-Based

### 3.1 Build the Adapter Container

Supported flags for `build-adapter.sh`:

| Flag         | Description                                                              |
| ------------ | ------------------------------------------------------------------------ |
| `--dev`      | Build development container (debug tools included; not production-ready) |
| `--adapter`  | Build production adapter container                                       |
| `--no-cache` | Disable docker build cache                                               |

Build the adapter:

```bash
./build-adapter.sh --adapter
```

> [!TIP]
> Rebuild after config changes using the same command.

### 3.2 Start Telnet Server (Separate Terminal)

The adapter uses a telnet-based interface to communicate with the RAN.
Install dependencies:

```bash
sudo apt update
sudo apt install python3-pip
pip3 install telnetlib3
```

Run the telnet test server:

```bash
cd o1-adapter/docker/scripts/
python3 servertest.py
```

### 3.3 Start the gNB Adapter

```bash
./start-adapter.sh --adapter
```

At this point, if the configuration is correct, the adapter will be listening for NETCONF clients and ready to communicate with the gNB’s telnet server.

## 4. Deployment Method 2: Source-Based Build

### 4.1 System Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y tzdata build-essential git cmake pkg-config unzip wget \
  libpcre2-dev zlib1g-dev libssl-dev autoconf libtool python3-pip

sudo apt install -y --no-install-recommends psmisc unzip wget openssl \
  openssh-client vsftpd openssh-server

pip3 install telnetlib3
```

### 4.2 NETCONF User Preparation

Create a dedicated system user:

```bash
sudo adduser --system netconf
echo "netconf:netconf!" | sudo chpasswd
```

SSH directory:

```bash
sudo mkdir -p /home/netconf/.ssh
sudo chmod 700 /home/netconf/.ssh
```

### 4.3 Configure OpenSSH for netconf User

Edit `/etc/ssh/sshd_config`:

```bash
sudo vim /etc/ssh/sshd_config
```

Append:

```
Match User netconf
       ChrootDirectory /
       X11Forwarding no
       AllowTcpForwarding no
       ForceCommand internal-sftp -d /ftp
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

### 4.4 Configure vsftpd

Prepare directories:

```bash
sudo mkdir -p /ftp
sudo chown -R netconf:nogroup /ftp
sudo mkdir -p /var/run/vsftpd/empty
sudo mkdir -p /run/sshd
```

Edit `/etc/vsftpd.conf`:

```bash
sudo vim /etc/vsftpd.conf
```

Ensure the following settings are present:

```
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
chroot_local_user=YES
allow_writeable_chroot=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
ssl_enable=NO
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
```

Add `netconf` to `/etc/vsftpd.userlist`:

```bash
sudo vim /etc/vsftpd.userlist
```

Insert:

```
netconf
```

### 4.5 Prepare O1 Adapter Source Tree

The adapter expects configuration inside `src/`:

```bash
cp -r docker/config/ src/
```

### 4.6 Install NETCONF Dependencies

```bash
cd o1-adapter/docker/scripts/
sudo ./netconf_dep_install.sh
sudo ldconfig
```

Run hostkey merge utilities:

```bash
sudo /usr/local/share/netopeer2/merge_hostkey.sh
sudo /usr/local/share/netopeer2/merge_config.sh
```

Fetch and install YANG models:

```bash
mkdir -p yang
./get-yangs.sh
sudo ./install-yangs.sh
```

### 4.7 Build the gNB Adapter (Source)

```bash
cd ../../src/
./build.sh
```

Rebuild after config changes:

```bash
rm gnb-adapter
./build.sh
```

### 4.8 Start Supporting Processes

#### Terminal 1: NETCONF Server

```bash
sudo netopeer2-server -d -v3 -t 60
```

#### Terminal 2: Telnet Test Server

```bash
cd o1-adapter/docker/scripts/
python3 servertest.py
```

#### Terminal 3: gNB Adapter

```bash
export TERM=xterm-256color
cd o1-adapter/src/
sudo ./gnb-adapter
```

## 5. Deployment Method 3: Makefile-Based Deployment

Through `Makefile`, we can deploy the **four different combinations** depending on what you want to integrate with:

1. **Adapter alone** (no gNB, no SMO)
2. **Adapter + telnet test server** (fake gNB)
3. **Adapter + SMO** (real O-RAN SMO integration, but still needs a telnet server or real gNB)
4. **Adapter + SMO + telnet test server** (full O-RAN environment with fake gNB)

### 5.1 Launch Adapter Using Makefile

#### Standalone Adapter

```bash
make run-o1-oai-adapter
```

#### Adapter + Telnet Test Server

```bash
make run-o1-adapter-telnet
```

> [!IMPORTANT]
> The below methods for SMO are outdated and still expected to fail. See [ngkore/OSC-OAM](https://github.com/ngkore/OSC-OAM) for more details on SMO deployment and integration.

#### Adapter + SMO

```bash
make run-o1-oai-adapter-smo
```

#### Adapter + SMO + Telnet Server

```bash
make run-o1-oai-adapter-smo-telnet
```

### 5.2 Teardown

```bash
make teardown
```

## 6. Testing the gNB Adapter with OAI gNB

Start NETCONF server (only for source-based builds). Skip this step if using docker or Makefile based adapter.

```bash
sudo netopeer2-server -d -v3 -t 60
```

### 6.3 Start gNB Adapter

Docker or Makefile-based deployment:

```bash
./start-adapter.sh --adapter
```

Source build deployment:

```bash
cd src/
sudo ./gnb-adapter
```

### 6.4 Start OAI gNB (RFsim + O1 Telnet)

Add following flags to gNB command to enable O1 telnet server on port `9091`:

```bash
  --telnetsrv \
  --telnetsrv.listenport 9091 \
  --telnetsrv.shrmod o1
```
