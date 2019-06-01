---
layout: post
published: true
title: What's the Difference Between Tech Preview?
subtitle: 'About 12 Improvements, 8 Features, and 2 Vehicles'
date: '2019-05-27'
---

We can use the [Windows and Office Deployment Lab Kit](https://www.microsoft.com/en-us/evalcenter/evaluate-lab-kit) I posted about previously to try out the [technical preview](https://www.microsoft.com/en-us/evalcenter/evaluate-system-center-configuration-manager-and-endpoint-protection-technical-preview). 

##### This post will start the same as that first [SCCM lab set up blog post](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/).  However this environment will not be as pre-configured as an out-of-box Deployment Lab Kit.

### The Requirements
1. **Set up a host device** - For this lab, I'm using a Windows 10 Pro workstation with an old i7 CPU, 16GB of RAM, and a secondary 500GB hard drive.

2. **Enable Hyper-V** -  If Hyper-V is not yet installed, open _Turn Windows features on or off_, check _Hyper-V_, and click _OK_. Reboot.

    ![enable_hyper-v.PNG](/img/200/enable_hyper-v.PNG)
3. **Configure networking** - Launch Hyper-V as administrator, and open the _Virtual Switch Manager_.  Under _Virtual Switches_, select _New virtual network switch_.  Select _External_ and click _Create Virtual Switch_.  Name it _Lab_ and and leave everything else default.  Click _OK_, and if you are prompted with a warning, click _OK_ again.

    ![configure_virtual_switch.PNG](/img/200/configure_virtual_switch.PNG)

### The Setup
1. Download the kit from the link above.  It has the virtual machines and step-by-step documentation on how to configure services.

    ![labdownload.PNG](/img/200/labdownload.png)
2. Extract the lab zip file, preferably to a drive that is large and fast.

3. **Install** - Right click Setup.exe and run as administrator.  If prompted by SmartScreen, click _More Info_ and then click _Run anyway_.

    ![run_setup_exe.PNG](/img/200/run_setup_exe.PNG)
4. **Setup Wizard** - Click _Next_ all the way through to the end.  It will import all the VMs into Hyper-V.

5. **Configure VM Settings** - You should see HYD-DC1 and HYD-GW1 already running.  Shut them down.  We won't be using HYD-GW1 again.

6. **Domain Controller** - Right click HYD-DC1 and select _Settings_. Set _Maximum Memory_ to **2048**MB and leave _Enable Dynamic Memory_ checked.  Set CPU to **one** virtual processor.

7. **SCCM Server** - Right click HYD-CM1 and select _Settings_. Leave memory settings at default. Set CPU to **two** virtual processors.

    ![hyd-cm1_settings.PNG](/img/200/hyd-cm1_settings.PNG)

### NAT Networking
###### Note: We will NOT be using the external virtual switch called _Lab_ from _Step 3_ of the _Requirements_ section.  It was only necessary so that Setup.exe from the _Setup_ section would run.
1. **NAT Networking** - We'll use [Ami Casto's NAT network script](https://deploymentresearch.com/Research/Post/558/Setting-Up-New-Networking-Features-in-Server-2016 "Setting Up New Networking Features in Server 2016") instead of the Internet Gateway (HYD-GW1) to make the lab simpler.  You can learn more about NAT networking [here](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network "Set up a NAT network").

2.  **Prepare the Virtual Switch** - The Deployment Lab Kit creates it's own private network switch, so we need to make it an _Internal_ one to work with Ami's script.  In Hyper-V click _Virtual Switch Manager_.  Click on _HYD-CorpNet_.  Select _Internal network_ and click _OK_.

    ![internal_network.PNG](/img/200/internal_network.PNG)
3. **Customize the script** - On the host system, launch Windows Powershell ISE as Administrator.  Copy and paste the following code into the top script pane.  This is an edited version of Ami's code customized for our Microsoft lab.  Hit F5 to run it.
```powershell
    New-NetIPAddress –IPAddress 10.0.0.254 -PrefixLength 24 -InterfaceAlias "vEthernet (HYD-CorpNet)" 
    New-NetNat –Name HYD-CorpNetNATNetwork –InternalIPInterfaceAddressPrefix 10.0.0.0/24
```
4. We now have our host Windows 10 OS performing NAT on the internal virtual switch HYD-CorpNet.  Our VMs are already pointing to it as the default gateway.

### The Test
1. Power on HYD-DC1 and wait for the log on screen.  This is so our servers and workstations can talk to Active Directory.

2. Power on HYD-CM1 and log in.  The passwords for the local administrator accounts and for CORP\LabAdmin is _P@ssw0rd_

3. Confirm the SCCM server has internet access by launching command prompt and pinging 8.8.8.8.

### Installing the SCCM Technical Preview
1. **Download** - On the host system, download the [Technical Preview](https://www.microsoft.com/en-us/evalcenter/evaluate-system-center-configuration-manager-and-endpoint-protection-technical-preview).

    ![techpreviewdownload.png](/img/500/techpreviewdownload.png)
2. Right click the tech preview installer, select _Copy_.

3. On HYD-CM1, navigate to the Downloads folder and paste the file.

    ![copy_tech_preview.png](/img/500/copy_tech_preview.png)
4. Launch the tech preview installer, click _Unzip_.  It will extract the files to the root of C:\\.

5. **Uninstall Current Branch** - Navigate to C:\SC_Configmgr_SCEP_TechPreview1902\SMSSETUP\BIN\X64, right click _setup.exe_, select _Run as administrator_.

6. When the _System Center Configuration Manager Setup Wizard_ launches, click _Next_.  At the _Available Setup Options_ screen, the only option available is _Uninstall this Configuration Manager site_.  Click _Next_.  Leave default options, continue clicking _Next_, and accept any prompts to confirm uninstallation.  Reboot _HYD-CM1_.

    ![uninstall_site.png](/img/500/uninstall_site.png)
7. **Patch** - The installer will fail if there are pending reboots.  Open Windows Update, check for and install updates.  Reboot _HYD-CM1_.

    ![install_windows_updates.png](/img/500/install_windows_updates.png)
8. **Install** - Navigate to C:\SC_Configmgr_SCEP_TechPreview1902\SMSSETUP\BIN\X64 again. Right click _setup.exe_, select _Run as administrator_.

9. When the _System Center Configuration Manager Setup Wizard_ opens, click _Next_.  This time at the _Available Setup Options_ screen, check the box for _Use typical installation options for a stand-alone primary site_.  Click _Next_.

    ![techpreview_setup_wizard.png](/img/500/techpreview_setup_wizard.png)
10. At the _Product License Terms_ screen, check all the license agreements boxes and click _Next_.

11. Leave the default _Download required files_ option, and click _Browse_.  Choose the _Downloads_ folder and click _OK_.  Click _Next_.

    ![techpreview_save_prereqs.png](/img/500/techpreview_save_prereqs.png)
12. At the _Site and Installation Settings_ screen, set the site code to **CHQ**.  Set the site name to **cm1.corp.contoso.com**.  Leave everything else and click _Next_.

    ![techpreview_site_settings.png](/img/500/techpreview_site_settings.png)
13. Continue accepting defaults and click _Next_ until the prequisite check finishes.  Ignore any warnings and click _Begin Install_.

    ![techpreview_begin_install.png](/img/500/techpreview_begin_install.png)
14. When the installation completes, click _Close_.

    ![techpreview_install_complete.png](/img/500/techpreview_install_complete.png)

### The Finishing Touch
###### Now that we've got a functioning domain, SCCM server, and internet access, it's time to update.
1. On HYD-CM1, launch the SCCM console.  Navigate to the _Administration_ node and select _Updates and Servicing_. Click on _Check for updates_.

    ![check_for_updates.png](/img/500/check_for_updates.png)
2. If the latest version hasn't already started downloading, select it (in this case 1902), right click and choose _Download_.

3. Once it is downloading, on the bottom pane click on the _Show Status_ link.

4. On the _Updates and Servicing Status_ page for our chosen update, right click update package and choose _Show Status_ again.

5. From here, we can follow the download AND installation statuses of the latest SCCM upgrade.

6. Once the download is complete, go back to the _Administration_ node and click on _Updates and Servicing_ again.  The update we downloaded should now say _Ready to install_.

7. **Update** - Right click the update and select _Install Update Pack_. Check _Ignore any prerequisite check warnings..._ and click _Next_.  
    ![ignore_prereq_warnings.PNG](/img/500/ignore_prereq_warnings.png)

8. At the _Features included in update pack_ screen, check all available options and click _Next_ until we reach the _License Terms_.  Check the box, and keep clicking _Next_ until the wizard completes successfully.  Click _Close_.

    ![select_features.png](/img/500/select_features.png)
9. Repeat steps 3 and 4 and watch the update installation progress.  Refresh until the Update Wizard is complete and click _Close_.

	![install_status.PNG](/img/500/install_status.PNG)
10. Close the SCCM console and relaunch it.  We will be prompted to upgrade the console to the new version. Click _OK_, and if prompted for elevation click _OK_ again.

#### Congratulations!  We now have a functional SCCM tech preview environment we can configure and customize.
**Next up** - [Operating System Deployment over PXE](https://doug.seiler.us/2018-06-05-you-down-with-osd/)