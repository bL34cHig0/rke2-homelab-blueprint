# Prerequisites for all Node Servers - Control Plane(s) & Worker(s):
After setting up Ubuntu 24.04 servers for control plane(s) and worker nodes on Hetzner. Switch to root user or run the commands below with `sudo` on all instances to 
install the prerequisites for RKE2.

Note: Exclude `sudo` from the commands if running as the `root` user on all instances.

## Install essential utility packages for secure key management, curl, wget, and repository management capabilities:
```
sudo apt update && sudo apt upgrade -y 
sudo apt install -y curl wget gnupg2 software-properties-common
```

## Change time zone & enable automatic time synchronization:
```
timedatectl
timedatectl list-timezones
timedatectl list-timezones | grep "Cairo"
sudo timedatectl set-timezone Africa/Cairo
sudo timedatectl set-ntp true
```

## Disable swap to prevent performance issue - read more on this [here](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/#risks-and-caveats):
```
sudo swapoff -a 
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Allow iptables / nftables to inspect and filter traffic passing through Linux bridges:
```
sudo modprobe br_netfilter
```
## Load OverlayFS kernel module for union filesystem functionality:
```
sudo modprobe overlay
```

## Create and apply critical kernel networking parameters required for Kubernetes and container runtimes to function correctly:
```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf 
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1 
EOF
```

## Apply all kernel parameter configurations system-wide:
```
sudo sysctl --system
```





