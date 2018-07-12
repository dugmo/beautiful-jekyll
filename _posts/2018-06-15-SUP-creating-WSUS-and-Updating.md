---
layout: post
published: false
title: 'His SUPs are steady, knees weak, WSUS ready'
subtitle: There's logic to his ADR already. Mom's spaghetti.
date: '2018-07-10'
---
## Patching with SCCM and WSUS
An essential function of SCCM is deploying patches and providing compliance reporting around that patching.

### The Overview
1. [Prepare our environment for WSUS](https://doug.seiler.us/2018-07-10-SCCM-WSUS-Patching/#prepare-the-environment)
2. [Install WSUS](https://doug.seiler.us/2018-07-10-SCCM-WSUS-Patching/#install-wsus)
3. [Configure SCCM](https://doug.seiler.us/2018-07-10-SCCM-WSUS-Patching/#configure-sccm)
4. [Optimize updates](https://doug.seiler.us/2018-07-10-SCCM-WSUS-Patching/#optimize-wsus)
5. [Set up an automatic deployment rule](https://doug.seiler.us/2018-07-10-SCCM-WSUS-Patching/#automatic-deployment-rules)
6. Examine update compliance reports

### The Sources
1. [Rich Mawdsley's three-part series for configuring WSUS](https://everythingsccm.com/2017/03/27/configuring-wsus-with-sccm-current-branch-server-2016-part-i/) is my defacto guide for initial setup.  This post is honestly just going to be 99% shamelessly stealing Part 1 with minor customizations.
2. [Justin Chalfant's WSUS maintenance guide](https://setupconfigmgr.com/maintaining-the-wsus-catalog-by-declining-updates-for-better-sccm-scanning) has a terrific step-by-step video and the definitive list of WSUS maintenance resource links.
3. [Bryan Dam's "Software Update Maintenance" script](http://damgoodadmin.com/2018/04/17/software-update-maintenance-script-updated-all-the-wsusness/) is necessary for maintaining an efficient set of updates.  It can be automated to remove/expire update metadata that our environment will never need, but otherwise clogs up and drastically slows down the patch scanning process.

## Prepare the environment
#### Expand the Server
###### DISCLAIMER: It is not considered best practice to install SCCM or keep any content on the C: drive.  However that is how the 365 Powered Device Lab is set up by default, so we need to increase the amount of disk space to make room for WSUS.
1. Launch Hyper-V.  Right-click _HYD-CM1_ and select _Settings_.
2. Select _Hard Drive_.  Copy the path to the _Virtual hard disk_.  Click _Cancel_.

	![prepare_vhdx-location.PNG]({{site.baseurl}}/img/prepare_vhdx-location.PNG)
3. Click _Edit Disk_.  At the _Before you Begin_ screen click _Next_.  Paste the path to the _HYD-CM1_ disk.  Click _Next_.

	![prepare_select-vhdx.PNG]({{site.baseurl}}/img/prepare_select-vhdx.PNG)
4. Choose _Expand_ and click _Next_.
5. Set _New size_ to **300** or so and click _Next_.  Click _Finish_.

	![prepare_expand-300gb.png]({{site.baseurl}}/img/prepare_expand-300gb.png)
6. Connect to _HYD-CM1_.  Launch the _Create and format hard disk partitions_ control panel.
7. Click _Action_ and choose _Rescan disks_.
8. Right-click _Windows (C:)_ and choose _Extend Volume_.  Click _Next_.
	
    ![prepare_diskmgmt-extend.png]({{site.baseurl}}/img/prepare_diskmgmt-extend.png)
9. The wizard should already have the amount at it's maximum.  Click _Next_ then click _Finish_.

	![prepare_diskmgmt-maximum.png]({{site.baseurl}}/img/prepare_diskmgmt-maximum.png)

#### Update Group Policy
###### We need to inform our systems that they will check for updates from WSUS/SCCM.  Since it's a lab we'll just use the built-in Default Domain Policy.
1. Log into the domain controller _HYD-DC1_.
2. Launch _Group Policy Management_.
3. Expand _Domains_ -> _corp.contoso.com_.
4. Right-click _Default Domain Policy_ and select _Edit_.

	![gpo_edit-default-domain-policy.png]({{site.baseurl}}/img/gpo_edit-default-domain-policy.png)
5. Expand _Computer Configuration_ -> _Policies_ -> _Administrative Templates_, _Windows Components_ -> _Windows Update_.

	![gpo_windows-updates.png]({{site.baseurl}}/img/gpo_windows-updates.png)
6. Double-click _Specify intranet Microsoft update service location_ and set it to **Enabled**.  For _Set the intnranet update service for detecting updates_ and _Set the intranet statistics server_ enter **http://cm1.corp.contoso.com:8530** and click _OK_.

    ![gpo_set-intranet-update-service.png]({{site.baseurl}}/img/gpo_set-intranet-update-service.png)
	![gpo_enable-intranet-update-service.png]({{site.baseurl}}/img/gpo_enable-intranet-update-service.png)
7. Double-click _Allow signed updates from an intranet Microsoft update service location_ and set it to **Enabled**.  This will allow us to deploy third party updates later if we want.  Click _OK_.
	
    ![gpo_allow-signed-updates.png]({{site.baseurl}}/img/gpo_allow-signed-updates.png)
	![gpo_enable-allow-signed-updates.png]({{site.baseurl}}/img/gpo_enable-allow-signed-updates.png)
8. Close the _Group Policy Management Editor_.

## Install WSUS
###### The default WSUS configuration in the 365 Powered Device Lab needs to be redone
#### Remove Roles and Features
1. Connect to _HYD-CM1_ and launch _Server Manager_.  Click _Manage_ and select _Remove Roles and Features_.
	
    ![wsus_remove-role_select.png]({{site.baseurl}}/img/wsus_remove-role_select.png)
2. If the _Before you begin_ screen comes up first, click _Next_.
3. **Server Selection** - Leave _CM1.corp.contoso.com_ selected and click _Next_.

	![wsus_remove-role_server-selection.png]({{site.baseurl}}/img/wsus_remove-role_server-selection.png)
4. **Remove server roles** - Uncheck _Windows Server Update Services_.  When the _Remove Roles and Features Wizard_ pops up, click _Remove Features_.  Click _Next_.
	
    ![wsus_remove-role_remove-roles-wizard.png]({{site.baseurl}}/img/wsus_remove-role_remove-roles-wizard.png)
	![wsus_remove-role_remove-server-roles.png]({{site.baseurl}}/img/wsus_remove-role_remove-server-roles.png)
5. **Remove features** - Don't make any changes.  Click _Next_.  Click _Remove_.

	![wsus_remove-role_confirmation-remove.png]({{site.baseurl}}/img/wsus_remove-role_confirmation-remove.png)
6. Wait for removal to succeed and click _Close_.

	![wsus_remove-role_results-close.png]({{site.baseurl}}/img/wsus_remove-role_results-close.png)
7. Reboot _HYD-CM1_.

#### Add Roles and Features
1. Log into _HYD-CM1_ and launch _Server Manager_.  Click _Add roles and features_.  If the _Before you begin_ screen comes up first, click _Next_.

	![wsus_add-role_add-roles-and-features.png]({{site.baseurl}}/img/wsus_add-role_add-roles-and-features.png)
2. **Installation Type** - Select _Role-based or feature-based installation_ and click _Next_.
3. Leave _CM1.corp.contoso.com_ selected and click _Next_.
	
    ![wsus_add-role_server-selection.png]({{site.baseurl}}/img/wsus_add-role_server-selection.png)
4. **Select server roles** - Check _Windows Server Update Services_.  When the _Add Roles and Features Wizard_ pops up, click _Add Features_.  Click _Next_.

	![wsus_add-role_add-roles-and-features-wizard.png]({{site.baseurl}}/img/wsus_add-role_add-roles-and-features-wizard.png)
	![wsus_add-role_select-server-roles.png]({{site.baseurl}}/img/wsus_add-role_select-server-roles.png)
5. **Select features** - Expand _.NET Framework 4.6 Features_, expand _WCF Services_, and check all features.  For any wizards that pop up, click _Add Features_.  Click _Next_, and click _Next_ again.

	![wsus_add-role_select-features.png]({{site.baseurl}}/img/wsus_add-role_select-features.png)
6. **Select role services** - Uncheck _WID Connectivity_ and check _SQL Server Connectivity_.  Click _Next_.
	
    ![wsus_add-role_select-role-services.png]({{site.baseurl}}/img/wsus_add-role_select-role-services.png)
7. **Content location selection** - Leave _Store updates in the following location_ checked, and paste **C:\PackageSource\WSUS**.  Click _Next_.

	![wsus_add-role_content-location.png]({{site.baseurl}}/img/wsus_add-role_content-location.png)
8. **Database instance selection** - Type **CM1.CORP.CONTOSO.COM** and click _Check connection_.  Click _Next_.

	![wsus_add-role_database-instance-selection.png]({{site.baseurl}}/img/wsus_add-role_database-instance-selection.png)
9. At the confirmation screen, check _Restart the destination server automatically if required_ and click _Install_.

	![wsus_add-role_confirmation.png]({{site.baseurl}}/img/wsus_add-role_confirmation.png)
10. Wait for the installation to succeed and click _Launch Post-Installation tasks_. 

	![wsus_add-launch-post-installation-tasks.png]({{site.baseurl}}/img/wsus_add-launch-post-installation-tasks.png)
11. Wait for configuration to successfuly complete and click _Close_.

	![wsus_add-config-success-completed.png]({{site.baseurl}}/img/wsus_add-config-success-completed.png)
12. Reboot _HYD-CM1_.

#### Configure WSUS
###### [Rich's guide](https://everythingsccm.com/2017/03/27/configuring-wsus-with-sccm-current-branch-server-2016-part-i/) will do a partial set up and then cancel setup.  The goal is to speed up the SCCM sync and reduce metadata.
1. Connect to _HYD-CM1_ and launch _Windows Server Update Services_.
2. When the _Windows Sever Update Services Configuration Wizard_ launches, click _Next_ until you reach _Connect to Upstream Server_.
3. **Connect to Upstream Server** - Click _Start Connecting_.  When it completes, click _Next_.

	![wsus_config-wizard_connect-to-upstream.png]({{site.baseurl}}/img/wsus_config-wizard_connect-to-upstream.png)
4. **Choose Languages** - Select _Download updates in all languages, including new languages_ and click _Next_.

	![wsus_config-wizard_languages.png]({{site.baseurl}}/img/wsus_config-wizard_languages.png)
5. **Choose Products** - Leave the defaults and click _Next_.
6. **Choose Classifications** - Click _Cancel_ to abort set up.  We'll finish configuration from within SCCM.  Close the _Update Services_ console.
	
    ![wsus_config-wizard_classifications.png]({{site.baseurl}}/img/wsus_config-wizard_classifications.png)

#### Folder and Share permissions
###### The script we used from Odd-Magne in the [OSD guide](https://doug.seiler.us/2018-06-05-you-down-with-osd#sources) sets the permissions on our WSUS folder.  If you've performed that setup you can skip this step.
1. If the network access account and NETWORK SERVICE account do not yet have permission to C:\PackageSource\WSUS, continue with the rest of the steps.
2. Log into _HYD-CM1_ and launch Windows Powershell ISE as administrator.  Paste the following code into the top script pane.  It's a truncated and modified version of [Odd-Magne's package source script](https://sccmguru.wordpress.com/2012/10/25/configuration-manager-folder-structure/).
```powershell
  #Set the Following Parameters
  $Source = 'C:\PackageSource'
  $ShareName = 'PackageSource'
  $NetworkAccount = 'CORP\CM_NetAcc'
  
  #Create SCCMDeploymentPackages Directory
  New-Item -ItemType Directory -Path "$Source\WSUS\SCCMDeploymentPackages"
  
  #Create the Share and Permissions
  New-SmbShare -Name "SCCMDeploymentPackages” -Path "$Source\WSUS\SCCMDeploymentPackages" -CachingMode None -FullAccess $NetworkAccount,"NETWORK SERVICE"

  #Set Security Permissions
  $Acl = Get-Acl "$Source\WSUS"
  $Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("$NetworkAccount","FullControl", "ContainerInherit, ObjectInherit", "None", "Allow")
  $Acl.SetAccessRule($Ar)
  Set-Acl "$Source\WSUS" $Acl

  $Acl = Get-Acl "$Source\WSUS"
  $Ar = New-Object System.Security.AccessControl.FileSystemAccessRule("NETWORK SERVICE","FullControl", "ContainerInherit, ObjectInherit", "None", "Allow")
  $Acl.SetAccessRule($Ar)
  Set-Acl "$Source\WSUS" $Acl
```
	![permissions_powershell-script.png]({{site.baseurl}}/img/permissions_powershell-script.png)

#### Configure IIS
###### Microsoft recommends [modifying the IIS application pool](https://docs.microsoft.com/en-us/sccm/sum/plan-design/plan-for-software-updates) for WSUS when used with a Software Update Point.
1. Connect to _HYD-CM1_ and launch the _Internet Information Services (IIS) Manager_.
2. Expand _CM1_ and select _Application Pools_.

	![iis_applications-pools.png]({{site.baseurl}}/img/iis_applications-pools.png)
3. Right-click _WsusPool_ and select _Advanced Settings_.

	![iis_wsus-pool.png]({{site.baseurl}}/img/iis_wsus-pool.png)
4. Set _Queue Length_ to **2000** and set _Private Memory Limit (KB)_ to **0**.

	![iis_wsus-pool_queue-length.png]({{site.baseurl}}/img/iis_wsus-pool_queue-length.png)
	![iis_wsus-pool_private-memory-limit.png]({{site.baseurl}}/img/iis_wsus-pool_private-memory-limit.png)
5. Click _OK_ and reboot _HYD-CM1_.

## Configure SCCM
#### The Software Update Point
###### The 365 Powered Device Lab already has the Software Update Point role installed, so we just need to configure it.
1. Connect to _HYD-CM1_ and launch the _Configuration Manager Console_.
2. Expand _Administration_ -> _Site Configuration_, and select the _Sites_ node.
3. Select _CHQ - Constoso Headquarters_ and click the _Configure Site Components_ dropdown from the ribbon.  Select _Software Update Point_.

	![sup_configure-site-components.png]({{site.baseurl}}/img/sup_configure-site-components.png)
4. **Classifications tab** - Select everything except _Feature Updates_, _Tools_, and _Upgrades_.

	![sup_classifications.png]({{site.baseurl}}/img/sup_classifications.png)
5. **Products tab** - See if Windows 10 is there.  If so, select it and uncheck everything else.

	![sup_products.png]({{site.baseurl}}/img/sup_products.png)
6. **Sync Schedule tab** - Check _Enable synchroniztion on a schedule_ and select _Custom schedule_.  Click _Customize_.
7. On the _Custom Schedule_ page, select: _Monthly_, _Recur every 1 months on_, and _The Second Tuesday_.  Set the _Start_ time to something after noon, say **2:00 PM**.  Remember this for later when we schedule the cleanup script.  Click _OK_.

	![sup_sync-schedule.png]({{site.baseurl}}/img/sup_sync-schedule.png)	
8. Check _Alert when synchronization fails on any site in the hierarchy_.
9. **Supersedence Rules tab** - Set _Months to wait before a superseded software update is expired_ to **1**.  Check _Run WSUS cleanup wizard_.

	![sup_supersedence.png]({{site.baseurl}}/img/sup_supersedence.png)
10. **Languages tab** -  Uncheck everything except _English_ for both columns.  Click _OK_.

	![sup_languages.png]({{site.baseurl}}/img/sup_languages.png)

#### Synchronize Software Updates
1. In the SCCM console, expand the _Software Library_ -> _Software Updates_ node.
2. Select _All Software Updates_ and click the _Synchronize Software Updates_ button in the ribbon.  Click _Yes_.

	![software-updates_synchronize-software-updates.png]({{site.baseurl}}/img/software-updates_synchronize-software-updates.png)
3. Navigate to the _Monitoring_ -> _Software Update Points Synchronization Status_ node.  Look at the fields in the bottom pane, and wait for the _Synchronization Status_ to change to _Completed_.

	![software-updates_synchronization-status-completed.png]({{site.baseurl}}/img/software-updates_synchronization-status-completed.png)
6. Alternatively, you can open _"C:\Program Files\Microsoft Configuration Manager\Logs\wsyncmgr.log"_ in CMTrace and monitor progress.
7. Once complete, navigate to the _Software Library_ -> _Software Updates_ -> _All Software Updates_ node.  It should now be full of patches.
	
    ![software-updates_all-software-updates-patches.png]({{site.baseurl}}/img/software-updates_all-software-updates-patches.png)

## Optimize WSUS
###### Justin did a great write up on optimizing WSUS, including the set up for Bryan's cleanup script.  We're JUST going to focus on [configuring and scheduling the cleanup](https://www.youtube.com/watch?v=wqBaTp855sk&feature=youtu.be&t=946).
#### Configure the script
1. On _HYD-CM1_, download [Bryan Dam's Software Update Maintenance Script](http://damgoodadmin.com/2018/04/17/software-update-maintenance-script-updated-all-the-wsusness/).
2. Create a folder in _C:\PackageSource\Script_ called **SAM**.  Extract the script and plugins folder to _C:\PackageSource\Script\SAM_.

	![script_extract.png]({{site.baseurl}}/img/script_extract.png)
3. Navigate to _C:\PackageSource\Script\SAM\Plugins\Disabled_.  Right-click _Decline-Windows10Versions.ps1_ and choose _Edit_ so it opens in PowerShell ISE.
4. Un-comment (remove the #) before _UnsupportedVersions = @("1511")_.
5. Add all versions we do NOT want to support.  Since I'm deploying 1703 in the lab with the plans to upgrade to 1803 (or later), mine would look like **$UnsupportedVersions = @("1507","1511","1607","1709")**.  Save the script and close PowerShell ISE.

	![script_windows10versions.png]({{site.baseurl}}/img/script_windows10versions.png)
6. Move the _Decline-Windows10Versions.ps1_ and _Decline-Windows10Languages.ps1_ scripts out of the _Disabled_ folder and into the root _Plugins_ folder.

	![script_plugins.png]({{site.baseurl}}/img/script_plugins.png)

#### Schedule the script
1. On HYD-CM1, launch _Task Scheduler_.
2. Click _Create Basic Task_.

	![task_create-basic-task.png]({{site.baseurl}}/img/task_create-basic-task.png)
3. Name it **WSUS Maintenance** and click _Next_.

	![task_name.png]({{site.baseurl}}/img/task_name.png)
4. **Trigger wizard** - Select _Monthly_ and click _Next_.

	![task_trigger.png]({{site.baseurl}}/img/task_trigger.png)
5. **Monthly wizard** - Click the _Months_ dropdown and check _Select all months_.  Select _On_ and set it to _Second Tuesday_.  Set the _Start_ time to two hours after the _WSUS sync_ we set above.  In this example, we'll set the task to run at **4:00 PM**.  Click _Next_.

	![task_monthly.png]({{site.baseurl}}/img/task_monthly.png)
6. **Action wizard** - Select _Start a program_ and click _Next_.

	![task_action.png]({{site.baseurl}}/img/task_action.png)
7. **Start a Program wizard** - Set _Program/Script_ to **powershell.exe**.  Set _Add arguments_ to **-NoLogo -NoProfile -NonInteractive -ExecutionPolicy ByPass -command C:\PackageSource\Script\SAM\Invoke-DGASoftwareUpdateMaintenance.ps1 -DeclineSuperseded -UpdateListOutputFile C:\PackageSource\Logs\DeclinedUpdates.csv -DeclineByTitle @('*Itanium*','*ia64*','*Beta*') -DeclineByPlugins -CleanSUGs -RemoveEmptySUGs -RunCleanUpWizard -Force** and click _Next_.

	![task_start-a-program.png]({{site.baseurl}}/img/task_start-a-program.png)
8. **Finish** - Check _Open the Properties dialog for this task when I click Finish_ and click _Finish_.  The _WSUS Maintenence Properties screen_ will open.

	![task_finish.png]({{site.baseurl}}/img/task_finish.png)
9. **General tab** Click _Change User or Group_.  Type **SYSTEM** and click _OK_.  Check _Run with highest privileges_ and click _OK_.

	![task_change_user.png]({{site.baseurl}}/img/task_change_user.png)
10. Right-click the _WSUS Maintenance_ task we created and select _Run_.

	![task_run.png]({{site.baseurl}}/img/task_run.png)
11. Open _C:\PackageSource\Script\SAM\updatemaint.log_ in CMtrace.exe and watch as the unnecessary updates are declined.

	![task_updatemaint-log.png]({{site.baseurl}}/img/task_updatemaint-log.png)

## Automatic Deployment Rules
###### ADRs are the method by which we schedule the creation of a Software Update Group, define the updates for the SUG, and select the collection to which we deploy.
#### Create the collection
###### To avoid ever deploying anything to the built-in _All Systems_ collection, we'll create one for workstations.  Anders Rodland has a great [list of collection query examples](https://www.andersrodland.com/ultimate-sccm-querie-collection-list/).
1. Connect to _HYD-CM1_ and launch the _Configuration Manager Console_.
2. Navigate to the _Assets and Compliance_ -> _Device Collections_ node.
3. Click _Create_ and select _Create Device Collection_.

	![adr_create-device-collection.png]({{site.baseurl}}/img/adr_create-device-collection.png)
4. In the _Create Device collection Wizard_, name the collection **All Workstations**.  Set the limiting collection to _All Systems_ and click _Next_.

	![adr_collection-name.png]({{site.baseurl}}/img/adr_collection-name.png)
5. **Membership Rules wizard** - click the _Add Rule_ dropdown and select _Query Rule_.

	![adr_query-rule.png]({{site.baseurl}}/img/adr_query-rule.png)
6. At the _Query Rules Properties_ screen, set the name to **All Workstations** and click _Edit Query Statement_.
7. **Criteria tab** - On the _Criteria_ tab click the star to add criterion.
8. At the _Criterion Propteries_ screen, leave _Criterion Type_ as **Simple value**.  Click the _Select_ button.
9. Set the _Attribute class_ to **System Resource** and set the _Attribute_ to **Operating System Name and Version**.  Click _OK_.

	![adr_query-criteria.png]({{site.baseurl}}/img/adr_query-criteria.png)
10. Set the _Operator_ to **is like**.  Click the _Value_ button.
11. Set the _Values_ to **Microsoft Windows NT Workstation%** and click _OK_.

	![adr_query-value.png]({{site.baseurl}}/img/adr_query-value.png)
12. Click _OK_ to close the _Criterion Properties_ screen.  Click _OK_ again to close the _Query Statement 
Properties_ screen.  Click _OK_ yet again to close the _Query Rule Properties_ screen.

	![adr_query-properties.png]({{site.baseurl}}/img/adr_query-properties.png)
13. Check _Schedule a full update on this collection_ and click _Schedule_.
14. Select _Custom interval_ and set it to recur every **1** days.  Change the _Start_ time to **11:59 PM**.  Click _OK_.

	![adr_collection-schedule.png]({{site.baseurl}}/img/adr_collection-schedule.png)
15.  Click _Next_.  At the _Summary_ screen, click _Next_.  At the _Completion_ screen, click _Close_.

#### Create the ADR
1. In the _Configuration Manager Console_, expand the _Software Library -> Software Updates_ node.
2. Right-click _Automatic Deployment Rules_ and select _Create Automatic Deployment Rule_ to launch the _Create Automatic Deployment Rule Wizard_.

	![adr_create-automatic-deployment-rule.png]({{site.baseurl}}/img/adr_create-automatic-deployment-rule.png)
3. **General** - Set the name to **Workstation Patching**.  For _Collection_ click _Browse_ select _All Workstations_ and click _OK_.  Leave the rest of the settings default and click _Next_.

	![adr_name.png]({{site.baseurl}}/img/adr_name.png)
4. **Deployment Settings** - Leave defaults and click _Next_.
5. **Software Updates** - Check _Product_, _Required_, _Superseded_, and _Update Classification_.

	![adr_updates-filter-before.png]({{site.baseurl}}/img/adr_updates-filter-before.png)
6. For _Product_, click _items to find_.  Check _Windows 10_ and click _OK_.
7. For _Required_, click _text to find_.  Type **>0** and click _Add_.  Click _OK_.
8. For _Superseded_, click _value to find_.  Select **No** and click _OK_.
9. For _Update Classification_, click _items to find_.  Check _Critical Updates_, _Security Updates_, _Update Rollups_, and _Updates_.  Click _OK_.  Click _Next_.

	![adr_updates-filter-after.png]({{site.baseurl}}/img/adr_updates-filter-after.png)
10. **Evaluation Schedule** - Select _Run the rule on a schedule_.  We want to provide time between the SUP sync, the optimization script scheduled task, and the ADR evaluation.  Click _Customize_.
11. On the _Custom Schedule_ page, select: _Monthly_, _Recur every **1** months on_, and _The Second Tuesday_.  We've set the WSUS sync to 2:00pm, the script is scheduled for 4:00pm, so we'll set the _Start_ time to **6:00 PM**.  Click _OK_ and click _Next_.

	![adr_customize-schedule.png]({{site.baseurl}}/img/adr_customize-schedule.png)
12. Set _Installation deadline_ to **As soon as possible** so our patches start deploying immediately.  Click _Next_.

	![adr_deployment-schedule.png]({{site.baseurl}}/img/adr_deployment-schedule.png)
13. **User Experience** - For _Suppress the system restart on the following devices_ check **Servers** and **Workstations**.  Check _If any update in this deployment requires a system restart, run updates deployment evaulation cycle after restart_.  Click _Next_ until you reach the _Deployment Package_ wizard.

	![adr_user-experience.png]({{site.baseurl}}/img/adr_user-experience.png)
14. **Deployment Package** - Set the deployment package _Name_ to **WorkstationPatches**.  For _Package source_ click _Browse_.  In the address bar paste **\\CM1.CORP.CONTOSO.COM\PackageSource\WSUS\SCCMDeploymentPackages** and click _Select Folder_.  Check _Enable binary differential replication_ and click _Next_.

	![adr_deployment-package.png]({{site.baseurl}}/img/adr_deployment-package.png)
15. **Distribution Points** - Click _Add_ and select _Distribution Point_.  Check _CM1.CORP.CONTOSO.COM_ and click _OK_.  Click _Next_.

	![adr_distribution-point.png]({{site.baseurl}}/img/adr_distribution-point.png)
16. Click _Next_ until you reach the _Completion_ screen.  Click _Close_.
17. At this point we've already synced WSUS and performed a cleanup using Bryan's script.  Right-click the newly created _Workstation Patching_ ADR and select _Run Now_.

	![adr_run-now.png]({{site.baseurl}}/img/adr_run-now.png)
