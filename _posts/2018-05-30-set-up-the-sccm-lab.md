---
layout: post
published: true
title: In the MEMCM Lab
date: '2018-05-30'
subtitle: 'With a pen and a pad, trying to get this deployment off'
---
######  Updated: 7/8/2020
My first lab was [Johan's hydration kit](https://deploymentresearch.com/hydration-kit-for-windows-server-2019-sql-server-2017-and-configmgr-current-branch/).  It's incredibly powerful, customizable, and educational.  Unfortunately it takes a little more time and know-how than a novice like myself was initially prepared for.

However, at MMS [Steve Jesok](https://twitter.com/sejesok) pointed out that Microsoft provides an **all-in-one** solution: the [Windows and Office Deployment Lab Kit](https://www.microsoft.com/en-us/evalcenter/evaluate-lab-kit).  Within minutes, we can have a fully functional domain controller and MEMCM server.


### The Requirements
1. **Set up a host device** - For this lab, I'm using a Windows 10 Pro workstation with an old i7 CPU, 16GB of RAM, and a secondary 500GB hard drive.

2. **Enable Hyper-V** -  If Hyper-V is not yet installed, open _Turn Windows features on or off_, check _Hyper-V_, and click _OK_. Reboot.

    ![enable_hyper-v.PNG](/img/200/enable_hyper-v.PNG)
3. **Configure networking** - Launch Hyper-V as administrator, and open the _Virtual Switch Manager_.  Under _Virtual Switches_, select _New virtual network switch_.  Select _External_ and click _Create Virtual Switch_.  Name it _Lab_ and and leave everything else default.  Click _OK_, and if you are prompted with a warning, click _OK_ again.

    ![configure_virtual_switch.PNG](/img/200/configure_virtual_switch.PNG)

### The Setup
1. Download the kit from the link above.  It has the virtual machines and step-by-step documentation on how to configure services.  This is the **only** thing we need to download.

    ![labdownload.PNG](/img/200/labdownload.png)
2. Extract the lab zip file, preferably to a drive that is large and fast.

3. **Install** - Right click Setup.exe and run as administrator.  If prompted by SmartScreen, click _More Info_ and then click _Run anyway_.

    ![run_setup_exe.PNG](/img/200/run_setup_exe.PNG)
4. **Setup Wizard** - Click _Next_ all the way through to the end.  It will import all the VMs into Hyper-V.

    ![setup_wizard.png](/img/200/setup_wizard.png)

5. **Configure VM Settings** - You should see HYD-DC1 and HYD-GW1 already running.  Shut them down.  We won't be using HYD-GW1 again.

6. **Domain Controller** - Right click HYD-DC1 and select _Settings_. Set _Maximum Memory_ to **2048**MB and leave _Enable Dynamic Memory_ checked.  Set CPU to **one** virtual processor.

    ![hyd-dc1_settings.png](/img/200/hyd-dc1_settings.png)
7. **MEMCM Server** - Right click HYD-CM1 and select _Settings_. Leave memory settings at default. Set CPU to **two** virtual processors.

    ![hyd-cm1_settings.PNG](/img/200/hyd-cm1_settings.PNG)

### NAT Networking
###### Note: We will NOT be using the external virtual switch called _Lab_ from _Step 3_ of the _Requirements_ section.  It was only necessary so that Setup.exe from the _Setup_ section would run.
1. **NAT Networking** - We'll use [Ami Arwidmark's NAT network script](https://deploymentresearch.com/Research/Post/558/Setting-Up-New-Networking-Features-in-Server-2016 "Setting Up New Networking Features in Server 2016") instead of the Internet Gateway (HYD-GW1) to make the lab simpler.  You can learn more about NAT networking [here](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network "Set up a NAT network").

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

3. Confirm the MEMCM server has internet access by launching command prompt and pinging 8.8.8.8.

4. Give HYD-CM1 another moment for services to start up.  Launch the _Microsoft Endpoint Manager Configuration Manager Console_ and confirm that it loads successfully.

    ![memcm_console.png](/img/200/memcm_console.png)

Refer to the [**troubleshooting**](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/#troubleshooting) section if anything isn't working at this point

### The Finishing Touch
###### Now that we've got a functioning domain, MEMCM server, and internet access, it's time to update.
1. In the MEMCM console, navigate to the _Administration_ node and select _Updates and Servicing_. Click on _Check for updates_.

    ![check_for_updates.png](/img/200/check_for_updates.png)
2. If the latest version hasn't already started downloading, select it (in this case 1902), right click and choose _Download_.

3. Once it is downloading, on the bottom pane click on the _Show Status_ link.

4. On the _Updates and Servicing Status_ page for our chosen update, right click update package and choose _Show Status_ again.

5. From here, we can follow the download AND installation statuses of the latest MEMCM upgrade.

    ![updates_and_servicing.PNG](/img/200/updates_and_servicing.PNG)
6. Once the download is complete, go back to the _Administration_ node and click on _Updates and Servicing_ again.  The update we downloaded should now say _Ready to install_.

7. Right click the update and select _Install Update Pack_. Check _Ignore any prerequisite check warnings..._ and click _Next_ until we reach the _License Terms_.  Check the box, and keep clicking _Next_ until the wizard completes successfully.  Click _Close_.

    ![ignore_prereq_warnings.PNG](/img/200/ignore_prereq_warnings.png)
	![install_update_pack.PNG](/img/200/install_update_pack.PNG)
    
8. Repeat steps 3 and 4 and watch the update installation progress.  Refresh until the Update Wizard is complete and click _Close_.

	![install_status.PNG](/img/200/install_status.PNG)
9. Close the MEMCM console and relaunch it.  We will be prompted to upgrade the console to the new version. Click _OK_, and if prompted for elevation click _OK_ again.

	![update_console.PNG](/img/200/update_console.PNG)

#### Congratulations!  We now have a functional MEMCM environment we can configure and customize.
**Next up** - [Operating System Deployment over PXE](https://doug.seiler.us/2018-06-05-you-down-with-osd/)

### Troubleshooting

###### If there are any obstacles during set up, we can try some of these troubleshooting tips
1. **Firewall** - If you cannot ping 8.8.8.8, we don't have access to the internet.  From CM1, try pinging DC1 at 10.0.0.6.  If that works, try pinging the NAT gateway at 10.0.0.254.  If that doesn't work, try temporarily disabling the firewall as that might be blocking access.  
You may need to remove and redo the NAT networking as well, so run the following command in an elevated Powershell terminal:
```powershell
    Remove-NetIPAddress -IPAddress 10.0.0.254
    Remove-NetNat

    New-NetIPAddress –IPAddress 10.0.0.254 -PrefixLength 24 -InterfaceAlias "vEthernet (HYD-CorpNet)" 
    New-NetNat –Name HYD-CorpNetNATNetwork –InternalIPInterfaceAddressPrefix 10.0.0.0/24
```
2. **NAT** - If you can ping 10.0.0.254 but STILL can't ping 8.8.8.8, make sure HYD-GW1 is powered off.  If it is, the issue is on the host system.  From the host system, ping CM1 at 10.0.0.7 to confirm NAT is working.  If NAT is working, from CM1 ping the host IP of the physical adapter.

3. **Subnet** - The lab network is 10.0.0.0/24.  If our home network is also on 10.0.0.0/24 we'll have trouble getting out.  We will either need to ditch the NAT and rely on GW1, or re-IP DC1 and CM1 and our NAT configuration on a different network.  Just keep in mind in subsequent blog posts we'll need to adjust networking respectively.  
For example if you wanted to change the lab from the default 10.0.0.0/24 network to a 10.11.12.0/24 network, change the CM1 IP to 10.11.12.7 and the DC1 IP to 10.11.12.6.  Remove the NAT config and make a new one on that network like so:
```powershell
    Remove-NetIPAddress -IPAddress 10.0.0.254
    Remove-NetNat

    New-NetIPAddress –IPAddress 10.11.12.254 -PrefixLength 24 -InterfaceAlias "vEthernet (HYD-CorpNet)" 
    New-NetNat –Name HYD-CorpNetNATNetwork –InternalIPInterfaceAddressPrefix 10.11.12.0/24
```