# fail2ban_iptables_docker
Custom actions for fail2ban to block traffic also on Docker-Containers.

## Situation
Each time `fail2ban` creates the first ban for a jail, it creates a chain in the format `f2b-<name>`. If `fail2ban` now creates a chain that uses the sshd filter, the chain `f2b-sshd` is created. In addition, a target is created in the `INPUT` chain on the previously created chain `f2b-sshd`.

```text
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
f2b-sshd   tcp  --  anywhere             anywhere

Chain f2b-sshd (2 references)
target     prot opt source               destination         

REJECT     all  --  186.67.248.6         anywhere             reject-with icmp-port-unreachable
REJECT     all  --  171.111.192.1        anywhere             reject-with icmp-port-unreachable
REJECT     all  --  165.22.62.225        anywhere             reject-with icmp-port-unreachable
REJECT     all  --  165.22.22.227        anywhere             reject-with icmp-port-unreachable
RETURN     all  --  anywhere             anywhere
```

If a packet reaches the `INPUT` chain, the packet filter jumps to the chain `f2b-sshd` - if no other blocking rule was found before. If no rule is found there that blocks the packet, the packet is finally accepted. If a blocking rule was found in `f2b-sshd`, the packet is rejected.
The same logic is applied to other filters. So it makes no difference if it is the chain `f2b-nginx`, `f2b-dovcot` or another chain created by fail2ban.

## Problem
Incoming packets for Docker containers are handled by the `FORWARD` chain rather than the `INPUT` chain, so the rules in the `INPUT` chain have no relevance to fail2ban and Docker containers. However, we may also want to block packets from blocked clients for Docker containers.

## Official documentation
The official Docker documentation states that custom packet filter rules should be created in the `DOCKER-USER` chain.  
When Docker-services is started, it creates, among other things, the `DOCKER-USER` chain and the corresponding target in the `FORWARD` chain.

```text
Chain FORWARD (policy DROP)
target     prot opt source               destination         
DOCKER-USER  all  --  anywhere             anywhere            
DOCKER-ISOLATION-STAGE-1  all  --  anywhere             anywhere            
ACCEPT       all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER       all  --  anywhere             anywhere            
ACCEPT       all  --  anywhere             anywhere            
ACCEPT       all  --  anywhere             anywhere
```

## Solution
Using the existing action iptables-allports.conf as an example, a new action iptables-allports-with-docker.conf was created.  
`actionstart` has been extended with `<iptables> -N DOCKER-USER` and `<iptables> -I DOCKER-USER -j f2b-<name>`. `<iptables> -N DOCKER-USER` first creates the chain `DOCKER-USER`, since fail2ban is usually started before fail2ban and this chain therefore does not yet exist. `<iptables> -I DOCKER-USER -j f2b-<name>` creates a target in the chain `DOCKER-USER` to the corresponding chain `f2b-<name>` created before.  
  
`actionstop` has been extended by `<iptables> -D DOCKER-USER -j f2b-<name>` so that the target on the previously deleted chain is removed from the chain DOCKER-USER again.

## Conclusion
With a small adaptation of the existing actions for `iptables`, incoming packets from already blocked IP's can also be blocked for Docker containers without having to hold block twice in another chain.
