# Highly Available Load Balancer for OpenShift

[![License: Apache v2](https://img.shields.io/badge/license-Apache%20v2-brightgreen.svg)](https://www.apache.org/licenses/LICENSE-2.0)

Ansible playbook for one-step installation of a highly available (HA)
load balancer (LB) setup for OpenShift. This setup provides fault
tolerancy, connection failover, and load balancing across Openshift
master and router nodes. Supports both ingress traffic (to OpenShift
pods) as well as egress traffic (pods initiating connections to external
networks). Only the load balancers are on the public network thus no
OpenShift nodes are exposed to a potentially hostile external network.

## Technical Description

The OpenShift Ansible installer provides only very limited load balancer
setup which lacks high-availability support. Hardware load balancers are
a great alternative but they are not always available or feasible.
Luckily, all modern Linux distributions come with all the needed
components to build highly available load balancers. Notably, RHEL 7
contains all the needed components on its base channel (so no need for
additional subscriptions).

Below we describe highly available load balancer setup with standard
Linux distribution components using two Linux servers (can be virtual
machines, VMs). Everything described below will be automatically
configured by the provided Ansible playbook.

The load balancers are connected to an external network, for example the
Internet, with one NIC (network interface card) and to an internal
network with another NIC.

keepalived is used to provide a VIP (virtual IP) on both sides. For
ingress traffic towards OpenShift, keepalived does load balancing itself
(by using the kernel IPVS modules - for a higher level description see
https://en.wikipedia.org/wiki/Linux_Virtual_Server). All keepalived
related management traffic is using the internal network only.

For OpenShift internal communication to the master API on the intranet
side HAProxy is used for load balancing (keepalived provides only the
internal VIP but not load balancing as RHEL does not support connections
from keepalived real servers to their own VIP).

For egress traffic initiated from OpenShift nodes iptables/netfilter is
used to provide SNAT using the external VIP and conntrackd provides
connection synchronization to ensure transparent connection failovers.

Linux iptables filters all the incoming traffic.

keepalived logs to syslog, conntrackd to its own log file. haproxy logs
most important messages to its own log and a separate log containing
connection level information can be easily enabled.

Optionally, a separate external VIP for each API and apps can be used to
allow traffic filtering to API VIP.

## Prerequisites

* Two Fedora / RHEL servers (can be VMs) to act as load balancers
  * Two NICs configured on both
  * Both connected to the internal and external networks
  * Three IPs from both the internal and external network allocated
    * One IP per each load balancer NIC and one VIP on both sides
* Ansible installed on a bastion host
* Passwordless SSH configured for Ansible
* Repositories (yum/dnf) configured on the servers
* OpenShift installation after this HA LB setup in the internal network
  * OpenShift nodes use the internal VIP as their default gateway
    * In more advanced setups policy based routing is also an alternative
* Basic understanding of the components in play

## Installation

```
vi ansible.hosts
ansible-playbook -i ./ansible.hosts lb-ha-setup.yml
```

## Monitoring and Testing

```
tail -f /var/log/messages /var/log/conntrackd.log /var/log/haproxy.log
firefox https://openshift.ext.example.com:8443/
watch -n 1 ipvsadm -lnc
conntrack -L -p TCP --dport 22
```

## TODO

* Consider HAProxy for ingress traffic as well
  * To pave way for using this setup on masters or routers
    * Would remove the need for additional nodes
* Test with N load balancers and with N routers
  * N load balancers would need to use multicast instead of UDP
* Fine-tune conntrackd/haproxy/keepalived/kernel parameters
  * https://bugzilla.redhat.com/show_bug.cgi?id=1365002
* IPv6 support (NB. OpenShift does not support IPv6 yet)
  * https://bugzilla.redhat.com/show_bug.cgi?id=871569
  * https://bugzilla.redhat.com/show_bug.cgi?id=1425552
  * https://bugzilla.redhat.com/show_bug.cgi?id=1436708

## License

Apache v2
