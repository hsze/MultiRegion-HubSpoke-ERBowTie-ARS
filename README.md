# Secure Multi-Region Hub-Spoke Design with ExpressRoute Bow-Tie and Azure Route Server
## Overview
Azure customers have workload presence in multiple regions.  The Hub-Spoke design using First- or Third-Party Network Virtual Appliances (NVAs) to secure North-South (N-S) and East-West (E-W) traffic is a proven and well documented [deployment model](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/hub-spoke-network-topology).  In addition, Enterprise customers who require reliable connectivity to support hybrid connectivity have adopted ExpressRoute as the technology of choice, very often using the highly available [“Bow-Tie” design](https://docs.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering#large-distributed-enterprise-network). This article describes a method of scaling and simplifying this configuration by minimizing User Defined Routes (UDRs) while maintaining the principles and components of the design: ExpressRoute, “Bow-Tie,” Hub-Spoke, Secure NVA.
## Classic Use Case
-	Customer has hub-spoke design with dozens of spokes
-	Customer has this hub-spoke design stamped across multiple regions
-	Customer leverages ExpressRoute with the “Bow-Tie” configuration for Hybrid Networking
-	Customer requires East-West (E-W) and North-South (N-S) firewalling
### Scenario
-	In the Hub-Spoke model, typically the combination of “Use Remote Gateway” and “Allow Gateway Transit” is used to allow spokes to learn the OnPrem prefixes, and for OnPrem to learn about the spoke prefixes.  
-	To force spoke traffic data flow through the NVA firewall in the hub, a Route Table is associated with Spoke subnets with the configuration “Propagate gateway routes” set to “No.”  A simple one- line User Defined Route (UDR) is defined with default route 0/0 pointing to the NVA firewall as next-hop.
-	The NVA vNIC is in the hub and is aware of all the peered Spoke VNET prefixes the OnPrem routes. 
-	UDRs are required on the GatewaySubnet to ensure traffic from OnPrem to Spoke VNET is symmetrically routed through the NVA. The UDR needs to EXPLICITLY match the spoke prefixes (or be more specific), since a supernet UDR will be bypassed if there is a more specific prefix learned by system route.
-	In the ExpressRoute “Bow-Tie” configuration, the remote region Hub VNET prefixes are also learned via the ExpressRoute Gateway, and learned at the GatewaySubnet as well as the NVA vNIC Subnet.  To ensure traffic is not hairpinned through the “Bow-Tie” (i.e. go down to Microsoft Edge and back), the NVA vNIC subnet would need to be associated to a Route Table that contains UDRs to override each of the cross region prefixes, pointing to the Remote region NVA.  Again, the UDRs needs to EXPLICITLY match the prefix (or be more specific), since a supernet UDR will not be evaluated when there is a more specific prefix.

Below is an illustration of the classic design using AzureFirewall as a collapsed E-W and N-S firewall.
![Classic](/Diagrams/1-classic.jpg)

## An Alternate Design using Route Server + NVA
The Classic Use Case described above has been proven to work with numerous customer implementations. However, as the number of spokes in each region grows beyond dozens to hundreds, some issues arise:
-	Overriding UDRs for each of the prefixes to force traffic through remote firewall becomes cumbersome, even with automation 
-	When the total number of remote VNET prefixes is greater than 400, the [limit of number of UDR](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits) on the NVA subnet route table has been reached  
Now that [Azure Route Server](https://docs.microsoft.com/en-us/azure/route-server/overview) has reached General Availability, an alternative solution with simplified configuration is possible.  The main points and caveats are
-	“Use Remote Gateway” and “Allow Gateway Transit” are disabled on the spoke-hub peering links. This prevents the spokes from learning the OnPrem prefixes, as well as the hub from learning remote-region spoke prefixes.
-	For illustration purposes, the N-S firewall function is separated from the E-W firewall. 
-	As with the Classic design, the spoke subnets still have a simple route table with a single UDR entry of Default 0/0 route pointing to Next-Hop IP of the E-W NVA Firewall in Hub
-	The E-W NVA NIC learns the OnPrem routes.  It is also aware of the remote Hubs prefixes, learned via Global VNET peering.  It is NOT aware of the remote spoke prefixes as they are no longer propagated via the ER Hairpin.
-	Azure Route Server is deployed in the Hub and peers with the N-S NVA which is originating a supernet summary of the Local Region.  This is important for OnPrem reachability as Use Remote Gateway and Allow Gateway Transit flags have been disabled.  By originating this supernet, the OnPrem learns of the Local region as an aggregate route.  This prevents traffic from OnPrem to local Spoke from being blackholed. 
-	The E-W Firewall NVA subnet can have a UDR to the remote region’s supernet prefix with the next hop of the the remote region’s E-W NVA Firewall.  This allows very few UDR statements in the route table.
-	The E-W Firewall NVA subnet will also have a UDR for default 0/0 pointing to the N-S Firewall NVA trusted subnet.  As the NVA subnet has routes for of all on-prem and remote region supernet, anything which does match is bound for the Internet (N-S).
-	The GatewaySubnet will still require a UDR for each local spoke, the remote hub, and will require a UDR for a supernet of the remote region.

## Configuration 
The following configuration was tested to validate the design:
![TestConfig](/Diagrams/2-testconfig.jpg)

### Environment:
-	WestUS2 and EastUS2 are used.  WestUS2 Regions use the supernet 10.1.0.0/16, broken down into /20s for Hub and Spoke VNETs.  EastUS2 Regions use the supernet 10.2.0.0/16, broken down into /20s for the Hub and Spoke VNETs.
-	A single ExpressRoute circuit, with a Primary path of 172.16.10.0/30 and a Secondary path of 172.16.11.0/30, is connected to ExpressRoute Gateways in both EastUS2 Hub and WestUS2 Hub.
-	Azure Firewall is used for E-W Firewalling.  A Cisco CSR is used for N-S Firewalling.  AzFW is deployed in “Force Tunneled” mode, with a AzureFirewallManagementSubnet.
-	Each Spoke Subnet is associated with a Route Table with a simple UDR for default 0/0 with Next-Hop of the private IP of its local AzFW (10.x.2.4)
-	The CSR serves not only as N-S Firewall but is also originating a summary local Region route, e.g. 10.x.0.0/16.  This is critical so OnPrem to Spoke traffic does not blackhole at the Microsoft Edge. Below shows ExpressRoute has learned the supernets.

![Megaport](/Diagrams/3-MegaportRT.png)
-	GatewaySubnet is associated with Route Table with UDR pointing to the AzureFirewall for each spoke prefix, and for the supernet of the remote region (for failure scenario).
![GatewaySubnet](/Diagrams/4-GatweaySubnet.png)
-	AzureFirewallSubnet is associated with Route Table with UDR pointing to the remote AzureFirewall private IP for the remote supernet, and for the remote Hub.  It has an additional UDR for default 0/0, pointing to the CSR Trusted Private IP.
![AzureFirewallSubnet](/Diagrams/5-AzureFirewallSubnet.png)
-	CSR uses the Azure-assigned Static Public IP address on its Untrusted Interface to reach Internet.

## Validation
The below shows the validation of Spoke to OnPrem, Spoke to Spoke (InterHub) and Spoke to Internet
### Spoke to OnPrem
AzureAdmin@VM1-EastUS2-Spoke12:~$ ping 172.16.10.1
PING 172.16.10.1 (172.16.10.1) 56(84) bytes of data.
64 bytes from 172.16.10.1: icmp_seq=1 ttl=252 time=69.4 ms
64 bytes from 172.16.10.1: icmp_seq=2 ttl=252 time=68.7 ms
64 bytes from 172.16.10.1: icmp_seq=3 ttl=252 time=68.7 ms
64 bytes from 172.16.10.1: icmp_seq=4 ttl=252 time=68.7 ms
^C
--- 172.16.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 68.756/68.937/69.461/0.441 ms

### InterHub Spoke to Spoke
AzureAdmin@VM1-EastUS2-Spoke12:~$ ssh 10.1.33.4
AzureAdmin@10.1.33.4's password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.4.0-1055-azure x86_64)

AzureAdmin@VM1-WestUS2-Spoke12:~$ netstat -a
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:25324           0.0.0.0:*               LISTEN     
tcp        0      0 localhost:domain        0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN     
tcp        0      0 vm1-westus2-spoke:41474 13.71.195.200:https     TIME_WAIT  
**tcp        0    316 vm1-westus2-spoke12:ssh 10.2.33.4:41082         ESTABLISHED**

tcp        0      0 vm1-westus2-spoke:56690 169.254.169.254:http    TIME_WAIT  
tcp6       0      0 [::]:ssh                [::]:*                  LISTEN     
udp        0      0 localhost:domain        0.0.0.0:*                          
udp        0      0 vm1-westus2-spok:bootpc 0.0.0.0:*                          
udp        0      0 localhost:25224         0.0.0.0:*                          
raw6       0      0 [::]:ipv6-icmp          [::]:*                  7        

### Spoke to Internet (via AzFW and then CSR)
AzureAdmin@VM1-EastUS2-Spoke12:~$ curl ifconfig.io
137.116.63.88

(137.116.63.88 is the Public IP of the CSR Untrusted Interface)

## Summary
Many customers have adopted the tried-and-true Hub-Spoke design with NVA in Hub, with ER Bow-Tie. There are scale and management considerations with such a design.  This article introduced a configuration leveraging Azure Route Server with an NVA propagating Supernet routes, which allows for simplification of the UDR configuration.  

## Futures
- E-W and N-S may be consolidated in the AzFW, but an NVA is still required to originate the Supernets.  A future iteration will show AzFW as the consolidated Firewall, with an NVA serving as the sole function for injecting “dummy routes”.
- A future iteration of doc will use OpenSource NVA for N-S firewall, leveraging [OPNSense](https://github.com/dmauser/opnazure).

## Acknowledgements
Many Github contributions have discussed this general topic and need to be acknowledge.  This article is an iteration focused on the Cross-Region “Bow-Tie” route-leaking considerations.

1. [How to use Azure Firewall for intra/inter-region Hub and Spoke traffic filtering in Virtual Networks](https://github.com/jwrightazure/lab/tree/master/inter-region-spoke-spoke-azfw)
2. [Route Server Multi-Region Design](https://blog.cloudtrooper.net/2021/03/06/route-server-multi-region-design/)
3. [Inspecting Traffic across ExpressRoute Circuits in Azure](https://github.com/jocortems/azurehybridnetworking/tree/main/Inspect-Traffic-Between-ExpressRoute-Circuits)
4. [Forced Tunneling of Internet traffic through Active-Active OPNsense Firewalls using Azure Route Server (ExpressRoute)](https://github.com/dmauser/Lab/tree/master/RS-AA-OPNsense-ForceTunnel-ER)
