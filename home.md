# Active Directory Domain Compromise via AS-REP roasting and Kerberoasting
Federico Castelli, IN2300056

## Setting up the DC
### Installing the Windows Server
1) To prepare an enviroment , on the page https://www.microsoft.com/it-it/evalcenter/download-windows-server-2022 choose Download ISO to obtain the file: `SERVER_EVAL_x64FRE_en-us` (it was chosen the english version).
After that: open VirtualBox, add a new VM; configure the first screen as follows:
+ Name: Choose a clear name, for example `Windows_Server_2022_Target`.
+ Machine Folder: Leave the default location.
+ ISO Image: Click the drop-down menu, select Other..., and browse to the `SERVER_EVALx64FRE_en-us` file located in your Downloads folder.
+ Type: Windows.
+ Version: Windows 2022 (64-bit).
+ Check the `Skip Unattended Installation` box. This is important to perform the standard manual installation to avoid potential errors.
+ Click Next.

2) On the next screen, dedicated to hardware:
+ Base Memory (RAM): Allocate at least 4096 MB (4 GB).
+ Processors (CPU): Allocate at least 2 CPUs.
+ Click `Next`.


3) Select Create a virtual hard disk now:
+ Disk Size: Set it to at least 50 GB or 60 GB. With less than 40 GB, you risk running into issues due to a full disk.
+ Click `Next`, and then click `Finish`.

4)  Start the Windows Installation, you will now see your new virtual machine in the list on the left side of VirtualBox:
+ Select it and click the `Start` button (the green arrow at the top).
+ IMPORTANT possibility: As soon as the VM’s black screen appears, you will see a white message saying "Press any key to boot from CD or DVD...". Immediately press the Spacebar on your keyboard. If you do not press a key within 2–3 seconds, the machine will fail to boot correctly (you may see a yellow screen or the UEFI Shell instead). If you miss the timing, click Machine → Reset in the top menu and try again; BUT if it appears for  a brief short of time and then porceeds normally, there are no problems.
+ The purple Windows Server Setup screen will appear. Click `Next`, then click `Install Now`.
+ After a few seconds, Windows will ask which version you want to install. You will be presented with four options: you must select a version that includes "(Desktop Experience)", preferably Standard (Desktop Experience); if you select the regular version without "(Desktop Experience)", Windows will be installed without a graphical user interface (GUI).


When you are given the choice between:
+ Upgrade: install Microsoft Server Operating System and keep files, settings, and applications
+ Custom: Install Microsoft Server Operating System only (advanced)

Select the second option: Custom: Install Microsoft Server Operating System only (advanced)
Since we are installing the virtual machine from scratch on an empty disk, there is no existing operating system to upgrade.
Immediately after clicking Custom, you will see a screen listing the available disks. You should see a single entry called:
"Drive 0 Unallocated Space" (this should correspond to the 50–60 GB you allocated in VirtualBox).
Select that entry by clicking on it.
Then click `Next`.
At this point, Windows will automatically create the necessary partitions and begin the actual installation process.

Then, let Windows complete the installation, and when it prompts you to set a password for the local `Administrator` account (when the desktop first starts), choose a simple, familiar password that you can remember easily (the choice of the name for the account should be locked for `Administrator`).


Before proceeding, it's important to have the two VMs being able to connect to each other:
1) Set up the network in VirtualBox without shutting down the machine, look at the top menu of the Windows VM in VirtualBox and click on `Devices -> Network -> Network Settings...`; in the `Attached to:` dropdown menu, select Host-only Adapter (`VirtualBox Host-Only Ethernet Adapter`). Below, check that the adapter name is the same one you use for Kali. Click `OK` (Do exactly the same in your Kali network settings if it is not already set to Host-Only in the same slot).

2) Configure the static IP on Windows ServerActive Directory strictly requires a fixed IP and the server to point to itself as DNS, to configure it graphically: inside Windows Server, press `Windows + R` on your keyboard to open the “Run” window. Type `ncpa.cpl` and press `Enter`. The Network Connections window will open. Right-click the `active network adapter` (e.g. Ethernet0) and select `Properties` . Double-click `Internet Protocol Version 4 (TCP/IPv4)`. Select `Use the following IP address` and enter the following details:
+ IP address: 192.168.56.10
+ Subnet mask: 255.255.255.0 (it will auto-fill as soon as you click it)
+ Default gateway: Leave it blank or set 192.168.56.10

