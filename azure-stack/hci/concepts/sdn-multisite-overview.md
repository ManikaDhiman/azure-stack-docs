---
title: SDN Multisite overview (preview)
description: Learn about SDN Multisite solution for Azure Stack HCI (preview)
author: alkohli
ms.author: alkohli
ms.topic: how-to
ms.date: 11/29/2023
---

# SDN Multisite overview

This article provides an overview of the Software Defined Networking (SDN) Multisite solution and its existing capabilities and limitations. This information can help you design an effective network topology and disaster recovery plan.

## What is SDN Multisite

SDN Multisite provides Layer 2 and Layer 3 connectivity for virtualized customer workloads spread across multiple sites. It also provides unified network policy management for the customer workloads, thus eliminating the need to update policies when a workload VM moves from one site to another.

In a multisite environment, one of the sites is a primary site and the other is a secondary site. The primary site is responsible for resource synchronization. If the primary site is unreachable, then resources cannot be updated through the secondary site. On the other hand, if the secondary site is unreachable, resources can be updated through the primary site.

To restore a secondary site, you must first extract the resources from the backup, then restore the prerequisite local resources that need to be present when global resources are synced. After this restore is complete, both sites' global resources need to be synced. The last call to the script will restore all local resources.

## Prerequisites 

There must be underlying physical network connectivity between two sites, and the Provider Network name must be the same on both sites. TCP port 49001 must be permitted by firewalls for cross-cluster communication. SDN must be installed on both sites separately, using SDN Express scripts or using Windows Admin Center. At least one of the two sites must not have any virtual networks and/or Network Security Groups
(NSGs) and/or user defined routes configured. The SDN MAC pool and the IP pools for the logical networks must not overlap between the two sites.

## Limitations

Currently, this feature is supported between 2 sites. Higher scale will be supported in future releases. Sites must be connected over a private network, as encryption support for sites connected over the Internet is not provided. Internal Load Balancing is not supported with this feature presently.

## Network Controller Peering

Peering is initiated like virtual network peering. A connection is initiated on both sites. One site is the initiator, and the other site is listening for the connection. Once a connection is established, peering becomes successful.

### Primary Site and Secondary Site

When multisite is configured, one of the sites becomes primary and another one is secondary. The primary site is responsible for resource synchronization. If the primary site is unreachable, then resources cannot be updated through the secondary site. On the other hand, if the secondary site is unreachable, resources can be updated through the primary site.

To restore a secondary site, you must first extract the resources from the backup, then restore the prerequisite local resources that need to be present when global resources are synced. After this restore is complete, both sites' global resources need to be synced. The last call to the script will restore all local resources.

## Software Load Balancers

Currently Software Load Balancers are still local resources for each of your data centers. This means that load balancing policies and configurations are not synced across sites through Multisite as it stands now.

This concept can be best illustrated through an example. In the figure below, we have two data centers, each with their own SDN infrastructure deployed and configured. Multisite is enabled. Let's say a client wants to reach VM1 with IP address 10.0.0.5 with VIP of 11.0.0.5. If you don't expect your VMs to migrate from one location to another, then data packets get forwarded as usual as illustrated below.

![](./media/media/image1.gif){width="6.490972222222222in"
height="3.6486111111111112in"}

However, if you decided to migrate one VM or all VMs sitting behind this VIP, then you will find that the VM you are trying to reach becomes unreachable or that your VIP will fail, respectively. This is due to load balancer resources being localized to each data center. As workloads move, the configurations on the MUXes aren't global and thus your other site isn't aware of migrations.

![](./media/media/image2.gif){width="6.497222222222222in"
height="3.660416666666667in"}

### Workaround

You might be asking yourself now how to accommodate such a limitation in your network. Here's a possible solution to this constraint. You can enable an external load balancer to test for MUXes that become unhealthy (i.e. backend VMs all move from one site to another as in our previous example). This external load balancer will ensure connectivity to workloads even as VMs move from one site to another.

## ![](./media/media/image3.png){width="6.5in" height="3.65625in"}

## Gateways

SDN gateway connections are not synchronized between sites. This means that each site has its own gateways and gateway connections. When a VM is created or migrated to a site, it gets local gateway configuration like gateway routes. If you create a gateway connection on one site for a particular virtual network, and not on the other site, the VMs on that virtual network will not get gateway connectivity when they move to the
other site. If you want VMs to retain gateway connectivity when they move to the other site, you must configure a separate gateway connection for the other site for the same virtual network.

