---
title: Troubleshoot networking issues and errors in Azure Kubernetes Service on Azure Stack HCI 
description: Get help to troubleshoot networking issues and errors in Azure Kubernetes Service on Azure Stack HCI.
author: mattbriggs
ms.topic: troubleshooting
ms.date: 12/13/2021
ms.author: mabrigg 
ms.lastreviewed: 1/14/2022
ms.reviewer: abha

---

# Fix known issues and errors when configuring a network

Use this topic to help you troubleshoot and resolve networking-related issues in AKS on Azure Stack HCI.

## Get-AksHCIClusterNetwork does not show the current allocation of IP addresses
Running the [Get-AksHciClusterNetwork](./reference/ps/get-akshciclusternetwork.md) command provides a list of all virtual network configurations. However, the command does not show the current allocation of the IP addresses. To find out what IP addresses are currently in use in a virtual network, use the steps below:

1. To get the group, run the following command:

   ```powershell
   Get-MocGroup -location MocLocation
   ```
2. To get the list of IP addresses that are currently in use, and the list of available or used virtual IP addresses, run the following command:

   ```powershell
   Get-MocNetworkInterface -Group <groupName> | ConvertTo-Json -depth 10
   ```

3. Use the following command to view the list of virtual IP addresses that are currently in use: 

   ```powershell
   Get-MocLoadBalancer -Group <groupName> | ConvertTo-Json -depth 10
   ```

## Cloud agent may fail to start successfully when using path names with spaces in them
When using [Set-AksHciConfig](./reference/ps/set-akshciconfig.md) to specify `-imageDir`, `-workingDir`, `-cloudConfigLocation`, or `-nodeConfigLocation` parameters with a path name that contains a space character, such as `D:\Cloud Share\AKS HCI`, the cloud agent cluster service will fail to start with the following (or similar) error message:

```
Failed to start the cloud agent generic cluster service in failover cluster. The cluster resource group os in the 'failed' state. Resources in 'failed' or 'pending' states: 'MOC Cloud Agent Service'
```

To work around this issue, use a path that does not include spaces, for example, `C:\CloudShare\AKS-HCI`.

## Network proxy server blocks HTTP requests
When applying the platform configuration, the network proxy server blocked HTTP requests originating from the user agent string **Google Chrome 65** because this string is an out-of-date user agent client. 

The user agent will be updated to **Google Chrome 91** in the next release.

## Set-AksHciConfig fails with WinRM errors, but shows WinRM is configured correctly
When running [Set-AksHciConfig](./reference/ps/./set-akshciconfig.md), you might encounter the following error:

```powershell
WinRM service is already running on this machine.
WinRM is already set up for remote management on this computer.
Powershell remoting to TK5-3WP08R0733 was not successful.
At C:\Program Files\WindowsPowerShell\Modules\Moc\0.2.23\Moc.psm1:2957 char:17
+ ...             throw "Powershell remoting to "+$env:computername+" was n ...
+                 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : OperationStopped: (Powershell remo...not successful.:String) [], RuntimeException
    + FullyQualifiedErrorId : Powershell remoting to TK5-3WP08R0733 was not successful.
```

Most of the time, this error occurs as a result of a change in the user's security token (due to a change in group membership), a password change, or an expired password. In most cases, the issue can be remediated by logging off from the computer and logging back in. If this still fails, you can file an issue at [GitHub AKS HCI issues](https://aka.ms/aks-hci/issues).

## The workload cluster is not found 

The workload cluster may not be found if the IP address pools of two AKS on Azure Stack HCI deployments are the same or overlap. If you deploy two AKS hosts and use the same `AksHciNetworkSetting` configuration for both, PowerShell and Windows Admin Center will potentially fail to find the workload cluster as the API server will be assigned the same IP address in both clusters resulting in a conflict.

The error message you receive will look similar to the example shown below.

