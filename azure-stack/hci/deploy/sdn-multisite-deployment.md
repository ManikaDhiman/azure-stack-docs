---
title: Deploy SDN Multisite (preview)
description: Learn how to deploy a multisite SDN solution in Azure Stack HCI (preview)
author: alkohli
ms.author: alkohli
ms.topic: how-to
ms.date: 11/17/2023
---

# Deploy SDN Multisite in Azure Stack HCI

This article describes how to deploy the Software Defined Networking (SDN) Multisite solution in Azure Stack HCI. If you'd like an in-depth description of how this feature works with other SDN components, such as Software Load Balancer or Gateways, see [deep-dive link].

SDN Multisite enables you to have native Layer 2 (L2) and Layer 3 (L3) connectivity across multiple physical locations for SDN deployments.

## Benefits

Here are the benefits of using SDN Multisite:

- **Unified policy management system.** Manage and configure your networks across multiple locations from a single primary location, with shared virtual networks and policy configurations.
- **Seamless workload migration.** Seamlessly migrate workloads across phyical locations without having to reconfigure IP addresses or pre-existing Network Security Groups (NSGs).
- **Automatic reachability to new VMs.** Get automatic reachability to newly created virtual machines (VMs) and automatic manageability to any of their associated NSGs across your physical locations.

## Capabilities and limitations

The following table provides the capabilities and limitations of the SDN Multisite feature in this release:

| Capabilities | Limitations |
|--|--|
| Software Load Balancers | Limited to 2 physical locations |
| Gateways | Software Load Balancers |
| Native Layer 2 and Layer 3 connectivity | Gateways |
| Unified policy management |  |

## Prerequisites

Before you begin the deployment, ensure the following prerequisites are met:

