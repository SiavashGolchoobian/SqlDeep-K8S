#####  [^1]1-Set SELinux to `permissive` mode:
```bash
# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
##### 2-Uninstall any existed docker
```bash
systemctl disable --now docker docker.socket
dnf remove -y docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras containerd.io
rm -rf /var/lib/docker
rm -rf /etc/docker
dnf remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
```
##### 3-After uninstalling docker query your remaining docker NIC's and remove them
```bash
#Query NIC's
sudo ip link

#Remove docker NIC
sudo ip link delete docker0
#Remove any bridge NIC's of docker
sudo ip link delete br-9cbd9d549307
sudo ip link delete ******
sudo ip link delete ******
```
##### 4-Disable Linux SWAP feature
```bash
swapon --show
#make a backup from fstab file
cp -a /etc/fstab /etc/fstab.bak
#turn off swap
sudo swapoff -a
sudo swapoff /dev/dm-1
#And o make this change persistent across reboots run bellow command
sudoedit /etc/fstab
#and In `/etc/fstab`, comment out the line whose filesystem type is `swap` by adding `#` at its start. For example:

#Reload the configuration and verify:
sudo systemctl daemon-reload
swapon --show
free -h
```
##### 5-Enable IPv4 packet forwarding
```bash
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify that net.ipv4.ip_forward is set to 1 with:
sysctl net.ipv4.ip_forward
```
##### [^2]6-Installing containerd from the official binaries
Download the `containerd-<VERSION>-<OS>-<ARCH>.tar.gz` archive from [https://github.com/containerd/containerd/releases](https://github.com/containerd/containerd/releases) , also download "containerd.service" file from https://raw.githubusercontent.com/containerd/containerd/main/containerd.service , "runc" from https://github.com/opencontainers/runc/releases, CNI Plugins from https://github.com/containernetworking/plugins/releases and install them as following:

```shell
cd /opt
mkdir sources
cd /opt/sources

# download and install containerd
wget https://github.com/containerd/containerd/releases/download/v2.4.1/containerd-2.4.1-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-2.4.1-linux-amd64.tar.gz

# download and enable containerd service file:
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
mv containerd.service /usr/lib/systemd/system/
systemctl daemon-reload
systemctl enable --now containerd

# install runc
wget https://github.com/opencontainers/runc/releases/download/v1.5.2/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

# download and install CNI plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.9.1/cni-plugins-linux-amd64-v1.9.1.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.9.1.tgz
```
##### Step 7: Add the Kubernetes yum repository
```bash
# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.37/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.37/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
##### Step 8: Install kubelet, kubeadm and kubectl
```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
[^1]: [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
[^2]: https://github.com/containerd/containerd/blob/main/docs/getting-started.md
