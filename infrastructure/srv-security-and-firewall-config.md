# CrowdSec - Control Plane Server Security

[CrowdSec](https://github.com/crowdsecurity/crowdsec) is an open-source security solution that protects servers from brute-force attacks, port scans, and other malicious activities. It is particularly effective at protecting SSH access.

The configurations below should be installed and configured only on the `control plane` or on a `designated ingress node` that is publicly exposed to the internet. In other words, this should be a server with both a public and a private IP address.

## Table of Contents

- [Install CrowdSec](#install-crowdsec)
- [Firewall Rules Configuration for K8s/RKE2](#firewall-rules-configuration-for-k8srke2)
  - [Firewall Rules for Control Plane Nodes](#firewall-rules-for-control-plane-nodes)
  - [Firewall Rules for Worker Nodes](#firewall-rules-for-worker-nodes)
- [CrowdSec Commands Playbook](#crowdsec-commands-playbook)
  - [CrowdSec Docs](#crowdsec-docs)

## Install CrowdSec

1. Install CrowdSec:
   Login as the service account user if you have terminated the ssh session. Add the CrowdSec repository and install the package.

    ```
    curl -s https://install.crowdsec.net | sudo sh
    sudo apt update && sudo apt install crowdsec
    ```

2. Install firewall bouncer with iptables:
   The firewall bouncer uses iptables to automatically block malicious IPs detected by CrowdSec. This will also ensure iptables is installed.

    ```
    sudo apt install crowdsec-firewall-bouncer-iptables -y
    ```

3. After installation, verify that CrowdSec has created its iptables chains:

    ```
    sudo iptables -L
    ```

    Note: You should see the output showing the `CROWDSEC_CHAIN` has been created and is active.

## Firewall Rules Configuration for K8s/RKE2

The following rules are to be configured to allow certain connections for administration and to comply with `RKE2`'s [Networking](https://docs.rke2.io/install/requirements?cni-rules=Canal#networking) and [Inbound Network Rules](https://docs.rke2.io/install/requirements?cni-rules=Canal#inbound-network-rules).

> **Important — iptables backend:** On modern Ubuntu/Debian, the `iptables` command is a symlink to either `iptables-nft` (default) or `iptables-legacy`. RKE2's `kube-proxy` and CNI manage their own large rule sets and will silently break service routing if the host uses one backend while the cluster components use the other. Confirm the active backend before applying rules:
>
> ```bash
> update-alternatives --display iptables
> iptables -V   # should report nf_tables or legacy
> ```
>
> If a mismatch exists between the host and `kube-proxy`'s expectation, switch with `update-alternatives --set iptables /usr/sbin/iptables-legacy` (or the nft path) on every node, then restart `rke2-server`/`rke2-agent`.

### Firewall Rules for Control Plane Nodes

1. Allow established and related connections:

   ```
   sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
   ```

2. Allow loopback traffic:

   ```
   sudo iptables -A INPUT -i lo -j ACCEPT
   ```

3. Allow ICMP echo requests from the private network:

   ```
   sudo iptables -A INPUT -i <private-interface> -p icmp --icmp-type 8 -s <private-network-subnet> -d <control-plane-private-ip> -j ACCEPT
   ```

4. Allow ICMP echo replies from the private network:

   ```
   sudo iptables -A INPUT -i <private-interface> -p icmp --icmp-type 0 -s <private-network-subnet> -d <control-plane-private-ip> -j ACCEPT
   ```

5. Allow SSH access on a non-standard port (2222):

   ```
   sudo iptables -A INPUT -p tcp -d <control-plane-public-ip> --dport 2222 -j ACCEPT
   ```

6. Allow HTTP traffic on port 80 for application access:

   ```
   sudo iptables -A INPUT -p tcp -d <control-plane-public-ip> --dport 80 -j ACCEPT
   ```

7. Allow HTTPS traffic on port 443 for secure access to exposed services:

   ```
   sudo iptables -A INPUT -p tcp -d <control-plane-public-ip> --dport 443 -j ACCEPT
   ```

8. Allow Kubernetes API Server access (TCP 6443):

   ```
   sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 6443 -j ACCEPT
   ```

9. Allow RKE2 supervisor API access (TCP 9345):

   ```
   sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 9345 -j ACCEPT
   ```

10. Allow etcd client traffic (TCP 2379):

      ```
      sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 2379 -j ACCEPT
      ```

11. Allow etcd peer traffic (TCP 2380):

      ```
      sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 2380 -j ACCEPT
      ```

12. Allow etcd metrics/health traffic (TCP 2381):
   
      ```
      sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 2381 -j ACCEPT
      ```

13. Allow kubelet API access (TCP 10250):

      ```
      sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 10250 -j ACCEPT
      ```

14. Allow kube-proxy health/metrics endpoint (TCP 10256):

      ```
      sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 10256 -j ACCEPT
      ```

      > Required by `kube-proxy` for its `/healthz` endpoint. MetalLB and external load balancers depend on this for backend health checks.

15. Allow NodePort service range (TCP 30000–32767) — *skip if NodePort is not used*:

      ```
      sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --match multiport --dports 30000:32767 -j ACCEPT
      ```

16. Allow VXLAN overlay networking (UDP 8472):

      ```
      sudo iptables -A INPUT -p udp -d <control-plane-private-ip> --dport 8472 -j ACCEPT
      ```

17. Allow RKE2 control-plane metrics and health endpoint (TCP 9099):

      ```
      sudo iptables -A INPUT -p tcp -d <control-plane-private-ip> --dport 9099 -j ACCEPT
      ```

18. Allow traffic from the Kubernetes Pod CIDR (cluster CIDR):

      ```
      sudo iptables -A INPUT -s <cluster CIDR> -j ACCEPT
      ```

19. Allow traffic from the Kubernetes Service CIDR (service CIDR):

      ```
      sudo iptables -A INPUT -s <service CIDR> -j ACCEPT
      ```

20. Allow pod and node egress from the private network to the public network:

      ```
      sudo iptables -A FORWARD -i <private-network-interface> -o <public-network-interface> -j ACCEPT
      ```

21. Allow established and related return traffic from the public network:

      ```
      sudo iptables -A FORWARD -i <public-network-interface> -o <private-network-interface> -m state --state ESTABLISHED,RELATED -j ACCEPT
      ```

22. Set default FORWARD policy to ACCEPT:

      ```
      sudo iptables -P FORWARD ACCEPT
      ```

      > **Why ACCEPT, not DROP:** Almost all Kubernetes networking traffic — pod-to-pod, pod-to-service, VXLAN-decapsulated inner packets — traverses the `FORWARD` chain, **not** `INPUT`. RKE2's `kube-proxy` and Canal CNI add their own filtering rules within `FORWARD` via `KUBE-FORWARD` and `cali-FORWARD` chains; setting the base policy to `ACCEPT` ensures those chains govern the decision instead of falling through to a blanket DROP that silently breaks pod and service traffic. If a future requirement demands `FORWARD = DROP`, comprehensive ACCEPT rules for the cluster CIDR, service CIDR, and VXLAN return traffic must be added first.

23. Set default INPUT policy to DROP:

      ```
      sudo iptables -P INPUT DROP
      ```

24. Set default OUTPUT policy to ACCEPT:

      ```
      sudo iptables -P OUTPUT ACCEPT
      ```

25. Confirm the firewall rules:

      ```
      sudo iptables -L -n -v
      ```

26. Install iptables persistence utilities:

      ```
      sudo apt install iptables-persistent -y
      ```

27. Save the current iptables ruleset. Click `yes` for both `IPv4` and `IPv6` when prompted:

      ```
      sudo netfilter-persistent save
      ```

28. To update the ruleset in the future:

      ```
      # Saves ruleset after updating it
      sudo netfilter-persistent save

      # Reloads saved ruleset from disk
      sudo netfilter-persistent reload
      ```

### Firewall Rules for Worker Nodes

Worker nodes are not externally exposed in this cluster, but they still require a host firewall. "Not publicly exposed" only addresses external internet attackers — worker firewalls protect against:

- **Lateral movement** after a compromise of the externally exposed control plane (the most likely first foothold).
- **Compromised pods** that escape their container or use `hostNetwork: true` and land directly on the worker's host network.
- **Untrusted neighbors** on the private network — many VPS providers share VLANs across customers, and bare-metal private subnets often share switches with non-cluster systems (jumphosts, monitoring, build agents).
- **Service-binding bugs** where an add-on or debug tool accidentally listens on `0.0.0.0` instead of `127.0.0.1`.
- **CIS / compliance** — CIS Benchmark, PCI-DSS, SOC 2, and ISO 27001 all require host-level firewalls regardless of network position. "Behind another firewall" is not an accepted compensating control.

The principle is **zero trust within the private network**, not "the private network is trusted."

Worker nodes do **not** open the control-plane-only ports (`6443`, `9345`, `2379`, `2380`, `2381`) or any externally facing port (no public IP, no SSH on a public address, no `80`/`443`, NodePort not used in this cluster). Run the following on **each worker node** as root:

1. Allow established and related connections:

   ```
   sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
   ```

2. Allow loopback traffic:

   ```
   sudo iptables -A INPUT -i lo -j ACCEPT
   ```

3. Allow ICMP echo requests from the private network:

   ```
   sudo iptables -A INPUT -i <private-interface> -p icmp --icmp-type 8 -s <private-network-subnet> -d <worker-private-ip> -j ACCEPT
   ```

4. Allow ICMP echo replies from the private network:

   ```
   sudo iptables -A INPUT -i <private-interface> -p icmp --icmp-type 0 -s <private-network-subnet> -d <worker-private-ip> -j ACCEPT
   ```

5. Allow SSH access on a non-standard port (2222) from the private network only:

   ```
   sudo iptables -A INPUT -p tcp -s <private-network-subnet> -d <worker-private-ip> --dport 2222 -j ACCEPT
   ```

6. Allow kubelet API access (TCP 10250) from control plane and other cluster nodes only:

   ```
   sudo iptables -A INPUT -p tcp -s <control-plane-private-ip> -d <worker-private-ip> --dport 10250 -j ACCEPT
   ```

   > Tightening kubelet access by source IP is a meaningful hardening win — only the control plane (and metrics scrapers, if running on the host network) needs to reach `:10250`.

7. Allow kube-proxy health/metrics endpoint (TCP 10256):

   ```
   sudo iptables -A INPUT -p tcp -d <worker-private-ip> --dport 10256 -j ACCEPT
   ```

8. Allow VXLAN overlay networking (UDP 8472) from cluster nodes:

   ```
   sudo iptables -A INPUT -p udp -d <worker-private-ip> --dport 8472 -j ACCEPT
   ```

9. Allow Canal CNI health endpoint (TCP 9099):

   ```
   sudo iptables -A INPUT -p tcp -d <worker-private-ip> --dport 9099 -j ACCEPT
   ```

10. Allow traffic from the Kubernetes Pod CIDR (cluster CIDR):

      ```
      sudo iptables -A INPUT -s <cluster CIDR> -j ACCEPT
      ```

11. Allow traffic from the Kubernetes Service CIDR (service CIDR):

      ```
      sudo iptables -A INPUT -s <service CIDR> -j ACCEPT
      ```

12. Set default FORWARD policy to ACCEPT (same reasoning as control plane — pod and service traffic traverses the FORWARD chain):

      ```
      sudo iptables -P FORWARD ACCEPT
      ```

13. Set default INPUT policy to DROP:

      ```
      sudo iptables -P INPUT DROP
      ```

14. Set default OUTPUT policy to ACCEPT:

      ```
      sudo iptables -P OUTPUT ACCEPT
      ```

15. Confirm the firewall rules:

      ```
      sudo iptables -L -n -v
      ```

16. Install iptables persistence utilities and save the ruleset:

      ```
      sudo apt install iptables-persistent -y
      sudo netfilter-persistent save
      ```

> **CrowdSec on workers is optional.** Workers have no public exposure, so the brute-force/scanning attack surface CrowdSec defends against is minimal. If you want consistent decision enforcement across the cluster (e.g., a CrowdSec decision on the control plane should also block the IP from reaching workers via VPN/jumphost paths), install only the `crowdsec-firewall-bouncer-iptables` package on workers and point it at the control plane's CrowdSec API as the LAPI source. The full CrowdSec agent + log parsers don't need to run on workers.

## CrowdSec Commands Playbook

1. Checks the current status of the CrowdSec service:

   ```
   sudo systemctl status crowdsec
   ```

2. Lists all installed CrowdSec collections, which are sets of parsers, scenarios, and post-overflows used for detecting specific attack patterns:

   ```
   sudo cscli collections list
   ```

3. Displays all configured CrowdSec bouncers, which enforce decisions on traffic (e.g., block or allow IPs). Useful to verify active enforcement agents:

   ```
   sudo cscli bouncers list
   ```

4. Shows the rules, packet, and byte counters in the specific CrowdSec iptables chain, providing insight into blocked or allowed traffic:

   ```
   sudo iptables -L CROWDSEC_CHAIN -n -v
   ```

5. Lists all iptables rules with counters and numeric IP/port display. Helps verify overall firewall and CrowdSec integration:

   ```
   sudo iptables -L -n -v
   ```

6. Displays all current active decisions made by CrowdSec (e.g., blocked IPs), showing their scope and expiration:

   ```
   sudo cscli decisions list
   ```

7. Lists all alerts generated by CrowdSec scenarios, useful for auditing and incident analysis:

   ```
   sudo cscli alerts list
   ```

8. Shows statistics and metrics about CrowdSec performance, such as the number of decisions, alerts, and bouncer activity:

   ```
   sudo cscli metrics
   ```

10. Continuously monitors the CrowdSec log file in real-time, useful for troubleshooting or observing detection and enforcement activity live:

      ```
      sudo tail -f /var/log/crowdsec.log
      ```
   
### CrowdSec Docs

- [Intro](doc.crowdsec.net/docs/next/appsec/intro/)
- [Cyber Threat Intelligience (CTI)](https://app.crowdsec.net/cti)
- [Log Processor](https://doc.crowdsec.net/docs/next/log_processor/intro/)
- [Configuration](https://doc.crowdsec.net/docs/next/configuration/crowdsec_configuration)
- [Block Lists](https://app.crowdsec.net/blocklists)
- [White Listing IPs](https://doc.crowdsec.net/u/getting_started/post_installation/whitelists)
- [Hub](https://app.crowdsec.net/hub)
- [Rules](https://doc.crowdsec.net/docs/next/appsec/rules_syntax)
- [Troubleshooting](https://doc.crowdsec.net/docs/next/appsec/troubleshooting)

   

