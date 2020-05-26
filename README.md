
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


Obviously this solution is far from ideal. Too much work to manage this rules and IPPools (one IPPool to each src/dst/service rule); also we may have a big IP address waste allocating source addresses ranges (and IPv4 public address are now precious).


Note 1: Calico's entreprise version seems to allow pods be SNATed by namespace to different fixed IPv4 addresses. So, all above problems can be solved with static rules on border firewall (one source address to each service), it's a very cool feature. However, we still face (1) when running on IPv6. Despite NAT is possible on IPv6, don't seem too much elegant solution.

Note 2: Above discussion only make sense if you have pods routeable on your network, not performing NAT on cluster itself (masquerade). If you have NAT on ingress/egress cluster traffic your border firewall is "blind" about PODs IP addresses - you can't rule on it.


## Solution


## Configure options

## Feature in bullets


##Contribution

