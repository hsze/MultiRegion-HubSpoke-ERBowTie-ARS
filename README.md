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
