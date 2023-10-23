# From Setup to Start-FleetSweep

This is the traditional path of setting up VMFleet and running it using your desired DiskSpd parameters/flags.

 

## Prerequisites

Before we begin setting up VMFleet, there are a few prerequisites that you should have ready.

Ensure that you have 1 CSV per node

Within Storage Spaces Direct, CPU usage is based on the host. Therefore, it is recommended that you split the storage load by creating as many CSVs as there are host nodes. We can go ahead and create a CSV per node in the cluster.

You may run the following:
 

```
Get-ClusterNode |% {
    New-Volume -StoragePoolFriendlyName SU1_Pool* -FriendlyName $_ -FileSystem CSVFS_ReFS -Size <DESIRED SIZE>
}

``` 

Ensure that you create a “collect” volume.

You may run the following:
 
```

New-Volume -StoragePoolFriendlyName SU1_Pool* -FriendlyName collect -FileSystem CSVFS_ReFS -Size 200GB

``` 

If you have ran VMFleet in the past, please ensure that prior VMFleet directories are completely removed from existing volumes.

Retrieve or install a Server Core VHDX file. If you do not have one handy, we can create a new one by following the below instructions.

Download WS2019 Server Core ISO from the public website.

Open Hyper-V Manager.

Click “New”, then “Virtual Machine”.

Navigate through the prompts and pick a location to store your VM.

Once your VM is created, boot up the VM and follow the instructions. This is where you will decide your VM password, or what we will later call, “adminpass”.

This is important as we will use this later, so make sure you write this down.

Log out of the VM, and navigate to where you stored your VM. You should find a “Virtual Hard Disks” folder. Inside, you should find your new Server Core VHDX file.

Rename it to “Base1.vhdx”.

Copy or move the file to the cluster environment that you want to run VMFleet in.

Done!

### Deployment

1. First, we need to install the new PowerShell Module from the PowerShell Gallery and then load it into our terminal. We also need to disable cache as it is not supported currently for ArcVMs. Run the following:
 
```
Install-Module -Name “VMFleet”
Import-Module VMFleet
Set-ClusterStorageSpacesDirect -CacheState Disabled; 
 
```


2. Sanity Check:
    Run "Get-Module VMFleet" to confirm the module exists.

    Run "Get-Command -Module VMFleet" to obtain a list of functions included in the module.

    We will now set up the directory structure within the “Collect” CSV created earlier. Run "Install-Fleet"

    This creates the necessary VMFleet directories which include:

       * Collect/control

            Contains the scripts that the Virtual Machines continuously monitor.

       * Control.ps1: the control script the VMs use to implement the control loop (what used to be called “master.ps1”).

       * Run.ps1: The VMs continuously look for the most recent version of run.ps1 and runs the newly updated script (parameters).

       * Collect/flag

            Location where the control script drops the “go”, “pause”, and “done” flag files. Users should not need to look at these files.

       * Collect/result

            Location of the output files from the VMFleet test run.

       * Collect/tools

            DiskSpd will be preinstalled in this folder.

3. You need to create a new or use existing Resource group under the same subscription as the Resource bridge VM is under.

4. By default, VMs with 2GB static memory and 1 processor count will be created.

    Note: Please move the VHDX file into this collect folder. CSV Cache is also turned off by default.

    We will now create our “fleet” of VMs by running:
    
```
New-Fleet -basevhd <PATH TO VHDX> -vms [ENTER_NUM_VMS] -adminpass [ENTER_ADMINPASS] -connectuser [ENTER_NODE_USER] -connectpass [ENTER_NODE_PASS] -createArcVMs -resourceGroupForArcVM [ENTER_RESOURCE_GROUP] -azureRegistrationUser [ENTER_AZURE_REGUSER] -azureRegistrationPassword [ENTER_AZURE_REGPASS]
``` 

    -"adminpass" is the administrator password for the Server Core Image. This is the password you set on your Virtual Machine earlier.

    -"connectuser" is a domain account with access to the cluster.

    -"connectpass " is the password for the above domain account.

    -"vms" is the number of VM's to create per node.

    -"createArcVMs" is the flag to turn on Arc enabled VM creation.
    
    -"resourceGroupForArcVM" is the resource group where ARC VMs will be deployed.

    -"azureRegistrationUser" and "azureRegistrationPassword" are the azure account credentials.

5. If this parameter is not provided, the default is a 1:1 subscription ratio where the Number of VMs = Number of physical cores.

[Optional] You can consider modifying the VM hardware configuration. Run
 
