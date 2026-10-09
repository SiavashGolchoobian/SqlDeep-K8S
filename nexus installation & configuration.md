## Part 1 — Install Nexus

### 1. Prepare the host

Allocate at least:

- 2 vCPU
    
- 8 GB RAM
    
- 100 GB of fast local disk for the Nexus data directory
    

Following command creates a dedicated Linux user account named `nexus` to run the Nexus service safely.

```bash
sudo dnf install -y curl tar firewalld

sudo useradd \
  --system \
  --create-home \
  --home-dir /opt/sonatype-work \
  --shell /bin/bash \
  nexus

sudo passwd -l nexus
```

- `--system` creates a system/service account, not a normal human login account.
- `--create-home` creates its home directory.
- `--home-dir /opt/sonatype-work` sets that home directory to where Nexus stores data, logs, configuration, and cached packages.
- `--shell /bin/bash` gives the account a valid shell, which Nexus expects for its service user.
- `nexus` is the username.

The next command in the guide, `sudo passwd -l nexus`, locks its password so nobody can log in interactively as that account.
### 2. Download and verify Nexus Repository

This example uses Nexus Repository `3.96.4-01`, the current release at the time this runbook was written. Check the [official downloads page](https://help.sonatype.com/en/download.html) before installing.

```bash
export NEXUS_VERSION='3.96.4-01'
export NEXUS_FILE="nexus-${NEXUS_VERSION}-linux-x86_64.tar.gz"
export NEXUS_URL="https://download.sonatype.com/nexus/3/${NEXUS_FILE}"

cd /opt

sudo curl -fLO "${NEXUS_URL}"
sudo curl -fsSLO "${NEXUS_URL}.sha256"

sha256sum "${NEXUS_FILE}"
cat "${NEXUS_FILE}.sha256"
```

Confirm that the SHA-256 hash displayed by both commands is identical.

### 3. Extract and configure Nexus
In this step i installs the downloaded Nexus files under `/opt` and ensures the `nexus` service account owns and runs them.

```bash
cd /opt

sudo tar xzf "${NEXUS_FILE}"

sudo ln -sfn "nexus-${NEXUS_VERSION}" /opt/nexus

sudo chown -R nexus:nexus \
  "/opt/nexus-${NEXUS_VERSION}" \
  /opt/sonatype-work

sudo tee /opt/nexus/bin/nexus.rc > /dev/null <<'EOF'
run_as_user="nexus"
EOF

sudo chown nexus:nexus /opt/nexus/bin/nexus.rc
```

- `sudo tar xzf "${NEXUS_FILE}"`  
    Extracts the downloaded Nexus `.tar.gz` archive, creating a versioned directory such as `/opt/nexus-3.96.4-01` and the data directory `/opt/sonatype-work`.
    
- `sudo ln -sfn "nexus-${NEXUS_VERSION}" /opt/nexus`  
    Creates `/opt/nexus` as a symbolic link to the current Nexus version directory. This makes upgrades easier: you later change the link instead of changing service paths.
    
- `sudo chown -R nexus:nexus ...`  
    Gives the `nexus` user ownership of the Nexus application and its data directory, so Nexus does not run as `root`.
    
- `sudo tee /opt/nexus/bin/nexus.rc ...`  
    Creates the Nexus startup configuration file with:
    
    ```
    run_as_user="nexus"
    ```
    
    This tells the Nexus startup script to run the service as the `nexus` user.
    
- `sudo chown nexus:nexus /opt/nexus/bin/nexus.rc`  
    Ensures that configuration file is also owned by the `nexus` account.

Nexus includes its required Java runtime; installing a separate Java package is not necessary for current Nexus releases. [Sonatype Java guidance](https://help.sonatype.com/en/upgrade-nexus-repository-java-version.html)

### 4. Create the system service
This creates and activates a `systemd` service definition for Nexus, so it starts automatically after reboot and can be managed with `systemctl`.

```bash
sudo tee /etc/systemd/system/nexus.service > /dev/null <<'EOF'
[Unit]
Description=Sonatype Nexus Repository
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
User=nexus
Group=nexus
LimitNOFILE=65536
WorkingDirectory=/opt/nexus
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
Restart=on-failure
RestartSec=15
TimeoutStartSec=600

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now nexus
```

Check status:

```bash
sudo systemctl status nexus
```

The first startup can take several minutes.

#### Description:
`sudo tee ... > /dev/null <<'EOF'` means:
- `sudo tee` writes the content as root.
- `> /dev/null` prevents the same content from being printed to the screen.
- `<<'EOF'` sends all text until `EOF` into the file literally—without expanding variables such as $HOME.

The service configuration means:
- `[Unit]`  
    Basic service metadata.
    
    - `Description=...` gives the service a readable name.
    - `After=` and `Wants=` ensure system networking is available before Nexus starts.
- `[Service]`  
    Defines how Nexus runs.
    - `Type=forking` means the startup command launches Nexus in the background.
    - `User=nexus` and `Group=nexus` run Nexus without root privileges.
    - `LimitNOFILE=65536` raises the maximum number of open files; Nexus can keep many package files and network connections open.
    - `WorkingDirectory=/opt/nexus` sets its working directory.
    - `ExecStart=... start` starts Nexus.
    - `ExecStop=... stop` stops it cleanly.
    - `Restart=on-failure` restarts Nexus if it crashes.
    - `RestartSec=15` waits 15 seconds before restarting.
    - `TimeoutStartSec=600` permits up to 10 minutes for the first startup.
- `[Install]`
    - `WantedBy=multi-user.target` makes it part of normal server startup.

Then:

```
sudo systemctl daemon-reload
```

tells systemd to reread service files.

```
sudo systemctl enable --now nexus
```

does two things:

1. Enables Nexus to start automatically after boot.
2. Starts Nexus immediately.

Make sure there are no leading spaces before `[Unit]`, `[Service]`, `[Install]`, or the final `EOF`.

### 5. Allow access only from the lab network

```bash
sudo systemctl enable --now firewalld

sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="10.10.10.0/24" port port="8081" protocol="tcp" accept'

sudo firewall-cmd --reload
```

Replace `10.10.10.0/24` with the actual lab subnet.
Also add another firewall rule for your browser enabled client too.
### 6. Perform initial Nexus setup

Open:

```
http://node-repo.lab:8081/
```

Retrieve the initial administrator password from Node-Repo:

```bash
sudo cat /opt/sonatype-work/nexus3/admin.password
```

Then:

1. Sign in as `admin`.
    
2. Enter the initial password.
    
3. Set a strong new administrator password.
    
4. Complete the onboarding steps.
    
5. Open **Settings → Security → Anonymous Access**.
    
6. Enable anonymous access for the lab nodes.
    

Anonymous access is acceptable only because the firewall limits access to the lab subnet.

---

## Part 2 — Create Yum Proxy Repositories in Nexus

In Nexus, go to:

```
Settings → Repositories → Create repository → yum (proxy)
```

Create the following five repositories.

| Repository name    | Remote storage URL                                             |
| ------------------ | -------------------------------------------------------------- |
| `alma10-baseos`    | `https://repo.almalinux.org/almalinux/10/BaseOS/x86_64/os/`    |
| `alma10-appstream` | `https://repo.almalinux.org/almalinux/10/AppStream/x86_64/os/` |
| `alma10-crb`       | `https://repo.almalinux.org/almalinux/10/CRB/x86_64/os/`       |
| `alma10-extras`    | `https://repo.almalinux.org/almalinux/10/extras/x86_64/os/`    |
| `k8s-v1-37`        | `https://pkgs.k8s.io/core:/stable:/v1.37/rpm/`                 |
|                    |                                                                |

For every repository, use these settings:

|Setting|Value|
|---|---|
|Online|Enabled|
|Blob store|`default`|
|Maximum component age|`60` minutes|
|Maximum metadata age|`60` minutes|
|Not found cache enabled|Enabled|
|Not found cache TTL|`5` minutes|
|Auto blocking|Enabled|
|HTTP authentication|Not configured|

Nexus caches a missing package on first request and serves later requests from its blob store. After the cache age expires, Nexus checks the upstream again; immutable RPM files are not downloaded again unless the upstream content changes. [Nexus proxy caching behavior](https://help.sonatype.com/en/repository-types.html)

Do not create a Yum group for these repositories initially. Keep the AlmaLinux repositories separate; this makes troubleshooting and repository metadata behavior predictable.

---

## Part 3 — Configure Each of the other Kubernetes Nodes

Repeat these steps on all three control-plane nodes and all three worker nodes.

### 1. Ensure Node-Repo resolves

Use internal DNS, or add this entry:

```bash
echo '10.10.10.7 node-repo.lab node-repo' | sudo tee -a /etc/hosts
```

### 2. Create the AlmaLinux Nexus repository file

```bash
sudo tee /etc/yum.repos.d/nexus-almalinux.repo > /dev/null <<'EOF'
[alma10-baseos-nexus]
name=AlmaLinux 10 - BaseOS via Nexus
baseurl=http://node-repo.lab:8081/repository/alma10-baseos/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-10

[alma10-appstream-nexus]
name=AlmaLinux 10 - AppStream via Nexus
baseurl=http://node-repo.lab:8081/repository/alma10-appstream/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-10

[alma10-crb-nexus]
name=AlmaLinux 10 - CRB via Nexus
baseurl=http://node-repo.lab:8081/repository/alma10-crb/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-10

[alma10-extras-nexus]
name=AlmaLinux 10 - Extras via Nexus
baseurl=http://node-repo.lab:8081/repository/alma10-extras/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-AlmaLinux-10
EOF
```

`gpgcheck=1` validates RPM signatures and must remain enabled.

`repo_gpgcheck=0` is intentional. Nexus may rewrite Yum metadata, so its documentation recommends retaining RPM signature verification while disabling repository-metadata verification unless you configure Nexus to sign metadata with your own internal GPG key. [Nexus Yum GPG documentation](https://help.sonatype.com/en/gpg-signatures-for-yum-proxy-group.html)

### 3. Create the Kubernetes Nexus repository file

Replace `v1.37` if you deliberately target another Kubernetes minor release.

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo > /dev/null <<'EOF'
[kubernetes]
name=Kubernetes v1.37 via Nexus
baseurl=http://node-repo.lab:8081/repository/k8s-v1-37/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=http://node-repo.lab:8081/repository/k8s-v1-37/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

Kubernetes publishes packages in separate repositories per minor version. [Official Kubernetes repository documentation](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/)

### 4. Disable direct AlmaLinux repositories

First inspect the repository IDs:

```bash
sudo dnf repolist --all
```

Then disable the direct upstream AlmaLinux repositories:

```bash
sudo dnf install -y dnf-plugins-core

sudo dnf config-manager --set-disabled \
  baseos \
  appstream \
  crb \
  extras
```

If repository IDs differ on a node, use the output of `dnf repolist --all` and disable the equivalent upstream repositories manually.

### 5. Refresh and validate

```bash
sudo dnf clean all
sudo dnf makecache --refresh
sudo dnf repolist --enabled
```

The enabled repositories should include only:

```
alma10-appstream-nexus
alma10-baseos-nexus
alma10-crb-nexus
alma10-extras-nexus
kubernetes
```

When ready to install Kubernetes packages (==🟢You can skip this command and following k8s installation according to kubeadm guide==):

```bash
sudo dnf install -y \
  --disableexcludes=kubernetes \
  kubelet kubeadm kubectl
```

---

## Part 4 — Optional: Cache Kubernetes Container Images

RPM caching does not cache Kubernetes images such as `kube-apiserver`, `etcd`, and `pause`.

### 1. Create a Docker proxy repository

In Nexus:

```
Settings → Repositories → Create repository → docker (proxy)
```

Use:

| Setting               | Value                     |
| --------------------- | ------------------------- |
| Name                  | `k8s-images`              |
| Remote storage        | `https://registry.k8s.io` |
| Docker index          | Use proxy registry        |
| HTTP connector port   | `5000`                    |
| Docker V1 API         | Disabled                  |
| Maximum component age | `60` minutes              |
| Maximum metadata age  | `60` minutes              |
| Auto blocking         | Enabled                   |

Open port `5000` only for the lab subnet:

```bash
sudo firewall-cmd --permanent \
  --add-rich-rule='rule family="ipv4" source address="10.10.10.0/24" port port="5000" protocol="tcp" accept'

sudo firewall-cmd --reload
```

### 2. Configure containerd on all six nodes for HTTP

Create the containerd registry configuration path:

```bash
sudo mkdir -p /etc/containerd/certs.d/node-repo.lab:5000
```

Ensure `/etc/containerd/config.toml` contains:

```
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
```

Then create this file:

```bash
sudo tee /etc/containerd/certs.d/node-repo.lab:5000/hosts.toml > /dev/null <<'EOF'
server = "http://node-repo.lab:5000"

[host."http://node-repo.lab:5000"]
  capabilities = ["pull", "resolve"]
EOF

sudo systemctl restart containerd
```

When initializing the first control-plane node, use:

```bash
sudo kubeadm init \
  --image-repository node-repo.lab:5000 \
  ...
```

For the remaining control-plane and worker nodes, the kubeadm join process pulls Kubernetes images through Nexus automatically.

Plain HTTP for the image proxy is suitable only for this isolated lab. Container runtimes must explicitly be configured to trust HTTP registries; this is why the `hosts.toml` configuration is required. Nexus documents that container clients normally expect HTTPS. [Nexus Docker routing and TLS guidance](https://help.sonatype.com/en/docker-registry.html)

## Operational Notes

- Node-Repo is a single point of failure for future package and image downloads. Existing Kubernetes workloads continue running if it becomes unavailable.
    
- Back up `/opt/sonatype-work/nexus3` regularly.
    
- Monitor available disk space. Nexus can become read-only when disk space is critically low.
    
- Disable or proxy any extra repositories, such as EPEL, Docker CE, CRI-O, or Cilium repositories, if you want to prevent every direct Internet package download.
    
- Do not expose ports `8081` or `5000` outside the lab subnet.