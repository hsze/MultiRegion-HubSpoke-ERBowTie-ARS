# Secure Multi-Region Hub-Spoke Design with ExpressRoute Bow-Tie and Azure Route Server
## Overview
Azure customers have workload presence in multiple regions.  The Hub-Spoke design using first- or third-party firewall Network Virtual Appliances (NVAs) to secure North-South (N-S) and East-West (E-W) traffic is a proven and well documented [deployment model](https://docs.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/hub-spoke-network-topology).  In addition, Enterprise customers who require reliable connectivity to support hybrid networking have adopted ExpressRoute as the technology of choice, very often using the highly available [“Bow-Tie” design](https://docs.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering#large-distributed-enterprise-network). This article describes a method of scaling and simplifying this configuration by minimizing User Defined Routes (UDRs) while maintaining the principles and components of the design: ExpressRoute, “Bow-Tie,” Hub-Spoke, firewall NVA.
## Classic Use Case
-	Customer has hub-spoke design with dozens of spokes
-	Customer has this hub-spoke design stamped across multiple regions
-	Customer leverages ExpressRoute with the “Bow-Tie” configuration for Hybrid Networking
-	Customer requires East-West (E-W) and North-South (N-S) firewalling
### Scenario
-	In the Hub-Spoke model, typically the combination of “Use Remote Gateway” and “Allow Gateway Transit” is used to allow spokes to learn the OnPrem prefixes, and for OnPrem to learn about the spoke prefixes.  
-	To force spoke data traffic through a NVA firewall in the Hub, a Route Table is associated with Spoke subnets with the configuration “Propagate gateway routes” set to “No.”  A simple one-line User Defined Route (UDR) is defined with default route 0/0 pointing to the NVA firewall as next-hop.
-	The NVA vNIC is in the hub has awareness of all the peered Spoke VNET prefixes the OnPrem routes. 
-	UDRs are required on the GatewaySubnet to ensure traffic from OnPrem to Spoke VNET is symmetrically routed through the stateful NVA. The UDRs need to EXPLICITLY match the spoke prefixes (or be more specific), since a less-specific UDR will be bypassed if there is a more-specific prefix learned by system route.
-	In the ExpressRoute “Bow-Tie” configuration, the remote region VNET prefixes are also propagated to the ExpressRoute Gateway, and learned at the GatewaySubnet as well as the Hub NVA vNIC.  To ensure traffic is not hairpinned through the “Bow-Tie” (i.e. go down to Microsoft Edge and back), the NVA vNIC subnet would need to be associated to a Route Table that contains UDRs to override each of the cross region prefixes, with Next Hop (NH) of the Remote region NVA.  Again, the UDRs needs to EXPLICITLY match the prefix (or be more specific), since a less-specific UDR will not be evaluated when there is a more-specific prefix.

Below is an illustration of the classic design using AzureFirewall as a collapsed E-W and N-S firewall.
![Classic](/Diagrams/1-classic.jpg)

## An Alternate Design using Route Server + NVA
The Classic Use Case described above has been proven successful with countless customer implementations. However, as the number of spokes in each region grows beyond dozens to hundreds, some issues may arise:
-	Overriding UDRs for each of the prefixes to force traffic to bypass the ER Bow-Tie hairpin and through remote firewall can be cumbersome, even with automation 
-	When the total number of remote VNET prefixes is greater than 400, the [limit of number of entries](https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-resource-manager-virtual-networking-limits) on the NVA subnet route table has been reached.
  
Now that [Azure Route Server](https://docs.microsoft.com/en-us/azure/route-server/overview) has reached General Availability, an alternative solution with simplified configuration is possible.  The main points and caveats are
-	“Use Remote Gateway” and “Allow Gateway Transit” are disabled on the spoke-hub peering links, which has the effect of preventing OnPrem from learning spoke prefixes.  It also has the effect of preventing the hub from learning remote-region spoke prefixes (which was previously leaked by the Bow-Tie).
-	For illustration purposes, the N-S firewall function is separated from the E-W firewall. 
-	As with the Classic design, the spoke subnets still have a simple route table with a single UDR entry of Default 0/0 route pointing to Next-Hop IP of the E-W NVA Firewall in Hub
-	The E-W NVA vNIC sits in the Hub and learns the OnPrem routes.  It is also aware of the remote Hubs prefixes, learned via Global VNET peering.  It is NOT aware of the remote spoke prefixes as they are no longer propagated via the ER Hairpin.
-	Azure Route Server is deployed in the Hub and establishes eBGP peering with the N-S firewall NVA.  This N-S firewall NVA is ALS originating a supernet summary of the Local Region.  This is important for OnPrem reachability.  By originating this supernet, OnPrem learns of the Local region as an aggregate route.  This prevents traffic from OnPrem to local Spoke from being blackholed at the Microsoft Edge. 
-	The E-W Firewall NVA subnet has a UDR to the remote region’s supernet prefix with NH IP of the the remote region’s E-W NVA Firewall.  This allows fr reachability to remote spokes.  Because supernets can be used, very few UDR statements in the route table are required.
-	The E-W Firewall NVA subnet has a UDR for default 0/0 with NH IP of the N-S Firewall NVA trusted interface.  The E-W Firewall has awareness of the OnPrem routes, local prefixes, and remote region supernets, so anything which does match is bound for the Internet to be analyzed at the N-S firewall. 
-	The GatewaySubnet will still require a UDRs for each local spoke.

### Configuration 
The following configuration was tested to validate the design:
![TestConfig](/Diagrams/2-testconfig.jpg)

### Environment:
-	WestUS2 and EastUS2 are used.  WestUS2 Regions use the supernet 10.1.0.0/16, broken down into /20s for Hub and Spoke VNETs.  EastUS2 Regions use the supernet 10.2.0.0/16, broken down into /20s for the Hub and Spoke VNETs.
-	A single ExpressRoute circuit using Megaport as Provider, with a Primary path of 172.16.10.0/30 and a Secondary path of 172.16.11.0/30, is connected to ExpressRoute Gateways in both EastUS2 Hub and WestUS2 Hub.
-	Azure Firewall is used for E-W Firewalling.  A Cisco CSR is used for N-S Firewalling.  AzFW is deployed in “Force Tunneled” mode, and a AzureFirewallManagementSubnet is defined.
-	Each Spoke Subnet is associated with a Route Table with a simple UDR for default 0/0 with Next-Hop of the private IP of its local AzFW (10.x.2.4)
-	The CSR serves not only as N-S Firewall but is also originating a summary local Region route, e.g. 10.x.0.0/16.  This is critical so OnPrem to Spoke traffic does not blackhole at the Microsoft Edge. Below shows the Megaport OnPrem has learned the supernets through ExpressRoute.

![Megaport](/Diagrams/3-MegaportRT.png)
-	GatewaySubnet is associated with Route Table with UDR pointing to the AzureFirewall for each spoke prefix, and for the supernet of the remote region (for failure scenario).
![GatewaySubnet](/Diagrams/4-GatweaySubnet.png)
-	AzureFirewallSubnet is associated with Route Table with UDR pointing to the remote AzureFirewall private IP for the remote supernet, and for the remote Hub.  It has an additional UDR for default 0/0, pointing to the CSR Trusted Private IP.
![AzureFirewallSubnet](/Diagrams/5-AzureFirewallSubnet.png)
-	CSR uses the Azure-assigned Static Public IP address on its Untrusted Interface to reach Internet.

## Validation of configuration
The below shows the validation of Spoke to Spoke (InterHub), Spoke to OnPrem, and Spoke to Internet
### InterRegion Spoke to Spoke
A TCP session (SSH) is successfully established between VM in EastUS2 Spoke to VM in WestUS2 Spoke, traversing both the local and remote firewall

![SSH](/Diagrams/6-CrossRegionSSH.png)

### Spoke to OnPrem
EastUS2 Spoke VM is able to reach OnPrem.

![OnPrem](/Diagrams/7-OnPrem.png)   

### Spoke to Internet (via AzFW and then CSR)
EastUS2 Spoke VM is able to reach the Internet via both ICMP and via wget and curl.  The below illustrates the IP address presented to the Internet is the IP of the Public IP of the CSR Unrusted Interface (137.116.63.88).  Traffic from Spoke hits E-W AzFW and then "force tunnels" to the N-S CSR firewall.  The CSR can be replaced with a stateful firewall sandwich with load balancers.

![ToInternet](/Diagrams/8-SpoketoInternet.png)   

## Summary
Many customers have adopted the tried-and-true Hub-Spoke design with NVA in Hub, with ER Bow-Tie. There are scale and management considerations with such a design.  This article introduced a configuration leveraging Azure Route Server, conveniently using the N-S firewall NVA to propagate supernet routes, which allows for simplification of the UDR configuration.  

## Futures
- E-W and N-S may be consolidated in the AzFW, but an NVA is still required to originate the Supernets.  A future iteration will show AzFW as the consolidated Firewall, with an NVA serving as the sole function for injecting “dummy routes”.
- A future iteration of doc will use OpenSource NVA for N-S firewall, leveraging [OPNSense](https://github.com/dmauser/opnazure).

## Acknowledgements
Many Github contributions have discussed this general topic and need to be acknowledge.  This article is an iteration focused on the Cross-Region “Bow-Tie” route-leaking considerations.

1. [How to use Azure Firewall for intra/inter-region Hub and Spoke traffic filtering in Virtual Networks](https://github.com/jwrightazure/lab/tree/master/inter-region-spoke-spoke-azfw)
2. [Route Server Multi-Region Design](https://blog.cloudtrooper.net/2021/03/06/route-server-multi-region-design/)
3. [Inspecting Traffic across ExpressRoute Circuits in Azure](https://github.com/jocortems/azurehybridnetworking/tree/main/Inspect-Traffic-Between-ExpressRoute-Circuits)
4. [Forced Tunneling of Internet traffic through Active-Active OPNsense Firewalls using Azure Route Server (ExpressRoute)](https://github.com/dmauser/Lab/tree/master/RS-AA-OPNsense-ForceTunnel-ER)