In the DNS section below, select `Use the following DNS server addresses`:
+ Preferred DNS server: 192.168.56.10
Click `OK` and then `OK` again to save.

4) On the Powershell of the Windows Server: `Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False` to disable firewalls and allow Windows Server and Kali to communicate. This has to be done once you are entered in the desktop (you can do it in every moment before having to exchange communication with the Kali machine).

Once you click Finish, Windows will show the initial lock screen with the classic message “Press Ctrl+Alt+Delete to unlock”.

### Promote the Windows Server
To send this command inside the VirtualBox VM, go to the top menu of the VirtualBox window and click:
`Input → Keyboard → Insert Ctrl+Alt+Del`.

Once the desktop loads, choose `allow` when asked if you want that "Your PC to be discoverable by other PCs and devices on this network" and also a large window called Server Manager will open automatically.

+ In the top-right corner, click `Manage and select Add Roles and Features`.
+ In the wizard that opens, click `Next` through the first screens, leaving everything as default (Role-based, Select a server from the pool), until you reach the Server Roles screen.
+ In the list of roles, check `Active Directory Domain Services`. A pop-up will appear, click `Add Features` to confirm, then click `Next`.
+ Continue clicking `Next` through the following screens without changing anything, and finally click `Install`.
+ Wait about a minute for the progress bar to complete. Do not close the window.
+ Once installation is finished, you will see a blue link that says `Promote this server to a domain controller`.
+ Click it to open the final configuration wizard:
  + In the first screen `Deployment Configuration`, select the third option: `Add a new forest`.
  + In the Root domain name field, type the domain name that will use, for example `vuln.local`, then click `Next`.
  + On the next screen, you will be asked for a Directory Services Restore Mode (DSRM) password. Enter the same password you always use for the lab (e.g. Password123!) in both fields and click `Next`.
  + In the following screens `DNS Options, Additional Options, Paths`, do not change anything, just click `Next` at each step (you will see that `VULN` will automatically appear in the NetBIOS field).

At the end, Windows will run a requirements check. You may see yellow warning triangles (this is normal in lab environments), but a green checkmark will indicate that all checks have passed. Click `Install`.

The system will then configure Active Directory for a couple of minutes and restart automatically.

After the reboot, you will no longer log in as a local user, but as the domain administrator (you will see `VULN\Administrator`).
Now that the machine has restarted, it has officially become the Domain Controller for the `vuln.local` domain.

### Populate the DC, create misconfigurations
Since the DC is empty, to perform anything useful Accounts have to be added.
Here are presented PowerShell commands to do so:
+ Create a new account:
  + `New-ADUser -Name "Mario Rossi" -SamAccountName "mrossi" -AccountPassword (ConvertTo-SecureString "Password_of_mrossi" -AsPlainText -Force) -Enabled $true`

+ Create a new service (with a SPN):
  + `New-ADUser -Name "sql_svc" -SamAccountName "sql_svc" -AccountPassword (ConvertTo-SecureString "SQLServerPass123!" -AsPlainText -Force) -Enabled $true`
  + `setspn -A MSSQLSvc/sql01.vuln.local:1433 sql_svc`

Note: this represent an instance to a real service.

+ Create a group:
  + `New-ADGroup -Name "IT_Support" -GroupScope Global`

+ Add accounts to a group:
  + `Add-ADGroupMember -Identity "IT_Support" -Members lserra,sferrari,mrossi`

Now here the instructions to create the misconfigurations:

+ Assign to an account the "Do not require Kerberos preauthentication" attribute:
  + Press the `Windows key` on the keyboard, type `dsa.msc` and press `Enter`. (This will open the `Active Directory Users and Computers` screen).
  + In the left column, click the `Users` folder.
  + In the central list, look for the user you created earlier, right-click on them and select `Properties`.
  + Go to the tab at the top called `Account`.
  + In the central panel (`Account options`), scroll down the list until you find: `Do not require Kerberos preauthentication`
  + Check that box, click `Apply`, and then `OK`.

