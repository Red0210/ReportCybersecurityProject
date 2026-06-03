# Active Directory Domain Compromise via AS-REP roasting and Kerberosting

## Introduction to the attack
The idea of this demo was to create and attack a small organization that uses Windows Active Directory which has credible/possible misconfiguration.
### Virtual machines utilized
For the demo has been used 2 VM (Virtual Machines)
+ A VM in which is installed Windows Server 2022 that will act as the Domain Controller for the domain "vuln.local", created expressly for the demo
+ A Kali Linux VM, that will be used by the Attacker
### Domain Overview
The Domain Controller (DC) is configured with the static IP address 192.168.56.10.
In order to to simulate a small corporate infrastructure, the domain environment was populated with: 12 user accounts and a service account; accounts were assigned to groups.
In order to create an enviroment ready to be attacked, and that simulates an organization with exploitable misconfigurations:
+ The account `srv_audit` has been configured with the "Do not require Kerberos preauthentication" attribute
+ The service `sql_svc` was added to the Helpdesk group.
+ The Helpdesk group is overprivileged has Write permissions to the Domain Admin group.
### Threat Model
The threat model for the demo is the following:
+ Can communicate with the DC
+ Knows the naming convention of the organization
In order to execute the attack, 2 textual files were created:
+ `guess_names.txt`: targeted list of potential domain usernames.
+ `rockyou.txt`: dictionary containing possible passwords for cracking.
## Tools used
From the toolkit Impacket:
+ `GetNPUsers.py` & `GetUserSPNs.py`: used respectively for AS-REP roasting and Kerberoasting

## Execution of the attack
### AS-REP roasting
Using LDAP queries, `GetNPUsers.py` contacts the DC to verify whether the usernames provided within the `guess_names.txt` file match actual accounts in the target domain. Specifically, the script checks if any of these users do not require Kerberos pre-authentication. Once a matching account with this misconfiguration is identified, the script constructs and transmits an unauthenticated AS-REQ message to the DC. In response, the DC issues an AS-REP message containing the TGT (Ticket Granting Ticket) along with an encrypted segment ciphered with the user's password key. The resulting TGT hashes are saved in another textual file which will be created at the moment.









> Note: Example page content from [GetGrav.org](https://learn.getgrav.org/17/content/markdown), included to demonstrate the portability of Markdown-based content

[^1]: [Markdown - John Gruber](https://daringfireball.net/projects/markdown/)