```
Set-Fleet -ProcessorCount 1 -MemoryStartupBytes 2048 -MemoryMaximumBytes 2048 -MemoryMinimumBytes 2048
 ```

Note:

If you specify “MemoryMaximumBytes”, you must specify “MemoryMinimumBytes”, which implies that your VMs will have dynamic memory.

If you omit “MemoryMaximumBytes” or “MemoryMinimumBytes”, it implies that your VMs will have static memory.

If MemoryStartupBytes = MemoryMinimumBytes = MemoryMaximumBytes, that also denotes static memory.

“MemoryStartupBytes” is a mandatory parameter.


### Start Running VMFleet!

6. Open 2 PowerShell terminals. In the first one, run Watch-Cluster and in the second one, run Start-Fleet. This second function will turn on all the VMs in a “paused” state.

7. At this point you can run Start-FleetSweep [ENTER_PARAMETERS] or take this time to explore and run any of the other functions!

Here is a sample sweep command to help you get started: "Start-FleetSweep -b 4 -t 8 -o 8 -w 0 -d 300 -p r"

8. Done!

### Aftermath

Once you are done running VMFleet you can run Stop-Fleet to shut down all the virtual machines or run Remove-Fleet to completely delete all the virtual machines on your environment.

# From Setup to Measure-FleetCoreWorkload

This is a new workflow for setting up VMFleet and the predefined profile workloads (General, Peak, VDI, SQL).

 

## Prerequisites

Before we begin setting up VMFleet, there are a few prerequisites that you should have ready.

Ensure that you have 1 CSV per node

Within Storage Spaces Direct, CPU usage is based on the host. Therefore, it is recommended that you split the storage load by creating as many CSVs as there are host nodes. We can go ahead and create a CSV per node in the cluster.

In order to be precise about the CSV size, please use our new VMFleet command: "Get-FleetVolumeEstimate"

This will output a prescribed CSV size based on different resiliency types. We recommend you select a 2-way mirrored value or 3-way mirrored value depending on your node count.

1. You may run the following: (use the CSV size from Get-FleetVolumeEstimate)
 
```
Get-ClusterNode |% {
    New-Volume -StoragePoolFriendlyName SU1_Pool* -FriendlyName $_ -FileSystem CSVFS_ReFS -Size <DESIRED SIZE>
}
```

    Ensure that you create a “collect” volume.

2. You may run the following:
 
```
New-Volume -StoragePoolFriendlyName SU1_Pool* -FriendlyName collect -FileSystem CSVFS_ReFS -Size 200GB
``` 

3. If you have ran VMFleet in the past, please ensure that prior VMFleet directories are completely removed from existing volumes.

    Retrieve or install a Server Core VHDX file. If you do not have one handy, we can create a new one by following the below instructions.

    Download WS2019 Server Core ISO from the public website.

    Open Hyper-V Manager.

    Click “New”, then “Virtual Machine”.

    Navigate through the prompts and pick a location to store your VM.

    Once your VM is created, boot up the VM and follow the instructions. This is where you will decide your VM password, or what we will later call, “adminpass”.

    This is important as we will use this later, so make sure you write this down.

    Log out of the VM, and navigate to where you stored your VM. You should find a “Virtual Hard Disks” folder. Inside, you should find your new Server Core VHDX file.
    Rename it to “Base1.vhdx”.

    Copy or move the file to the cluster environment that you want to run VMFleet in.

4. Done!

## Deployment
5. Let’s begin deploying VMFleet. First, we need to install the new PowerShell Module from the PowerShell Gallery and then load it into the terminal. Run the following:
 
```
Install-Module -Name “VMFleet”
Import-Module VMFleet
Set-ClusterStorageSpacesDirect -CacheState Disabled; 
```

 

## Sanity Check:

6. Run "Get-Module VMFleet" to confirm the module exists.

7. Run "Get-Command -Module VMFleet" to obtain a list of commands included in the module.

8. We will now set up the directory structure within the “Collect” CSV that we created earlier. Run "Install-Fleet"

9. We will now create our “fleet” of VMs by running:

 
```
New-Fleet -basevhd <PATH TO VHDX> -adminpass [ENTER_ADMINPASS] -connectuser [ENTER_NODE_USER] -connectpass [ENTER_NODE_PASS]
``` 

    -"adminpass" is the administrator password for the Server Core Image. This is the password you set on your Virtual Machine earlier.
    -"connectuser" is a domain account with access to the cluster.
    -"connectpass " is the password for the above domain account.
    -"vms" is the number of VM's to create per node.
    -"createArcVMs" is the flag to turn on Arc enabled VM creation.
    -"resourceGroupForArcVM" is the resource group where ARC VMs will be deployed.
    -"azureRegistrationUser" and "azureRegistrationPassword" are the azure account credentials.


