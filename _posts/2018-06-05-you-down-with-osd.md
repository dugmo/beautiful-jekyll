---
layout: post
published: true
title: You down with OSD?
date: '2018-06-05'
subtitle: Yeah P.X.E.
---
######  Updated: 6/1/2019
## Operating System Deployment over PXE boot
Operating system deployment is a core functionality of SCCM.  While there are other ways to deploy Windows 10, all the tools we need to configure the OS are already in SCCM.  Being able to bring it all together makes OSD in SCCM flexible, powerful, and simple.

In this post we'll do a minimalistic configuration of OSD over PXE.  This means we can boot a workstation directly into the task sequence over the network.  Eventually, after we cover more SCCM fundamentals, we will revisit this idea in a series more advanced posts to really highlight the potential of OSD.

[Windows and Office Deployment Lab Kit](https://www.microsoft.com/en-us/evalcenter/evaluate-lab-kit) does a couple convenient configurations if we're using the Current Branch lab we configured in the [previous post](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/).  If we're using the [tech preview lab](https://doug.seiler.us/2019-05-27-tech-preview-lab-kit/) we will need to do some additional set up.

### The Requirements
1. A Windows 10 Enterprise 64-bit ISO.

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

1. Log into the domain controller, _HYD-DC1_.
	
2. On the HYD-DC1, launch _Windows Powershell ISE_ as administrator.  Johan and Mikael have [created a script to set permissions](https://deploymentresearch.com/Research/Post/353/PowerShell-Script-to-set-permissions-in-Active-Directory-for-OSD) on the domain join account. This is so it can add computers to the _Workstations_ OU during OSD.

3. Below is a [modified version of Johan and Mikael's script](https://github.com/dugmo/DRFiles/blob/master/Scripts/Set-OUPermissions.ps1) to suit our lab.  It will now create the account and OU if they haven't been manually created already.  Paste the code into the top script pane, and hit _F5_ to run.

	<script src="https://gist.github.com/dugmo/96d3efefd65a205e85694e2a2781c3b1.js"></script>

4. When it prompts for _Account_, type **CM_DomainJoin**.  Hit enter.

5. When it prompts for _target OU_, type **Workstations**. Hit enter.

	![domainjoin_permissions_script.PNG](/img/300/domainjoin_permissions_script.PNG)
    
### The Windows Assessment and Deployment Kit
We ended the [SCCM lab setup](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/) by updating to the latest version of Current Branch (or the latest [tech preview](https://doug.seiler.us/2019-05-27-tech-preview-lab-kit/)).  So we'll need to install the latest version of the Windows ADK so we're fully compatible deploying the current version of Windows 10.
1. Log into the SCCM server, HYD-CM1.

2. Open the _Programs and Features_ control panel, and uninstall the existing _Windows Assessment and Deployment Kit - Windows 10_.

	![uninstall_adk.PNG](/img/300/uninstall_adk.PNG)

3. Uninstall the existing _Windows Assessment and Deployment Kit Windows Preinstallation Environment Add-ons - Windows 10_.  Reboot _HYD-CM1_.

	![uninstall_adk_winpe.PNG](/img/300/uninstall_adk_winpe.PNG)
4. On the host system, download the latest [Windows ADK](https://docs.microsoft.com/en-us/windows-hardware/get-started/adk-install) and Windows PE add-on for the ADK.  Jonathan Lefebvre of the System Center Dudes has a [great guide on setting up the ADK](https://www.systemcenterdudes.com/how-to-update-windows-adk-on-a-sccm-server/) for SCCM.  Copy _adksetup.exe_ and _adkwinpesetup.exe_ to _HYD-CM1_.

5. Run _adksetup.exe_.  Accept the defaults and click _Next_ until we reach the _Select the features you want to install_ section.  Uncheck everything except _Deployment Tools_ and _User State Migration Tool (USMT)_.  Click _Install_.  When it completes, click _Close_.

	![windows_adk_features.PNG](/img/300/windows_adk_features.PNG)
6. Run _adkwinpesetup.exe_.  Accept all defaults again at each screen and install.  When the install completes, click _Close_.

	![windows_adkwinpe_complete.PNG](/img/300/windows_adkwinpe_complete.PNG)
7. Once the install is complete, click _Close_.  Reboot the SCCM server.


## PXE
### The Boundary
Boundaries and Boundary groups are how we logically define which sites (servers) workstations use.  For the lab, we only need to configure one boundary and boundary group.
1. Log into the SCCM server, HYD-CM1.

2. In the SCCM console, expand the _Administration_ -> _Hierarchy Configuration_ node.

3. Right-click _Boundaries_ and select _Create Boundary_.

4. **General tab** - Set description to **PXE**.  Change _Type_ to **IP address range**.  We will use the DHCP range from the domain controller.  Set _Starting IP address_ to **10.0.0.100** and set _Ending IP address_ to **10.0.0.200**.  Click _OK_.

	![create_boundary.png](/img/300/create_boundary.png)
    
	For reference, here is what the DHCP address pool looks like:
    
    ![dhcp_scope2.PNG](/img/300/dhcp_scope2.PNG)
5. Expand the _Administration_ -> _Hierarchy Configuration_ node and select _Boundary Groups_.  The [current branch lab](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/) we set up has the boundary group configured already.  If we're using the [tech preview lab](https://doug.seiler.us/2019-05-27-tech-preview-lab-kit/), we need to create the boundary group manually.

6. If the _Corp Boundary Group_ doesn't exist yet, click _Create Boundary Group_.  Set the name and description to **Corp Boundary Group**.  If it does exist, right-click it and select _Properties_.

7. **General Tab** - Under _Boundaries..._ click _Add..._ and check our _PXE_ boundary.  Click _OK_.  Select the _References_ tab.

	![create_boundary_group.png](/img/300/create_boundary_group.png)
8. **References Tab** - If there are not _Site system servers_ listed, click _Add_ and check _CHQ_.  Click _OK_.  Make sure _Use this boundary group for site assignment_ is checked and click _OK_.

	![confirm_boundarygroup_references.png](/img/300/confirm_boundarygroup_references.png)

	![confirm_boundary_group2.png](/img/300/confirm_boundary_group2.png)


### The Distribution Point
In SCCM, distribution points are where workstations get their content.  This is true for OSD as well, and we need our DPs to function as PXE servers.
1. In the SCCM console, navigate to the _Administration_ -> _Distribution Points_ node.

2. Right-click _CM1.CORP.CONTOSO.COM_ and select _Properties_.

3. **General tab** - If we're using SCCM Current Branch, leave _HTTP_ selected and check _Allow clients to connect anonymously_.

	![dp_general_tab.png](/img/300/dp_general_tab.png)
4. **Communication tab** - If we're using the Tech Preview, the _Allow clients to connect anonymously_ is on the _Communication_ tab.  Check it.

	![dp_general_tab.png](/img/300/dp_general_tab.png)
5. **PXE tab** - To automatically install WDS and configure PXE, check _Enable PXE support for clients_ and click _Yes_ when the _Review Required Ports for PXE_ warning dialog pops up.  
	
	Additionally check _Allow this distribution point to respond to incoming PXE requests_ **AND** _Enable unknown computer support_, click _OK_ when the _Configuration Manager_ dialog box pops up.  
	
	Uncheck _Require a password when computers use PXE_.  Click _OK_.

	![dp_pxe_tab.png](/img/300/dp_pxe_tab.png)


## Sources
### The package source share
We'll use [Odd-Magne's script](https://sccmguru.wordpress.com/2012/10/25/configuration-manager-folder-structure/ ) to create the folder structure, share it, and grant permissions to our network access account.
1. On the SCCM server, launch Windows Powershell ISE as administrator.

2. In the top script pane, paste the following code.  The script is customized to our folder path and network access account name.  Press _F5_ to run.

	<script src="https://gist.github.com/dugmo/150e1015ed989286b5a4480a5e3ba87b.js"></script>

	![package_source_script.png](/img/300/package_source_script.png)


### The Operating System Image
We need an operating system image as the basis of OSD.  There are several guides on creating a fully patched reference image, but we're just going to use the WIM file off of the ISO for now and revisit reference images in a later tutorial.
1. On the host system, mount the Windows 10 Enterprise ISO.  If your host system is Windows 10 you can just double click the ISO.

2. Right-click the drive letter of the mounted ISO, and select _Properties_.

	![mounted_iso_properties.png](/img/300/mounted_iso_properties.png)
3. **Sharing tab** - Click _Advanced Sharing_.  Check _Share this folder_ and note the _Share name_.  Click _OK_.  Click _Close_.

	![mounted_iso_sharing.png](/img/300/mounted_iso_sharing.png)	
4. On _HYD-CM1_ (the SCCM server) navigate to **\\\\\<host system name>\\\<share name>\\sources**.  Right-click _install.wim_ and select _copy_.

	![install_wim.png](/img/300/install_wim.png)
5. Navigate to _C:\PackageSource\OSD\OSImages_ and paste the _install.wim_ to this folder.

	![install_wim_on_sccm.png](/img/300/install_wim_on_sccm.png)
6. From the SCCM console expand the _Software Library_ -> _Operating Systems_ node.

7. Right-click _Operating System Images_ and select _Add Operating System Image_.

8. **Data Source** - In the path field type **\\\cm1\PackageSource\OSD\OSImages\install.wim**.  Check _Extract a specific image index from the specified WIM file_ and select image index _3 - Windows 10 Enterprise_.
If we're using the Tech Preview, select _x64_ and click _Next_.

	![os_image_datasource.png](/img/300/os_image_datasource.png)
9. **General** - Set the name to **Windows 10 Enterprise**. Type the Windows 10 build in the _Version_ field.  Optionally we can add a comment.  Click _Next_.

	![os_image_wizard.png](/img/300/os_image_wizard.png)
10. At the _Summary_ screen, click _Next_.  At the _Completion_ screen, click _Close_.

11. Right-click the OS image we just created and select _Distribute Content_.  At the _General_ screen click Next.

12. **Content Destination** - Click _Add_ -> _Distribution Point_, and check _CM1.CORP.CONTOSO.COM_.  Click _OK_, then click _Next_.

	![os_image_distribute_content.png](/img/300/os_image_distribute_content.png)
	![os_image_distribute_content2.png](/img/300/os_image_distribute_content2.png)
13. At the _Summary_ screen, click _Next_.  At the _Completion_ screen, click _Close_.

### The Boot Image
SCCM requires a WinPE image to initiate the task sequence.  This is a low level Windows OS that runs in the beginning of a task sequence when we PXE boot.
1. From the SCCM console, expand the _Software Library_ -> _Operating Systems_ node.

2. Click on _Boot Images_.  If we're using the [Tech Preview lab](https://doug.seiler.us/2019-05-27-tech-preview-lab-kit/), right-click the boot image and select _Distribute Content_. Just like with the OS image, at the _Content Destination_ wizard click _Add_ and select _Distribution Point_.  Check CM1.CORP.CONTOSO.COM and click _OK_.  Click _Next_.  At the _Summary_ screen click _Next_. At the _Completion_ screen click _Close_.

	If we're using the [Current Branch lab](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/) the boot images are already distributed. Right-click _Boot image (x64)_ and select _Update Distribution Points_.  
	
	Check _Reload this boot image with the current Windows PE version from the Windows ADK_.  Click _Next_.  At the _Summary_ screen click _Next_.  At the _Completion_ screen click _Close_.

	![boot_image_update_wizard.PNG](/img/300/boot_image_update_wizard.PNG)
4. **General tab** - Right-click _Boot image (x64)_ and select _Properties_.  Set the _Version_ to the ADK _OS Version_.

	![boot_image_version.PNG](/img/300/boot_image_version.PNG)
5. **Customization tab** - Check _Enable command suppport (testing only)_.  This allows us to press F8 during the task sequence to bring up the command prompt, which is useful for troubleshooting.

	![boot_image_customization.png](/img/300/boot_image_customization.png)
6. **Data Source tab** - Confirm that _Deploy this boot image from the PXE-enabled distribution point_ is already checked.  If we're using the Tech Preview lab, select _x64_.  Click _OK_.
	
	![boot_image_confirm_pxe.PNG](/img/300/boot_image_confirm_pxe.PNG)
7. We'll be prompted to update the distribution points.  Click _Yes_, and the _Update Distribution Points Wizard_ will launch.

	![boot_image_update_distribution_points.png](/img/300/boot_image_update_distribution_points.png)

8. When the wizard launches, click _Next_.  At the _Summary_ screen click _Next_ again.  At the _Completion_ screen click _Close_.

9. Repeat steps 2-8 with _Boot image (x86)_.


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