+ Give to a group permissions to another group (can be generalized to be done with whatever entity)
  + Open `dsa.msc`
  + Look at the top menu bar of the main `Active Directory` window (where you see `File`, `Action`, `View`, `Help`).
  + Click `View`.
  + Click `Advanced Features`. The screen will briefly refresh and you will see many more folders appear in the left column.
  + Delegate the `Helpdesk` group the ability to modify `Domain Admins`
    + Still in the `Users` folder, look for the `Domain Admins` group.
    + Right-click `Domain Admins` and select `Properties`.
    + Go to the `Security` tab at the top.
(If you do not see the `Security` tab, click View in the top menu of `Active Directory` and enable `Advanced Features`.)
    + Click `Add`, type `Helpdesk`, and click `OK`.
    + Now select the `Helpdesk` group from the list at the top, and in the permissions panel at the bottom check `Write` or `Full Control`.
    + Click `Apply` and then `OK`.

#### Some other useful configuration about passwords
To avoid problems with services, this command can be used: 
+ `Set-ADUser -Identity "sql_svc" -PasswordNeverExpires $true -ChangePasswordAtLogon $false`.

For services accounts added as this tutorial suggest is not necessary, but could be useful for services not created this way.

A policy could give troubles if changing passwords become necessary:
+  press `Windows + R` on your keyboard to open the “Run” window. Type `gpmc.msc` and press `Enter`, this will open the Group Policy Management Editor.
+  Then go to `Account Policies -> Password Policy`.
+  `Minimum password age: 0 days`.

This will alow to change passwords without time-based blocks.

### Create the target file
To create `secrets.txt`, just create a file while being in the desktop of `VULN\Administrator` and write a message to be seen when the file will be stolen.

### Saving the state of the DC
Once the desired configuration is created, is fundamental to take a snapshot of the current state of the machine, since the attack could change the starting configuration of the DC or unwanted, unknown Windows mechanisms could modify the configuration after a period of time.

By doing this the configuration desired is preserved and the machine can be restored to the saved state. It's important to easily perform multiple tries of the wanted attack.

## Introduction to the Attack
The idea of this demo was to create and attack a small organization that uses Windows Active Directory which has several misconfigurations.

### Goal
The `Administrator` has the file `secrets.txt` on his desktop, the demo is designed for obtaining it as the final goal.

### Virtual Machines
Two virtual machines were used for the demo:
+ A VM in which is installed Windows Server 2022 that will act as the Domain Controller for the domain "vuln.local", created expressly for the demo
+ A Kali Linux VM, that will be used by the Attacker

### Domain Overview
The Domain Controller (DC) is configured with the static IP address 192.168.56.10.
In order to simulate a small corporate infrastructure, the domain environment was populated with:
+ 12 user accounts: `srv_audit` + 11 `Name Surname` accounts (sAMAccountName: `nsurname`)
+ 1 service account `sql_svc` mapped to a Service Principal Name (SPN)
+ 4 groups: `RisorseUmane`, `Finanza`, `IT_Group` and `Helpdesk`.

In order to create an environment ready to be attacked, and that simulates an organization with exploitable misconfigurations:
+ The account `srv_audit` has been configured with the "Do not require Kerberos preauthentication" attribute
+ The service `sql_svc` was added to the Helpdesk group.
+ The Helpdesk group is overprivileged has Write permissions to the `Domain Admins` group.

### Threat Model
The Threat Model for the demo is that the attacker:
+ Can communicate with the DC
+ Knows the naming convention of the organization

In order to execute the attack, 2 text files were created:
  - `guess_names.txt`: targeted list of potential domain usernames.
  - `rockyou.txt`: dictionary containing possible passwords for cracking.

## Tools Utilized
+ Impacket
  - GetNPUsers.py
  - GetUserSPNs.py

+ Openwall (John the Ripper)
  - john

+ OpenLDAP
  - ldapsearch

