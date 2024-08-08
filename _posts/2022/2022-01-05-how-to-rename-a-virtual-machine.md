---
title: NICOLTIP033 How to rename a virtual machine on Azure
date: 2022-01-05 10:00
tags: [Azure, Virtual Machine, IaaS, Powershell]

excerpt: "Do you remember that it is not possibile to rename a VM on Azure?"
header:
  overlay_image: https://live.staticflickr.com/65535/51811446245_5336ceb476_b.jpg
  overlay_filter: rgba(32, 255, 32, 0.5)
  caption: "Photo credit: [**nicola since 1972**](https://www.flickr.com/photos/15216811@N06/51811446245)"
---

It is not possible to rename a VM on azure. the only viable solution is to delete and re-create the machine.
This is the script I use to execute this task.

```powershell
#Set variables
$resourceGroup="my-resource-group"

$oldvmName="old-machine-name"
$newvmName="new-machine-name"

#Get the details of the VM
$originalVM = Get-AzVM -ResourceGroupName $resourceGroup -Name $oldvmName

#Remove the original VM
Remove-AzVM -ResourceGroupName $resourceGroup -Name $oldvmName

#Create the basic configuration for the replacement VM
$newVM = New-AzVMConfig -VMName $newvmName -VMSize $originalVM.HardwareProfile.VmSize
Set-AzVMOSDisk -VM $newVM -CreateOption Attach -ManagedDiskId $originalVM.StorageProfile.OsDisk.ManagedDisk.Id -Name $originalVM.StorageProfile.OsDisk.Name -Windows

#Add Data Disks
foreach ($disk in $originalVM.StorageProfile.DataDisks) {
Add-AzVMDataDisk-VM $newVM `
-Name$disk.Name `
-ManagedDiskId $disk.ManagedDisk.Id `
-Caching$disk.Caching `
-Lun$disk.Lun `
-DiskSizeInGB $disk.DiskSizeGB `
-CreateOption Attach
    }

#Add NICs
foreach ($nic in $originalVM.NetworkProfile.NetworkInterfaces) {
Add-AzVMNetworkInterface `
-VM $newVM `
-Id $nic.Id
    }
                
                
#Recreate the VM
New-AzVM -ResourceGroupName $resourceGroup -Location $originalVM.Location -VM $newVM-DisableBginfoExtension
```
