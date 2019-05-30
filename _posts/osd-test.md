---
layout: post
published: false
title: You down with OSD?
date: '2018-06-05'
subtitle: Yeah P.X.E.
---
######  Updated: 5/30/2019
## Operating System Deployment over PXE boot
Operating system deployment is a core functionality of SCCM.  While there are other ways to deploy Windows 10, all the tools we need to configure the OS are already in SCCM.  Being able to bring it all together makes OSD in SCCM flexible, powerful, and simple.

In this post we'll do a minimalistic configuration of OSD over PXE.  This means we can boot a workstation directly into the task sequence over the network.  Eventually, after we cover more SCCM fundamentals, we will revisit this idea in a series more advanced posts to really highlight the potential of OSD.

[Windows and Office Deployment Lab Kit](https://www.microsoft.com/en-us/evalcenter/evaluate-lab-kit) does a couple convenient configurations if we're using the Current Branch lab we configured in the [previous post](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/).  If we're using the [tech preview lab](https://doug.seiler.us/2019-05-27-tech-preview-lab-kit/) we will need to do some additional set up.

### The Requirements
1. A Windows 10 Enterprise ISO.

2. The latest [Windows Assessment and Deployment Kit](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) and the Windows PE add-on for the ADK.

### The Overview
For a no frills (non-MDT) OSD setup, I'm quite partial to Ricky Gao's [step-by-step guide](https://www.jqit.com.au/blog/step-by-step-configuring-and-troubleshooting-sccm-2012-r2-osd-deployment/).  It provides the basic outline of our OSD configuration.
1. [**Configure Active Directory and the ADK**](https://doug.seiler.us/2018-06-05-you-down-with-osd#active-directory-and-windows-adk)

2. [**Enable PXE**](https://doug.seiler.us/2018-06-05-you-down-with-osd#pxe)

3. [**Create sources**](https://doug.seiler.us/2018-06-05-you-down-with-osd#sources)

4. [**Create the Task Sequence**](https://doug.seiler.us/2018-06-05-you-down-with-osd#the-task-sequence)

5. [**Perform OSD over PXE**](https://doug.seiler.us/2018-06-05-you-down-with-osd#imaging)


## Active Directory and Windows ADK
### Active Directory Users and Computers
We need an OU for our workstations to go.  While we could use the builtin Computers OU, that doesn't leave us any wiggle room later for things like applying Group Polices.
#### Create the domain join account
###### You may now skip creating the domain join account, it will be created with the Powershell script
1. Log into the domain controller, HYD-DC1.

2. Launch _Active Directory Users and Computers_.

3. Right click the _Users_ OU, select _New_ -> _User_.

	![new_user.PNG](/img/300/new_user.PNG)
4. Leave _Initials_, _First name_, and _Last name_ blank.  For _User logon name_ and _Full name_, type **CM_DomainJoin**.  Click _Next_.

	![CM_DomainJoin](/img/300/cm_domainjoin_userlogonname.PNG)
5. Set the password to **P@ssw0rd**.  Uncheck _User must change password at next logon_ and check _Password never expires_.  Click _Next_, then click _Finish_.

	![cm_domainjoin_password.PNG](/img/300/cm_domainjoin_password.PNG)
    
#### Ceate the Workstations OU and set permissions
###### You may now skip creating the OU, it will be created with the Powershell script
1. In the _Active Directory Users and Computers_ tool, right-click on the _CORP_ OU. Select _New_ -> _Organizational Unit_.

	![new_ou.PNG](/img/300/new_ou.PNG)
2. Name the container **Workstations** and click OK.
	
    ![create_workstations_OU.PNG](/img/300/create_workstations_OU.PNG)
3. On the HYD-DC1, launch _Windows Powershell ISE_ as administrator.  Johan and Mikael have [created a script to set permissions](https://deploymentresearch.com/Research/Post/353/PowerShell-Script-to-set-permissions-in-Active-Directory-for-OSD) on the domain join account. This is so it can add computers to the _Workstations_ OU during OSD.

4. Below is a [modified version of Johan and Mikael's script](https://github.com/dugmo/DRFiles/blob/master/Scripts/Set-OUPermissions.ps1) to suit our lab.  It will now create the account and OU if they haven't been manually created already.  Paste the code into the top script pane, and hit _F5_ to run.

<script src="https://gist.github.com/dugmo/96d3efefd65a205e85694e2a2781c3b1.js"></script>

{% gist 96d3efefd65a205e85694e2a2781c3b1 %}

5. When it prompts for _Account_, type **CM_DomainJoin**.  Hit enter.

6. When it prompts for _target OU_, type **Workstations**. Hit enter.

	![domainjoin_permissions_script.PNG](/img/300/domainjoin_permissions_script.PNG)
    
### The Windows Assessment and Deployment Kit
We ended the [SCCM lab setup](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/) by updating to the latest version of Current Branch.  So we'll need to install the latest version of the Windows ADK so we're fully compatible deploying the current version of Windows 10.
1. Log into the SCCM server, HYD-CM1.

2. Open the _Programs and Features_ control panel, and uninstall the existing Windows Assessment and Deployment Kit - Windows 10.

	![uninstall_adk.PNG](/img/300/uninstall_adk.PNG)
3. Download the [Windows ADK](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) and run it.  Jonathan Lefebvre of the System Center Dudes has a [great guide on setting up the ADK](https://www.systemcenterdudes.com/how-to-update-windows-adk-on-a-sccm-server/) for SCCM.

4. Accept the defaults until we reach the _Select the features you want to install_ section.  Uncheck everything except _Deployment Tools_, _Windows Preinstallation Environment (Windows PE)_, and _User State Migration Tool (USMT)_.  Click _Install_.

	![windows_adk_features.PNG](/img/300/windows_adk_features.PNG)
5. Once the install is complete, click _Close_.  Reboot the SCCM server.


## PXE
### The Boundary
Boundaries and Boundary groups are how we logically define which sites (servers) workstations use.  For the lab, we only need to configure one boundary and boundary group.
1. Log into the SCCM server, HYD-CM1.

2. In the SCCM console, expand the _Administration_ -> _Hierarchy Configuration_ node.

3. Right-click _Boundaries_ and select _Create Boundary_.

4. **General tab** - Set description to **PXE**.  Change _Type_ to **IP address range**.  We will use the DHCP range from the domain controller.  Set _Starting IP address_ to **10.0.0.100** and set _Ending IP address_ to **10.0.0.200**.

	![create_boundary.png](/img/300/create_boundary.png)
    
	For reference, here is what the DHCP address pool looks like:
    
    ![dhcp_scope.png](/img/300/dhcp_scope.png)
5. **Boundary Groups tab** - Select the _Boundary Groups_ tab. Click _Add_.  Check _Corp Boundary Group_ and click _OK_.  Click _OK_ again to create the boundary.

	![create_boundary_group.png](/img/300/create_boundary_group.png)
6. The Microsoft lab already has the boundary group created and configured, but we'll just verify the settings.  Click on _Boundary Groups_, right-click _Corp Boundary Group_ and select _Properties_.  Select the _References_ tab and confirm that _User this boundary group for site assignment_ is checked.

	![confirm_boundary_group.PNG](/img/300/confirm_boundary_group.PNG)

### The Distribution Point
In SCCM, distribution points are where workstations get their content.  This is true for OSD as well, and we need our DPs to function as PXE servers.
1. In the SCCM console, navigate to the _Administration_ -> _Distribution Points_ node.

2. Right-click _CM1.CORP.CONTOSO.COM_ and select _Properties_.

3. **General tab** - Leave _HTTP_ selected and check _Allow clients to connect anonymously_.

	![dp_general_tab.png](/img/300/dp_general_tab.png)
4. **PXE tab** - To automatically install WDS and configure PXE, check _Enable PXE support for clients_.  Additionally check _Allow this distribution point to respond to incoming PXE requests_ **AND** _Enable unknown computer support_.  Uncheck _Require a password when computers use PXE_ and click _OK_.

	![dp_pxe_tab.png](/img/300/dp_pxe_tab.png)


## Sources
### The package source share
We'll use [Odd-Magne's script](https://sccmguru.wordpress.com/2012/10/25/configuration-manager-folder-structure/ ) to create the folder structure, share it, and grant permissions to our network access account.
1. On the SCCM server, launch Windows Powershell ISE as administrator.

2. In the top script pane, paste the following code.  The script is customized to our folder path and network access account name.  Press _F5_ to run.
```powershell
	#Set the Following Parameters
	$Source = 'C:\PackageSource'
	$ShareName = 'PackageSource'
	$NetworkAccount = 'CORP\CM_NetAcc'

	#Create Source Directory
	New-Item -ItemType Directory -Path "$Source"


	#Create Application Directory Structure
	New-Item -ItemType Directory -Path "$Source\Applications"
	New-Item -ItemType Directory -Path "$Source\Applications\Adobe"
	New-Item -ItemType Directory -Path "$Source\Applications\Apple"
	New-Item -ItemType Directory -Path "$Source\Applications\Citrix"
	New-Item -ItemType Directory -Path "$Source\Applications\Microsoft"

	#Create App-V Directory Structure
	New-Item -ItemType Directory -Path "$Source\App-V"
	New-Item -ItemType Directory -Path "$Source\App-V\Packages"
	New-Item -ItemType Directory -Path "$Source\App-V\Source"

	#Create Hardware Application Directory Structure
	New-Item -ItemType Directory -Path "$Source\HardwareApplications"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Dell"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Dell\Latitude E6510"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Dell\Latitude E6510\x86"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Dell\Latitude E6510\x64"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\HP"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\HP\EliteBook 8470p"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\HP\EliteBook 8470p\x86"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\HP\EliteBook 8470p\x64"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Lenovo"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Lenovo\X1 Carbon"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Lenovo\X1 Carbon\x86"
	New-Item -ItemType Directory -Path "$Source\HardwareApplications\Lenovo\X1 Carbon\x64"

	#Create Hotfix Directory Structure
	New-Item -ItemType Directory -Path "$Source\Hotfix"

	#Create Import Directory Structure
	New-Item -ItemType Directory -Path "$Source\Import"
	New-Item -ItemType Directory -Path "$Source\Import\Baselines"
	New-Item -ItemType Directory -Path "$Source\Import\MOFs"
	New-Item -ItemType Directory -Path "$Source\Import\Task Sequences"

	#Create Log Directory Structure
	New-Item -ItemType Directory -Path "$Source\Logs"
	New-Item -ItemType Directory -Path "$Source\Logs\MDTLogs"
	New-Item -ItemType Directory -Path "$Source\Logs\MDTLogsDL"

	#Create OSD Directory Structure
	New-Item -ItemType Directory -Path "$Source\OSD"
	New-Item -ItemType Directory -Path "$Source\OSD\BootImages"
	New-Item -ItemType Directory -Path "$Source\OSD\Branding"
	New-Item -ItemType Directory -Path "$Source\OSD\Branding\WinPE Background"
	New-Item -ItemType Directory -Path "$Source\OSD\Captures"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x86"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x86\Dell"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x86\HP"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x86\Lenovo"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x86\VMWare"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x64"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x64\Dell"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x64\HP"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x64\Lenovo"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 7 x64\VMWare"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 8 x64"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 8 x64\Dell"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 8 x64\HP"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 8 x64\Lenovo"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverPackages\Windows 8 x64\VMWare"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x86"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x86\Dell"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x86\HP"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x86\Lenovo"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x86\VMWare"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x64"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x64\Dell"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x64\HP"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x64\Lenovo"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 7 x64\VMWare"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 8 x64"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 8 x64\Dell"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 8 x64\HP"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 8 x64\Lenovo"
	New-Item -ItemType Directory -Path "$Source\OSD\DriverSources\Windows 8 x64\VMWare"
	New-Item -ItemType Directory -Path "$Source\OSD\MDTSettings"
	New-Item -ItemType Directory -Path "$Source\OSD\MDTToolkit"
	New-Item -ItemType Directory -Path "$Source\OSD\OSImages"
	New-Item -ItemType Directory -Path "$Source\OSD\OSInstall"
	New-Item -ItemType Directory -Path "$Source\OSD\Prestart"
	New-Item -ItemType Directory -Path "$Source\OSD\USMT"

	#Create Script Directory Structure
	New-Item -ItemType Directory -Path "$Source\Script"

	#Create State Capture Directory Structure
	New-Item -ItemType Directory -Path "$Source\StateCapture"

	#Create Tools Directory Structure
	New-Item -ItemType Directory -Path "$Source\Tools"
	New-Item -ItemType Directory -Path "$Source\Tools\PSTools"

	#Create Windows Update Directory Structure
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates"
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates\Endpoint Protection"
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates\Lync 2010"
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates\Office 2010"
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates\Silverlight"
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates\Visual Studio 2008"
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates\Windows 7"
	New-Item -ItemType Directory -Path "$Source\WindowsUpdates\Windows 8"

	#Create WSUS Directory
	New-Item -ItemType Directory -Path "$Source\WSUS"
	New-Item -ItemType Directory -Path "$Source\WSUS\SCCMDeploymentPackages"

	#Create the Share and Permissions
	New-SmbShare -Name "$ShareName” -Path “$Source” -CachingMode None -FullAccess Everyone
	New-SmbShare -Name "SCCMDeploymentPackages” -Path "$Source\WSUS\SCCMDeploymentPackages" -CachingMode None -FullAccess $NetworkAccount,"NETWORK SERVICE"

	#Set Security Permissions
	$Acl = Get-Acl "$Source\Logs"
	$Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("$NetworkAccount","FullControl", "ContainerInherit, ObjectInherit", "None", "Allow")
	$Acl.SetAccessRule($Ar)
	Set-Acl "$Source\Logs" $Acl

	$Acl = Get-Acl "$Source\OSD\Captures"
	$Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("$NetworkAccount","FullControl", "ContainerInherit, ObjectInherit", "None", "Allow")
	$Acl.SetAccessRule($Ar)
	Set-Acl "$Source\OSD\Captures" $Acl

	$Acl = Get-Acl "$Source\StateCapture"
	$Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("LOCALSERVICE","FullControl", "ContainerInherit, ObjectInherit", "None", "Allow")
	$Acl.SetAccessRule($Ar)
	Set-Acl "$Source\StateCapture" $Acl

	$Acl = Get-Acl "$Source\WSUS"
	$Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("$NetworkAccount","FullControl", "ContainerInherit, ObjectInherit", "None", "Allow")
	$Acl.SetAccessRule($Ar)
	Set-Acl "$Source\WSUS" $Acl

	$Acl = Get-Acl "$Source\WSUS"
	$Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("NETWORK SERVICE","FullControl", "ContainerInherit, ObjectInherit", "None", "Allow")
	$Acl.SetAccessRule($Ar)
	Set-Acl "$Source\WSUS" $Acl
```

### The Operating System Image
We need an operating system image as the basis of OSD.  There are several guides on creating a fully patched reference image, but we're just going to use the WIM file off of the ISO for now and revisit reference images in a later tutorial.
1. On the host system, mount the Windows 10 Enterprise (or Pro) ISO.  If your host system is Windows 10 you can just double click the ISO.

2. Navigate to the sources folder, right-click _install.wim_ and select _copy_.

	![install_wim.png](/img/300/install_wim.png)
3. On HYD-CM1 (the SCCM server) navigate to _C:\PackageSource\OSD\OSImages_

4. Paste the _install.wim_ to this folder.  Hyper-V will let you copy and paste files directly between host and VM.

	![install_wim_on_sccm.png](/img/300/install_wim_on_sccm.png)
5. From the SCCM console expand the _Software Library_ -> _Operating Systems_ node.

6. Right-click _Operating System Images_ and select _Add Operating System Image_.

7. **Data Source** - In the path field type **\\\cm1\PackageSource\OSD\OSImages\install.wim** and click _Next_.

8. **General** - Leave the default name, but type the Windows 10 build in the _Version_ field.  Click _Next_.

	![os_image_wizard.png](/img/300/os_image_wizard.png)
9. At the _Summary_ screen, click _Next_.  At the _Completion_ screen, click _Close_.

10. Right-click the OS image we just created and select _Distribute Content_.  At the _General_ screen click Next.

11. **Content Destination** - Click _Add_ -> _Distribution Point_, and check _CM1.CORP.CONTOSO.COM_.  Click _OK_, then click _Next_.

	![os_image_distribute_content.png](/img/300/os_image_distribute_content.png)
	![os_image_distribute_content2.png](/img/300/os_image_distribute_content2.png)
12. At the _Summary_ screen, click _Next_.  At the _Completion_ screen, click _Close_.

### The Boot Image
SCCM requires a WinPE image to initiate the task sequence.  This is a low level Windows OS that runs in the beginning of a task sequence when we PXE boot.
1. From the SCCM console, expand the _Software Library_ -> _Operating Systems_ node.

2. Click on _Boot Images_.  Right click _Boot image (x64)_ and select _Properties_.

3. **Customization tab** - Check _Enable command suppport (testing only)_.  This allows us to press F8 during the task sequence to bring up the command prompt, which is useful for troubleshooting.

	![boot_image_customization.png](/img/300/boot_image_customization.png)
    
4. **Data Source tab** - Confirm that _Deploy this boot image from the PXE-enabled distribution point_ is already checked.  Click _OK_.
	
	![boot_image_confirm_pxe.PNG](/img/300/boot_image_confirm_pxe.PNG)
5. We'll be prompted to update the distribution points.  Click _Yes_, and the _Update Distribution Points Wizard_ will launch.

	![boot_image_update_distribution_points.png](/img/300/boot_image_update_distribution_points.png)
5. At the _General_ screen, check _Reload this boot image with the current Windows PE version from the Windows ADK_.  Click _Next_.

	![boot_image_update_wizard.PNG](/img/300/boot_image_update_wizard.PNG)
6. At the _Summary_ screen click Next.  At the _Completion_ screen click _Close_.

7. We can now see _Boot image (x64)_ has a higher _OS Version_ but the _Version_ needs to be updated.

	![boot_image_upgraded.png](/img/300/boot_image_upgraded.png)
8. Right-click _Boot image (x64)_ and select _Properties_.  Change the _Version_ to match the _OS Version_.

	![boot_image_version.PNG](/img/300/boot_image_version.PNG)
#### Optionally Ricky Gao's guide creates a new _Configuration Manager Client_ package.  We will skip for our lab.


## The Task Sequence
### Creating the task sequence
We can now create our Windows 10 task sequence and deploy it.
1. From the SCCM console, expand the _Software Library_ -> _Operating Systems_ node.

2. Right-click _Task Sequences_ and select _Create Task Sequence_.

3. Choose _Install an existing image package_ and click _Next_.

4. Set the _Task sequence name_ to **Windows 10** and click _Browse_.  Select _Boot image (x64)_ and click _OK_.  Click _Next_

	![ts_boot_image.PNG](/img/300/ts_boot_image.PNG)
	![ts_information.png](/img/300/ts_information.png)
5. For the _Image package_ click _Browse_ and select the OS image we created earlier.  If the image contains multiple indexes, choose Windows 10 Enterprise.  Click _Next_.

	![ts_install_windows.png](/img/300/ts_install_windows.png)
6. Choose _Join a domain_.  For _Domain_ click _Browse_ and select _corp.contoso.com_.  Click _OK_.

	![ts_select_domain.png](/img/300/ts_select_domain.png)
7. For _Domain OU_ click _Browse_.  Expand _CORP_ and choose the _Workstations_ OU we created earlier.  Click _OK_.

	![select_workstations_OU.PNG](/img/300/select_workstations_OU.PNG)
8. For _Account_ click _Set_.  Set the _User name_ to **CORP\CM_DomainJoin** and enter **P@ssw0rd** as the _Password_.  Click _Verify_, and click _Test connection_.

	![verify_domainjoin_account.PNG](/img/300/verify_domainjoin_account.PNG)
9. Click _OK_, click _OK_ again, and click _Next_.

	![ts_configure_network.PNG](/img/300/ts_configure_network.PNG)
10. At the _Install the Configuration Manager client_ wizard, leave the default and click _Next_.

11. Since we're doing a bare metal install, we don't need USMT.  Uncheck _Capture user settings_, uncheck _Capture network settings_, and uncheck _Capture Microsoft Windows settings_.  Click _Next_.

12. Click _Next_ through the _Updates_, _Applications_, and _Summary_ screens.  We'll cover all of that in a later post which we'll eventually link from this one.  At the _Completion_ screen, click _Close_.

### Deploying the task sequence
We need to deploy our task sequence to the _All Unknown Computers_ collection so that we can image bare metal workstations.
1. Right-click the _Windows 10_ task sequence we just created and select _Deploy_.

2. For the _Collection_ click _Browse_.  Read the warning and click _OK_.  Select _All Unknown Computers_ and click _OK_.  Click _Next_.

	![ts_deploy_available_pxe.png](/img/300/ts_deploy_available_pxe.png)
3. Leave the purpose as _Available_.  Set _Make available to the following_ to _Only media and PXE_ and click _Next_.

	![ts_deploy_media_and_pxe.png](/img/300/ts_deploy_media_and_pxe.png)
4. 	Leave the rest of the wizard as default and click _Next_ until you reach the _Completion_ screen.  Click _Close_.

### Imaging
It's time to test all our hard work.  We'll create a brand new VM and image it over PXE.
1. On the host machine, open the Hyper-V console.  Right-click your host machine and select _New_ -> _Virtual Machine_.

	![imaging_new_virtual_machine.PNG](/img/300/imaging_new_virtual_machine.PNG)
2. If _New Virtual Machine_ wizard takes you to the _Before You Begin_ screen, click _Next_.  Name the VM **HYD-WS1** and click _Next_.

	![imaging_name.PNG](/img/300/imaging_name.PNG)
3. Select _Generation 2_.  This will make it UEFI based as opposed to BIOS, and allows us to do Secure Boot.  Click _Next_.

4. Set the _Startup memory_ to **2048** or higher.  Click _Next_.

	![imaging_memory.PNG](/img/300/imaging_memory.PNG)
5. Change the _Connection_ to our internal virtual switch _HYD-CorpNet_.  Click _Next_.

6. Set the _Location_ to the large and fast drive where your other VMs are located.  Change the _Size_ to **64**GB.  Click _Next_.

	![imaging_location.PNG](/img/300/imaging_location.PNG)
7. Set the installation option to _Install an operating system from a network-based installation server_.  Click _Next_.

8. At the _Summary_ screen click _Finish_.

9. In the Hyper-V console, right-click our new virtual machine _HYD-WS1_ and select _Settings_.  Click on _Security_, and check _Enable Trusted Platform Module_.  Click _OK_.

	![imaging_settings.PNG](/img/300/imaging_settings.PNG)
10. **HIT THE SWITCH** - Open _HYD-WS1_ and click _Start_.  Since we're using an _Available_ deployment, we will need to press ENTER to PXE boot.

	![imaging_pxe_boot.PNG](/img/300/imaging_pxe_boot.PNG)
11. The boot image will download and boot WinPE.  At the task sequence _Welcome_ screen, click _Next_.

	![imaging_ts_wizard.PNG](/img/300/imaging_ts_wizard.PNG)
12. Select the _Windows 10_ task sequence we created earlier and click _Next_.

	![imaging_ts_select.PNG](/img/300/imaging_ts_select.PNG)
13. Wait and watch as the task sequence performs each step.

	![imaging_apply_os.PNG](/img/300/imaging_apply_os.PNG)
14. That's it, we've successfully deployed a bare metal workstation over PXE with SCCM.  Log into HYD-WS1 and verify that it's on the domain, BitLocker is on, and the Configuration Manager control panel is there.  Make note of the computer name, open the SCCM console on _HYD-CM1_ and confirm the new workstation shows up under _Devices_.

#### Congratulations again!  Now, we have an SCCM environment we can populate with workstations to manipulate.
**Next up** - [Patching with SCCM and WSUS](https://doug.seiler.us/2018-07-10-SCCM-WSUS-Patching/)