+ bloodyAD (CravateRouge)
  - dacledit.py

+ Samba
  - net rpc
  - smbclient

+ RockYou Leak (Kali Linux Wordlists)
  - rockyou.txt

## Attack Execution
### AS-REP roasting
Using LDAP queries, `GetNPUsers.py` contacts the DC to verify whether the usernames provided within the `guess_names.txt` file match actual accounts in the target domain. Specifically, the script checks if any of these users do not require Kerberos pre-authentication. Once a matching account with this misconfiguration is identified, the script constructs and transmits an unauthenticated AS-REQ message to the DC. In response, the DC issues an AS-REP message containing the TGT (Ticket Granting Ticket) along with an encrypted segment ciphered with the user's password key. The resulting TGT hashes are saved in another text file which will be created at the moment.
In the command there is specified the format "john"; in fact once the TGT is obtained, John the Ripper (john) is used to perform password cracking.
As wordlist, `rockyou.txt` is used and the cracking is performed on the TGT in the text file previously created.
As a result the password of `srv_audit` is obtained and the AS-REP roasting is successful.

![1](images/cyberdemo1.png)

*Figure 1 — AS-REP roasting execution*

### Domain discovery
The next step is to have more information about the domain, using the credentials obtained with the previous step 2 actions were performed:
1. Discover which services are present in the domain
2. Discover the access rights on the "Domain Admins" group

#### Service discovery
Using the `ldapsearch` utility, a targeted query against the DC using the previously compromised credentials of  `srv_audit` is performed. By applying the selective filter `'(servicePrincipalName=*)'`, were shown all and only the service accounts linked to a Service Principal Name (SPN).

In the demo are present only 3 services: Active Directory Domain Services (AD DS), Kerberos Key Distribution Center (KDC), and `sql_svc`.
It has been done using the command:

`ldapsearch -x -H 'ldap://192.168.56.10' -D 'srv_audit@vuln.local' -w 'Password123!' -b 'DC=vuln,DC=local' '(servicePrincipalName=*)'`.

As a result it can be observed that `sql_svc` is in the `Helpdesk` group, indicating a misconfiguration.

![2](images/cyberdemo2.png)

*Figure 2 — Detail of the information about `sql_svc`*

#### Gathering information about the `Domain Admins` group
To obtain information about the `Domain Admins` group, the utility `dacledit.py` has been used. This tool is specifically designed to query and modify Access Control Lists (ACLs) within Active Directory, and in this case, the action performed is to read the permissions of the target group.
The command is:
`dacledit.py -action read -target 'Domain Admins' -dc-ip 192.168.56.10 vuln.local/srv_audit:'Password123!'`.

By reading the information, specifically at ACE[7] (Access Control Entry), which is the individual rule within the ACL that defines the permissions granted to a specific trustee, it can be seen that the group `Helpdesk` is present. This configuration grants the Helpdesk group full Write permissions over the `Domain Admins` group, exposing a critical security misconfiguration.

![3](images/cyberdemo3.png)

*Figure 3 — Detail of the ACE[7]*

#### Discovery results
By exposing these two misconfigurations, it becomes clear that `sql_svc` is crucial for the continuation of the attack.

### Kerberoasting
For performing Kerberoasting, the `GetUserSPNs.py` utility is used, it does a initial scan to see which accounts have an SPN and then, using the `-request` parameter in the command, it requests the Kerberos TGS for the services it found; to find them he script searches in the Active Directory via LDAP to locate accounts linked to a SPN.
As for AS-REP roasting: the resulting TGS hashes are saved in a text file which will be created at the moment, the john format is specified.
After the TGS is obtained, john is used again to perform password cracking.

![4](images/cyberdemo4.png)

*Figure 4 — Kerberoasting execution*


### Domain Admins Abuse
Since the Kerberoasting was successful, now is possible to take advantage of the privileges that `sql_svc` has, being in the `Helpdesk` group.

![5](images/cyberdemo5.png)

*Figure 5 — Privilege escalation*