- There must be underlying [physical network connectivity](../concepts/plan-software-defined-networking-infrastructure.md#physical-and-logical-network-configuration) between two sites. Additionally, the provider network name must be the same on both sites.

- Your firewall configuration must permit TCP port 49001 for cross-cluster communication.

- SDN must be installed on both sites separately, using [Windows Admin Center](./sdn-wizard.md) or [SDN Express scripts](../manage/sdn-express.md). Consequently, SDN components, such as Network Controller VMs, Software Load Balancer Multiplexor VMs, and SDN Gateway VMs are unique to each site.

- One of the two sites must not have any virtual networks, NSGs, and user defined routes configured.

- The SDN MAC pool must not overlap between the two sites.

- The IP pools for the logical networks, including Hyper-V Network Virtualization Provider Address (HNV PA), Public VIP, Private VIP, Generic Routing Encapsulation (GRE) VIP, and L3 must not overlap between the two sites.

## Deploy SDN Multisite

You can deploy SDN multisite using Windows Admin Center or PowerShell.

# [Windows Admin Center](#tab/windows-admin-center)

Follow these steps to deploy SDN Multisite using Windows Admin Center:

1. In Windows Admin Center, connect to your cluster in the primary site to begin peering across locations . Under **Tools**, scroll down to the **Networking** section, and select **Network controllers**.

1. On the **Network Controllers** page, select **New** to add a secondary site.

1. On the **Connect clusters** pane on the right, provide the necessary REST information:

    1. Enter the **Name** of this pairing. For example, **Secondary Site**.

    1. Enter the **Network Controller REST Uri** of the secondary site.

    1. Enter the **Cluster name for new site** or secondary site.
    
    1. Enter the **Network Controller VM name for new site** or secondary site.
    
    1. Enter the **Network Controller VM name for** your primary location.

1. Select **Submit**.

    :::image type="content" source="./media/sdn-multisite/deploy-sdn-multisite.png" alt-text="Deploy SDN Multisite using Windows Admin Center" lightbox="./media/sdn-multisite/deploy-sdn-multisite.png" :::

Once peering is initiated, resources such as virtual networks or policy configurations that were once local to the primary site become global resources synced across locations. 

# [PowerShell](#tab/powershell)

Follow these steps to deploy SDN Multisite using PowerShell.

1. To enable SDN Multisite, you must initiate peering from both sites using the `Set-NetworkControllerMultisiteConfiguration` cmdlet. The full details behind the commandlet are as follows:

    Here's the syntax of `Set-NetworkControllerMultisiteConfiguration`:

    ```powershell
    Set-NetworkControllerMultisiteConfiguration    [[-Tags] <PSObject>]    [-Properties] <NetworkControllerMultisiteProperties>  [[-Etag] <String>] [[-ResourceMetadata] <ResourceMetadata>]   [-Force] -ConnectionUri <Uri> [-CertificateThumbprint <String>] [-Credential <PSCredential>] [-PassInnerException] [-WhatIf] [-Confirm] [<CommonParameters>]
    ```

1. To initiate peering across two sites using a Certificate Authority, you'll have to create the `NetworkControllerMultisiteProperties` object for each site. You'll have to provide:

    1. REST Certificate of Site Peer for NetworkControllerMultisiteProperties object

    1. NetworkControllerSite object for Site Peer

    1. REST IP address of Site Peer for

    1. Encoded Certificate of Site Peer (If you're using self-signed certificates).

    ```powershell
    $cert1 = Get-ChildItem “Cert:\LocalMachine\My” | where {$_.Subject -like ‘sdnsite1.contoso.com’}
    $base64cert1 = [System.Convert]::ToBase64String($cert1.RawData)
    $cert2 = Get-ChildItem “Cert:\LocalMachine\My” | where {$_.Subject -like ‘sdnsite2.contoso.com’}
    $base64cert2 = [System.Convert]::ToBase64String($cert2.RawData)

    $prop = new-object Microsoft.Windows.NetworkController.NetworkControllerMultisiteProperties
    $prop.certificateSubjectName = "sdnsite1.contoso.com"
    $prop.Sites = new-object Microsoft.Windows.NetworkController.NetworkControllerSite
    $prop.Sites[0].ResourceId = "site2"
    $prop.Sites[0].Properties = new-object Microsoft.Windows.NetworkController.NetworkControllerSiteProperties
    $prop.Sites[0].Properties.RestIPAddress = "sdnsite2.contoso.com"
    $prop.Sites[0].Properties.CertificateSubjectName = "sdnsite2.contoso.com"
    $prop.Sites[0].Properties.EncodedCertificate = $base64cert2

    Set-NetworkControllerMultisiteConfiguration -ConnectionUri "https://sdnsite1.contoso.com" -Properties $prop -Force 

    $prop = new-object Microsoft.Windows.NetworkController.NetworkControllerMultisiteProperties
    $prop.certificateSubjectName = "sdnsite2.contoso.com"
    $prop.Sites = new-object Microsoft.Windows.NetworkController.NetworkControllerSite
    $prop.Sites[0].ResourceId = "site1"
    $prop.Sites[0].Properties = new-object Microsoft.Windows.NetworkController.NetworkControllerSiteProperties
    $prop.Sites[0].Properties.RestIPAddress = "sdnsite1.contoso.com"
    $prop.Sites[0].Properties.CertificateSubjectName = "sdnsite1.contoso.com"
    $prop.Sites[0].Properties.EncodedCertificate = $base64cert1

    Set-NetworkControllerMultisiteConfiguration -ConnectionUri "https://sdnsite2.contoso.com" -Properties $prop -Force

    ```

1. Run the following command from your Network Controller VM to confirm the setup.

    ```powershell
    Get-NetworkControllerMultisiteConfiguration -ConnectionUri "https://site1.com"  
    ```

    After you run this command, you should see your secondary site as confirmation.

Here's a sample output:

```output
"Properties": {
		"SecurityGroup": null,
		"CertificateSubjectName": "<CertificateSubjectName>",
		"ProvisioningState": "Succeeded",
		"Sites": [
				{
					"Etag":	"W/\"bel6615f-69bf-491d-a786-a3f478e4159d\"",
					"ResourceRef": "/multisite/configuration/networkControllerSite/remoteSite",
					"Instanceld":	"<InstanceID>",
					"ResourceMetadata":	{
								"Client": null,
								"Tenantld": null,
								"Groupld": null,
								"ResourceName":
								"OriginalHref": null
								},
					"Resourceld": "remoteSite",
					"Properties": {
							"IsPrimary": true,
							"RestlPAddress": "ncsdn0304.cfdev.nttest.microsoft.com",
							"State": "Connected",
							"Deploymentld": "<DeploymentID>",
							"ApiVersion":	"V6",
							"ProvisioningState": "Succeeded",
							"CertificateSubjectName": "<CertificateSubjectName>",
							"EncodedCertificate": "<EncodedCertificate>",
					"ConfigurationState":	{
								"LastUpdatedTime":	"\/Date(1693609205379)\/",
								"Detailedlnfo": null,
								"Status": "InProgress",
								"FailedResources":
								[
								]
								"ConflictingResources":
											[
											]
								}
					}
				}
			]
		}

```

---

## Next steps

Feel free to follow the links attached to our powershell documentation
and other Network controller related commandlets