10. Measure-FleetCoreWorkload also collects diagnostic data (Get-SDDCDiagnosticInfo). Therefore, before running the command, we must also install the NuGet Package if you have not previously done so. In doing so, we also need to temporairly set the PSGallery as a trusted repository source (Note: This will temporarily relax the security boundary). 

 
```
$repo = Get-PSRepository -Name PSGallery
if ($null -eq $repo) { Write-Host "The PSGallery is not configured on this system, please address this before continuing" }
else {
    if ($repo.InstallationPolicy -ne 'Trusted') {
        Write-Host "Setting the PSGallery repository to Trusted, original InstallationPolicy: $($repo.InstallationPolicy)"
        Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
    } 

    ### Installing the pre-requisite modules

    Install-PackageProvider NuGet -Force
    Install-Module -Name PrivateCloud.DiagnosticInfo -Force
    Install-Module -Name MSFT.Network.Diag -Force

    if ($repo.InstallationPolicy -ne 'Trusted') {
        Write-Host "Resetting the PSGallery repository to $($repo.InstallationPolicy)"
        Set-PSRepository -Name PSGallery -InstallationPolicy $repo.InstallationPolicy
    } 
}  
``` 

## Run Measure-FleetCoreWorkload!
11. We can now run Measure-FleetCoreWorkload. Running the command below will automatically run all 4 workloads (General, Peak, VDI, SQL) and place the individual outputs in the result directory. IMPORTANT: If you plan on running another test, please clear the result directory.

 
```
Measure-FleetCoreWorkload
``` 

12. Congratulations! You’re done! All you need to do is wait for the test to complete.

Note: If you ever run into an error and need to rerun Measure-FleetCoreWorkload, don’t be afraid to do so! It is smart enough to pick up from where it last stopped and continue the test without starting from scratch.


## Sample Script using CI pipeline VHDX file

1. Run this powershell script on one of the cluster nodes -

```

Get-ClusterNode |% {
    New-Volume -StoragePoolFriendlyName SU1_Pool* -FriendlyName $_ -FileSystem CSVFS_ReFS -Size 100GB
}
New-Volume -StoragePoolFriendlyName SU1_Pool* -FriendlyName collect -FileSystem CSVFS_ReFS -Size 200GB
Install-Module -Name “VMFleet”
Import-Module VMFleet -Force
Install-Fleet
$repo = Get-PSRepository -Name PSGallery
if ($null -eq $repo) { Write-Host "The PSGallery is not configured on this system, please address this before continuing" }
else {
    if ($repo.InstallationPolicy -ne 'Trusted') {
        Write-Host "Setting the PSGallery repository to Trusted, original InstallationPolicy: $($repo.InstallationPolicy)"
        Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
    } 

    ### Installing the pre-requisite modules

    Install-PackageProvider NuGet -Force
    Install-Module -Name PrivateCloud.DiagnosticInfo -Force
    Install-Module -Name MSFT.Network.Diag -Force

    if ($repo.InstallationPolicy -ne 'Trusted') {
        Write-Host "Resetting the PSGallery repository to $($repo.InstallationPolicy)"
   clear     Set-PSRepository -Name PSGallery -InstallationPolicy $repo.InstallationPolicy
    } 
} 
 Set-ClusterStorageSpacesDirect -CacheState Disabled; 
 $domainAdminPass=*Enter password*
 $domainAdmin = *Enter admin*
 $vmAdminPass=*Enter vm password*
 $azuser = *Enter azure user*
 $azpasswrd=*Enter azure password*
 $baseImagePath = "C:\ClusterStorage\Collect\Base1.vhdx";
 Switch-ASLocalWDACPolicy -Mode Audit;

```

2. Copy Base1.vhdx into the 'collect' folder from CI Test share.

3. Create Resource group or provide an existing resource group name in the same subscription as arc RB VM.

4. Run below command to create new fleet.

```
 New-Fleet -BaseVhd $baseImagePath -AdminPass $vmAdminPass -AzureRegistrationUser $azuser -AzureRegistrationPassword $azpasswrd -ConnectUser $domainAdmin -ConnectPass $domainAdminPass -VMs 30 -CreateArcVMs -ResourceGroupForArcVM VMFleetArcVMRG14;
```

5. Run below command to deploy default VMFleet workload.

```
Measure-FleetCoreWorkload

```
