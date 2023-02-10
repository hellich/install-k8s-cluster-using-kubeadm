# install-k8s-cluster-using-kubeadm

## setup prerequisites on all nodes

```
#on all nodes

vagrant@kubemaster:~$ lsmod | grep br_netfilter
vagrant@kubemaster:~$ sudo modprobe br_netfilter
vagrant@kubemaster:~$ lsmod | grep br_netfilter
br_netfilter           24576  0
bridge                155648  1 br_netfilter
vagrant@kubemaster:~$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> EOF
sudo sysctl --systemnet.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vagrant@kubemaster:~$ sudo sysctl --system
* Applying /etc/sysctl.d/10-console-messages.conf ...
kernel.printk = 4 4 1 7
* Applying /etc/sysctl.d/10-ipv6-privacy.conf ...
net.ipv6.conf.all.use_tempaddr = 2
net.ipv6.conf.default.use_tempaddr = 2
* Applying /etc/sysctl.d/10-kernel-hardening.conf ...
kernel.kptr_restrict = 1
* Applying /etc/sysctl.d/10-link-restrictions.conf ...
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
* Applying /etc/sysctl.d/10-lxd-inotify.conf ...
fs.inotify.max_user_instances = 1024
* Applying /etc/sysctl.d/10-magic-sysrq.conf ...
kernel.sysrq = 176
* Applying /etc/sysctl.d/10-network-security.conf ...
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_syncookies = 1
* Applying /etc/sysctl.d/10-ptrace.conf ...
kernel.yama.ptrace_scope = 1
* Applying /etc/sysctl.d/10-zeropage.conf ...
vm.mmap_min_addr = 65536
* Applying /usr/lib/sysctl.d/50-default.conf ...
net.ipv4.conf.all.promote_secondaries = 1
net.core.default_qdisc = fq_codel
* Applying /etc/sysctl.d/99-cloudimg-ipv6.conf ...
net.ipv6.conf.all.use_tempaddr = 0
net.ipv6.conf.default.use_tempaddr = 0
* Applying /etc/sysctl.d/99-sysctl.conf ...
* Applying /etc/sysctl.d/k8s.conf ...
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
* Applying /etc/sysctl.conf ...
vagrant@kubemaster:~$
```

## install container runtime (docker)

### Update the apt package index and install packages to allow apt to use a repository over HTTPS:

```
sudo apt-get update
sudo apt-get install \
ca-certificates \
curl \
gnupg \
lsb-release
```

### Add Docker’s official GPG key:

```
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Use the following command to set up the repository:

```
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install docker engine

```
sudo chmod a+r /etc/apt/keyrings/docker.gpg
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### setup daemon.json

```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

### start service

```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo systemctl daemon-reload
sudo systemctl restart docker

root@kubemaster:~# systemctl status docker.service
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: e
   Active: active (running) since Fri 2023-02-10 15:43:34 UTC; 50s ago
     Docs: https://docs.docker.com
 Main PID: 6026 (dockerd)
    Tasks: 8
   CGroup: /system.slice/docker.service
           └─6026 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/contain

Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.444315350Z"
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.528449935Z"
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.648380096Z"
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.794043712Z"
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.842617562Z"
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.842661464Z"
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.842688878Z"
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.868644686Z"
Feb 10 15:43:34 kubemaster systemd[1]: Started Docker Application Container Engi
Feb 10 15:43:34 kubemaster dockerd[6026]: time="2023-02-10T15:43:34.884372365Z"
lines 1-19/19 (END)

```