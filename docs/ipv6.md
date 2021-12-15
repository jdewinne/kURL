# IPv6

kURL can be installed on IPv6 enabled hosts by passing the `ipv6` flag to the installer or by setting the `kurl.ipv6` field to `true` in the yaml spec.

```
sudo bash install.sh ipv6
```

This example shows a valid spec for ipv6.

```
apiVersion: cluster.kurl.sh/v1beta1
kind: Installer
metadata:
  name: ipv6
spec:
  kurl:
    ipv6: true
  kubernetes:
    version: 1.19.15
  containerd:
    version: 1.4.6
  antrea:
    isEncryptionDisabled: true
    version: 1.2.1
  rook:
    version: 1.5.12
  kotsadm:
    version: 1.58.1
  ekco:
    version: 0.12.0
    enableInternalLoadBalancer: true
  registry:
    version: 2.7.1
```

There is no auto-detection of ipv6 or fall-back to ipv4 when ipv6 is not enabled on the host.


## Current Limitations

* Dual-stack is not supported. Resources will have only an ipv6 address when ipv6 is enabled. The host can be dual-stack, but control plane servers, pods, and cluster services will use IPv6. Node port services must be accessed on the hosts' IPv6 address.
* Ubuntu 18 is the only supported OS. (CentOS 8 and Ubuntu 20 fail with Rook 1.5.12 in ipv6 mode, and CentOS 7 NodePorts fail, but might work if you upgrade the kernel).
* Antrea is the only supported CNI.
* Antrea with encryption enabled is not supported.
* Rook is the only supported CSI.
* Snapshots and restore have not been tested.
* External load balancer hasn't been tested.
* HTTP proxy hasn't been tested.
* Host preflight checks fail even though install succeeds.


## Host Requirements

* IPv6 forwarding must be enabled and bridge-call-nf6tables must be enabled. The installer does this automatically and configures this to persist after reboots.

* Using antrea, TCP 8091 and UDP 6081 have to be open between nodes instead of the ports used by weave (6784 and 6783).

## Troubleshooting


Problem: joining 2nd node to cluster fails
Symptom: nodes in cluster can't ping6 each other.
Symptom: `ip -6 route` shows no default route
Solution: `sudo ip -6 route add default dev ens5`

Problem: cannot `curl https://replicated.app`
Solution: deploy a NAT64 server
Solution: use airgap
Solution: wait for AAAA records to be added to replicated.app

Problem: networking check fails in curl installer
Symptom: antrea-agent logs show:
```
E1210 19:44:12.494994       1 route_linux.go:119] Failed to initialize iptables: error checking if chain ANTREA-PREROUTING exists in table raw: running [/usr/sbin/ip6tables -t raw -S ANTREA-PREROUTING 1 
--wait]: exit status 3: modprobe: FATAL: Module ip6_tables not found in directory /lib/modules/4.18.0-193.19.1.el8_2.x86_64
ip6tables v1.8.4 (legacy): can't initialize ip6tables table `raw': Table does not exist (do you need to insmod?)
Perhaps ip6tables or your kernel needs to be upgraded.
```
Symptom: `sudo lsmod | grep ip6` is empty
Solution: `sudo modprobe ip6_tables`

## Testing

### Dual-Stack

#### AWS

AWS has VPCs with dual-stack enabled in the default subnet.

### IPv6 Only Clusters

#### AWS

1. [Create an IPv6-only subnet within a dual-stack VPC.](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-ipv6-only-subnets-and-ec2-instances/). Use us-east-1, us-west-1, or us-west-2 if you need a NAT64 server.
1. Be sure to add `::0` to your route table pointing to your internet gateway.
1. [Add a NAT64 server](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)

#### Connecting

If you are working on a network with only IPv4 you'll need to connect to your cluster through a dual stack jump box.

You'll also need to proxy node port services through your jump box. HAProxy works well for that:
```
sudo apt install -y haproxy
```
frontend kots
    bind 0.0.0.0:8800
    mode tcp
    option tcplog
    default_backend kots

backend kots
    mode tcp
    server kots [2600:1f14:75b:2d00:5e3d:25bf:6af1:643a]:8800
```

#### Hetzner

[Hetzner has IPv6-only servers now.](https://www.hetzner.com/news/12-21-ipv6-only/).