```powershell
A workload cluster with the name 'clustergroup-management' was not found.
At C:\Program Files\WindowsPowerShell\Modules\Kva\0.2.23\Common.psm1:3083 char:9
+         throw $("A workload cluster with the name '$Name' was not fou ...
+         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : OperationStopped: (A workload clus... was not found.:String) [], RuntimeException
    + FullyQualifiedErrorId : A workload cluster with the name 'clustergroup-management' was not found.
```

> [!NOTE]
> Your cluster name will be different.

## Load balancer in Azure Kubernetes Service requires DHCP reservation
The load balancing solution in Azure Kubernetes Service on Azure Stack HCI uses DHCP to assign IP addresses to service endpoints. If the IP address changes for the service endpoint due to a service restart, DHCP lease expires due to a short expiration time. Therefore, the service becomes inaccessible because the IP address in the Kubernetes configuration is different from what is on the endpoint. This can lead to the Kubernetes cluster becoming unavailable. To get around this issue, use a MAC address pool for the load balanced service endpoints and reserve specific IP addresses for each MAC address in the pool.

## Creating virtual networks with a similar configuration cause overlap issues
When creating overlapping network objects using the `new-akshcinetworksetting` and `new-akshciclusternetwork` PowerShell cmdlets, issues can occur. For example, issues may happen in scenarios where two virtual network configurations are almost the same.

## Using Remote Desktop to connect to the management cluster produces a connection error
When using Remote Desktop (RDP) to connect to one of the nodes in an Azure Stack HCI cluster and then running [Get-AksHciCluster](./reference/ps/get-akshcicluster.md), an error appears and says the connection failed because the host failed to respond.

The reason for the connection failure is because some PowerShell commands that use `kubeconfig-mgmt` fail with an error similar to the following one:

```
Unable to connect to the server: d ial tcp 172.168.10.0:6443, where 172.168.10.0 is the IP of the control plane.
```

The _kube-vip_ pod can go down for two reasons:

* The memory pressure in the system can slow down `etcd`, which ends up affecting _kube-vip_.
* The _kube-apiserver_ is not available.

To help resolve this issue, try rebooting the machine. However, the issue of the memory pressure slowing down may return.

## When deploying AKS on Azure Stack HCI with a misconfigured network, deployment timed out at various points
When deploying AKS on Azure Stack HCI, the deployment may time out at different points of the process depending on where the misconfiguration occurred. You should review the error message to determine the cause and where it occurred.

For example, in the following error, the point at which the misconfiguration occurred is in `Get-DownloadSdkRelease -Name "mocstack-stable"`: 

```
$vnet = New-AksHciNetworkSettingSet-AksHciConfig -vnet $vnetInstall-AksHciVERBOSE: 
Initializing environmentVERBOSE: [AksHci] Importing ConfigurationVERBOSE: 
[AksHci] Importing Configuration Completedpowershell : 
GetRelease - error returned by API call: 
Post "https://msk8s.api.cdp.microsoft.com/api/v1.1/contents/default/namespaces/default/names/mocstack-stable/versions/0.9.7.0/files?action=generateDownloadInfo&ForegroundPriority=True": 
dial tcp 52.184.220.11:443: connectex: 
A connection attempt failed because the connected party did not properly
respond after a period of time, or established connection failed because
connected host has failed to respond.At line:1 char:1+ powershell -command
{ Get-DownloadSdkRelease -Name "mocstack-stable"}
```

This indicates that the physical Azure Stack HCI node can resolve the name of the download URL, `msk8s.api.cdp.microsoft.com`, but the node can't connect to the target server.

To resolve this issue, you need to determine where the breakdown occurred in the connection flow. Here are some steps to try to resolve the issue from the physical cluster node:

1. Ping the destination DNS name: ping `msk8s.api.cdp.microsoft.com`. 
2. If you get a response back and no time-out, then the basic network path is working. 
3. If the connection times out, then there could be a break in the data path. For more information, see [check proxy settings](./set-proxy-settings.md). Or, there could be a break in the return path, so you should check the firewall rules. 

## Next steps
- [Troubleshooting overview](troubleshoot-overview.md)
- [Installation issues and errors](known-issues-installation.md)
- [Storage known issues](known-issues-storage.md)