## Backup, Restore, and Upgrade

**Backup:** When backup is initiated at one site, it is automatically initiated on the other site as well. This includes auto-backups. The backup file name for the latter site will be appended with the unique deployment Id of that site. If the backup was initiated at site1 and site2 does not have access to the backup folder, the backup for site2 will be stored on the Network Controller node in the same folder where
auto-backups are created. If logs share is configured, it will be later moved to this SMB share.

**Restore:** There are two scenarios for Network controller restore: one of the sites went down and needs to be restored while the other site is up and running, or both sites need to be restored. Both scenarios are supported by the RestoreReplay.ps1 script. The process for restoring in scenario 1 involves extracting the resources from the backup, restoring prerequisite local resources that need to be present when global
resources are synced, syncing both sites\' global resources, and restoring all local resources. The process for restoring in scenario 2 is similar to scenario 1, but it starts off by completely restoring one site.

**Upgrade:** Each site can be updated independently. It is recommended that both sites have the same version for SDN infrastructure (Network Controller, SLB MUX, and Gateways).

## FAQ

**What is SDN Multisite?**

SDN Multisite provides Layer 2 and Layer 3 connectivity for virtualized customer workloads spread across multiple sites. It also provides unified network policy management for the customer workloads, thus eliminating the need to update policies when a workload VM moves from one site to another.

**What are the limitations of SDN Multisite?** 

Currently, this feature is supported between 2 sites. Higher scale will be supported in future releases. Sites must be connected over a private network, as encryption support for sites connected over the Internet is not provided. Internal Load Balancing is not supported with this feature presently.

**What are the prerequisites for using SDN Multisite?** 

There must be underlying physical network connectivity between two sites, and the Provider Network name must be the same on both sites. TCP port 49001 must be permitted by firewalls for cross-cluster communication. SDN must be installed on both sites separately, using SDN Express scripts or using Windows Admin Center. At least one of the two sites must not have any virtual networks and/or Network Security Groups
(NSGs) and/or user defined routes configured. The SDN MAC pool and the IP pools for the logical networks must not overlap between the two sites.

**How do I set up Multisite Peering?** 

Site peering must be initiated from both sites using the Set-NetworkControllerMultisiteConfiguration cmdlet. This cmdlet uses the REST certificate on each site for authentication between Network Controller clusters. First, peering is initiated from site1 and then from site2.

**What is the difference between a primary and secondary site?** 

When multisite is configured, one of the sites becomes primary and another one is secondary. The primary site is responsible for resource synchronization. If the primary site is unreachable, then resources cannot be updated through the secondary site. On the other hand, if the secondary site is unreachable, resources can be updated through the primary site. During multisite peering, the primary site is
automatically selected. Later it can be changed using the Set-NetworkControllerMultisitePrimary cmdlet.

**How do I check the peering state?** 

To check the peering state, you can use the

Get-NetworkControllerMultisiteConfiguration cmdlet. If peering is
configured, NetworkControllerMultisiteProperties.Sites contains
information about remote sites. NetworkControllerSite has properties
such as IsPrimary, RestIPAddress, State, DeploymentId, ApiVersion, and
ConfigurationState.

**How do I remove peering?** 

Peering can be removed using the
Set-NetworkControllerMultisiteConfiguration cmdlet. When peering is
removed, resource synchronization is aborted, and each site keeps its
own copy of resources.

**What resources are synchronized between sites?** 

Once peering is established, the following policies will be synchronized
between sites: Virtual Networks, Network Security Groups (NSGs), and
User Defined Routes. These resources can be updated through any site,
and the system will internally ensure that they are synced across sites.

**Is Load Balancing synchronized between sites?** 

No, Load balancing policies and Virtual IP addresses (VIPs) are not synchronized across sites. These policies are created on the local site, and if you want the same policies on the other site, you can create them on the other site. If your backend VMs for load balancing policies are located on a single site, then connectivity over SLB will work fine without any additional configuration. But, if you expect the backend VMs
to move from one site to the other, by default, connectivity will only work if there are any backend VMs behind a VIP on the local site. If all the backend VMs move to another site, connectivity over that VIP will fail.
