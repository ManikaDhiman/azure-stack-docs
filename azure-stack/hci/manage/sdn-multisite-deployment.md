---
title: Deploy SDN Multisite (preview)
description: Learn how to deploy a multisite SDN solution in Azure Stack HCI (preview)
author: alkohli
ms.author: alkohli
ms.topic: how-to
ms.date: 11/17/2023
---

# Deploy a multisite SDN solution in Azure Stack HCI

This article describes how to deploy the Software defined networking (SDN) Multisite solution in Azure Stack HCI.

SDN Multisite enables you to have Layer 2 and Layer 3 connectivity for virtualized workloads across multiple physical locations.

## Benefits

Here are the benefits of using SDN Multisite:

- **Seamless workload migration.** Seamlessly migrate workloads across locations without having to reconfigure IP addresses or pre-existing Network Security Groups (NSGs).
- **Automatic reachability to new VMs.** Get automatic reachability to newly created virtual machines (VMs) and automatic manageability to any of their associated NSGs across your physical locations. manageability to any of their associated NSGs across your physical locations. This document will cover how to deploy this feature through various management. If you'd
like an in-depth description of how this feature will work with our other services such as Software Load Balancer or Gateways, then please refer to our technical deep-dive document linked in the Notes bubble below.

## Capabilities and limitations

The following table provides the capabilities and limitations of the SDN Multisite feature in this release:

| Capabilities | Limitations |
|--|--|
| Software Load Balancers | Limited to 2 physical locations |
| Gateways | Software Load Balancers |
| Native Layer 2 and Layer 3 connectivity | Gateways |
| Unified policy management |  |


*\*For an in depth discussion on our limitations and capabilities,
please refer to the our technical deep dive article (LINK).*

## Prerequisites

Before you begin, please ensure the following prerequisites are met:

- There must be underlying [physical network connectivity](../concepts/plan-software-defined-networking-infrastructure.md#physical-and-logical-network-configuration) between two sites. Moreover, the Provider Network name must be the same on both sites.

- TCP port 49001 must be permitted by firewalls for cross-cluster communication.

- SDN must be installed on both sites separately, using SDN Express scripts or using Windows Admin Center. Hence, SDN infrastructure like Network Controller VMs, SLB MUX VMs and SDN Gateway VMs are unique to each site.

- At least one of the two sites must not have any virtual networks and/or Network Security Groups (NSGs) and/or user defined routes configured.

- The SDN MAC pool must not overlap between the two sites.

- The IP pools for the logical networks (HNV PA, Public VIP, Private VIP, GRE VIP, L3) must not overlap between the two sites.

## Deploy SDN Multisite

You can deploy SDN multisite via Windows Admin Center and PowerShell.

# [Windows Admin Center](#tab/windows-admin-center)

![A screenshot of a computer Description automatically
generated](./media/media/image2.png){width="6.495138888888889in"
height="3.9166666666666665in"}

1. To set up multisite through Windows Admin Center, first click on the Network Controller tab in the extension column in Tools.

1. From there, click on New to add a secondary site.

1. Once you've clicked on New, another window will pop up asking for the following:

    a.  Name -- The name of this pairing

    b.  Network Controller REST Uri -- REST URI of your secondary
        location

    c.  Cluster Name for New site -- Cluster name of your secondary
        location

    d.  Network Controller VM name for new site - NC VM name of your
        secondary location

    e.  Network Controller VM name for -- NC VM for your primary
        location

1. Once you've inputted all fields, simply click on Submit and you're all done

# [PowerShell](#tab/powershell)

1. To enable Multisite, peering must be initiated from both sites using the `Set-NetworkControllerMultisiteConfiguration` cmdlet. The full details behind the commandlet are as follows:

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

1. To confirm set-up, you can run the following command your NC VM machine to confirm. Once you've ran this command, you should see your secondary site as confirmation.

    ```powershell
    Get-NetworkControllerMultisiteConfiguration -ConnectionUri "https://site1.com"  
    ```

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