The command utilizes the `net` utility via the Remote Procedure Call (RPC) protocol. RPC is a standard network communication protocol used by Windows environments that allows a client, in this case the Kali Linux machine, to request the execution of a specific administrative action directly on a remote server, the DC.

By establishing an RPC connection authenticated with the cracked credentials of `sql_svc`, the command invokes the remote procedure required to modify group memberships on the DC. Since `sql_svc` inherits the necessary write permissions from the `Helpdesk` group, the DC accepts the request and inserts `sql_svc` in the `Domain Admins` group.

### Data Exfiltration

After performing the privilege escalation, there is the possibility to look what the `Administrator` has on his desktop and subsequently acquiring it, specifically the file `secrets.txt`.
To achieve this, the Server Message Block (SMB) protocol is utilized to interact directly with the administrative network shares of the DC.

First, an enumeration of the `Administrator` desktop directory is performed.
The connection exploits the administrative share `C$`. Access to this specific share is strictly restricted by Windows to members of the local `Administrators` or `Domain Admins` groups. Since the `sql_svc` account was successfully added to the `Domain Admins` group in the previous step, the DC grants full read and write access to the entire file system.
The output of the directory listing confirms the presence of the targeted file, `secrets.txt`.
Then, by using the `get` command followed by a `-`, the content of `secrets.txt` is retrieved via the SMB channel and printed directly into the standard output of the terminal.
The message "All secrets are here! Don't look!" is the actual content of the file `secrets.txt`.


![6](images/cyberdemo6.png)

*Figure 6 — Data exfiltration*

## Conclusions
As said in the introduction, the demo proposed tries to simulate an attack to a small organization which use Windows Active Directory, since the AD has been configured expressly for the realization of the demo without the use of pre-configured AD, the misconfigurations are product of the imagination.

The environment was created to show different steps of an attack starting with little information about the domain: using a list of possible names for the AS-REP roasting, discovering the services present and the access rights on the `Domain Admins` group, and searching in the `Administrator` desktop to find the `secrets.txt` file.

The idea and structure of both the attack and the Active Directory is authentic; for the realisation, the usage of Gemini AI has been crucial for the Windows Server setup and creation of the misconfigured AD.

## Bibliography
* **[1] Windows Server 2022**
  * URL: [https://www.microsoft.com/it-it/evalcenter/download-windows-server-2022](https://www.microsoft.com/it-it/evalcenter/download-windows-server-2022)

* **[2] Gemini**
  * URL: [https://gemini.google.com/app?hl=it](https://gemini.google.com/app?hl=it)

* **[3] Impacket**
  * URL: [https://github.com/fortra/impacket](https://github.com/fortra/impacket)

* **[4] John the Ripper**
  * URL: [https://github.com/openwall/john](https://github.com/openwall/john)

* **[5] AS-REP roasting**
* URL: [https://www.youtube.com/watch?v=wA9w8t1fRWo](https://www.youtube.com/watch?v=wA9w8t1fRWo)

* **[6] Kerberoasting**
* URL: [https://www.youtube.com/watch?v=tRCvagjqx3c](https://www.youtube.com/watch?v=tRCvagjqx3c)

* **[7] RockYou Leak**
* URL: [https://www.kali.org/tools/wordlists/](https://www.kali.org/tools/wordlists/)

* **[8] Discovery**
* URL: [https://github.com/ShubhamDubeyy/Active-Directory-Workbook#module-4-enumerating-users-groups--privileged-accounts](https://github.com/ShubhamDubeyy/Active-Directory-Workbook#module-4-enumerating-users-groups--privileged-accounts)
* URL: [https://github.com/fortra/impacket/blob/master/examples/dacledit.py](https://github.com/fortra/impacket/blob/master/examples/dacledit.py)

* **[9] Privilege escalation**
* URL: [https://www.thehacker.recipes/ad/movement/dacl/addmember](https://www.thehacker.recipes/ad/movement/dacl/addmember)

* **[10] Data exfiltration**
* URL: [https://hackviser.com/tactics/tools/smbclient](https://hackviser.com/tactics/tools/smbclient)
* URL: [https://www.samba.org/samba/docs/current/man-html/smbclient.1.html](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)




