
# Calico's External Firewall Controller 


## Problem statement
Project Calico has a lot of security features to manage traffic rules from/to nodes and PODs on clusters. This is performed by Felix agent running on nodes witch is responsible to insert control plane rules on linux kernel (routes, netilter tables rules, etc.). However this rules are restricted to cluster itself, where Felix is running. Despite the high level of confidence on Linux kernel security features it is common sense that the closer to the edge you stop suspicious traffic is better. Is not a bad idea to have two layer of filtering too.

It's usual on companies to have a border firewall that helps on information security tasks, with a single tool to manage all control lists. Checkpoint, Cisco, Fortinet, Palo Alto are just few examples of firm selling this kind of equipment. Security rules in this appliances are tipically a list of IP address (source and destination) where some services are allowed or denied to flow. It's very common to have this equipment carrying out of network address translation (NAT) and intrusion prevention system (IPS) features too. 

These firewalls are built over a traditional network foundation, where services are tipically served by hosts with fixed IP address. The dynamic nature of pods in clusters like Kubernetes has brought us new challenges on how to manage border firewall rules. Some kind of rules are not so much affected by this problem (ingress): we still may have a public IP address fixed on cluster to serve e.g. http with a static rule on border firewall. To inbound traffic the traditional aprouch still not too bad (or complicated) to manage. 

However we may face problems to handle rules on border firewall when is required to:

1. filter the egress traffic from pods to a specific group of destination addresses and services; 
2. create outbound NAT rules to a set of pods witch is allowed to egress and haven't public IP addresses available on cluster;
3. create outbound NAT rules to a set of pods witch need to be SNATed to specific address to access external services (destination with source IP firewall filter)


If we want a second layer of filtering on (1) or solve problems (2) and (3), the current options may be:

For (1):
 - For each service create a static network rule on border firewall. The rule needs to have a network allocated with the maximum number os pods accessing the service. Allocate this net to IPPool and manage it on Calico to assign it to specific pods.

For (2) and (3):
 - Pretty like (1). Create static NAT rules on firewall to a set of IP addresses, assigned them to IP Pool and manage on Calico what POD should receive this static IP address range.


Obviously this solution is far from ideal. Too much work to manage this rules and IPPools (one IPPool to each src/dst/service rule); duplicated set of rules databases (at calico and border firewall); we also may have a big IP address waste allocating source addresses ranges.


Note 1: Calico's entreprise seems to allow pods be SNATed by namespace to different fixed IPv4 addresses. So, all above problems can be solved with static rules on border firewall (one source address to each service), it's a very cool feature. However, we still face (1) when running on IPv6. Despite NAT is possible on IPv6, don't seem too much elegant solution.

Note 2: Above discussion only make sense if you have pods routeable on your network, not performing NAT on cluster itself (masquerade). If you have NAT on ingress/egress cluster traffic your border firewall is "blind" about PODs IP addresses - you can't rule on it.


## Proposal

One solution for this problems is to allow Calico to control a set of rules on border firewall dynamically. Just like Felix manages the netfilter rules, we can allow it to include, change and delete an subset of rules on external firewall (referenced just as firewall from now). Firewall APIs are proprietary for each kind of vendor. So, for each vendor we need an specific connector to handle it: modern firewalls have REST http APIs but other may have only the CLI available for that.


At Calico side (Felix), the proposed component (cextfire) is attached as external dataplane driver [1]. For sake of simplity no changes is needed on Calico's usual pods. We just need one more pod on cluster running cextfire attached to a Felix instance. cextfire should receive all Calico's messages and keep an consistent table of the currents network policies rules. This table should me translated to a particular set of rules keeping the firewall database also in a consistency state.

Besides solving the issues previously discussed is also possible to minimize the number of static rules on firewall even for ingress cluster traffic, enforcing an consistent state between cluster network policies and external firewall database.


                       +-------+             +---------------+                             +---------------------+                +-----------+
                       | Felix |             | Cextfire_main |                             | Cextfire_connector  |                | Firewall  |
                       +-------+             +---------------+                             +---------------------+                +-----------+
-------------------------\ |                         |                                                |                                 |
| network policies rules |-|                         |                                                |                                 |
|------------------------| |                         |                                                |                                 |
                           |                         |                                                |                                 |
                           | sync/update msg         |                                                |                                 |
                           |------------------------>|                                                |                                 |
                           |                         | ------------------------------------\          |                                 |
                           |                         |-| process and manage                |          |                                 |
                           |                         | | a consistent state rules database |          |                                 |
                           |                         | |-----------------------------------|          |                                 |
                           |                         | wait sync state                                |                                 |
                           |                         |----------------                                |                                 |
                           |                         |               |                                |                                 |
                           |                         |<---------------                                |                                 |
                           |                         |                                                |                                 |
                           |                         | database rules changes                         |                                 |
                           |                         |----------------------------------------------->|                                 |
                           |                         |                                                | -----------------\              |
                           |                         |                                                |-| database rules |              |
                           |                         |                                                | |----------------|              |
                           |                         |                                                |                                 |
                           |                         |                                                | translate to Firewall API       |
                           |                         |                                                |--------------------------       |
                           |                         |                                                |                         |       |
                           |                         |                                                |<-------------------------       |
                           |                         |                                                |                                 |
                           |                         |                                                | apply rules subset              |
                           |                         |                                                |-------------------------------->|
                           |                         |                                                |                                 |


[1] https://github.com/projectcalico/felix/blob/master/proto/doc.go



## Configure options
- firewall rules prefix
- firewall NAT rules -- we need to check how to embedded this info on network policy.
- to be defined


## Contribution
We appreciate your opinion. Please fell free to share ideas about this proposal. Will help a lot, specially if you have an easy way to solve these problems.
For now thanks to Shaun and Robson for the tips on Calico's slack channel.


```
https://textart.io/sequence
object Felix Cextfire_main Cextfire_connector Firewall
note left of Felix: network policies rules
Felix->Cextfire_main: sync/update msg
note right of Cextfire_main: process and manage\na consistent state rules database
Cextfire_main->Cextfire_main: wait sync state
Cextfire_main->Cextfire_connector: database rules changes
note right of Cextfire_connector: database rules
Cextfire_connector->Cextfire_connector: translate to Firewall API
Cextfire_connector->Firewall: apply rules subset

```
