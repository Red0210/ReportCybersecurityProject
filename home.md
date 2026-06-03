# Active Directory Domain Compromise via AS-REP roasting and Kerberosting

## Introduction to the attack
The idea of this demo was to create and attack a small organization that uses Windows Active Directory which has credible/possible misconfiguration.
### Virtual machines utilized
For the demo has been used 2 VM (Virtual Machines)
+ A VM in which is installed Windows Server 2022 that will act as the Domain Controller for the domain "vuln.local", created expressly for the demo
+ A Kali Linux VM, that will be used by the Attacker
### Domain Overview
The Domain Controller (DC) is configured with the static IP address 192.168.56.10.
In order to to simulate a small corporate infrastructure, the domain environment was populated with:
+ 12 user accounts: `srv_audit` + 11 `name surname` accounts
+ 1 service account instance of Microsoft SQL Server, chosen for realism;
+ 4 groups which accounts were assigned to: `RisorseUmane`, `Finanza`, `IT_Group` and `Helpdesk`.
In order to create an enviroment ready to be attacked, and that simulates an organization with exploitable misconfigurations:
+ The account `srv_audit` has been configured with the "Do not require Kerberos preauthentication" attribute
+ The service `sql_svc` was added to the Helpdesk group.
+ The Helpdesk group is overprivileged has Write permissions to the `Domain Admins` group.
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
In the command there is specified the format "john", in fact once the TGT is obtained, John the Ripper (john) is used to perform password cracking.
As wordlist, `rockyou.txt` is used and the cracking is performed on the TGT in the textual file previously created.
As a result the password of `srv_audit` is obtained and the AS-REP roasting has been succesful.

![AS-REP execution](images/cyberdemo1.png)

### Domain discovery
The next step is to have more informations about the domain, using the credentials obtained with the previous step 2 actions were performed:
1. Discover which services are present in the domain
2. Discover the access rights on the "Domain Admins" group

#### Service discovery
Using the `ldapsearch` utility, a targeted query against the DC using the previously compromised credentials of  `srv_audit` is performed. By applying the selective filter `'(servicePrincipalName=*)'`, were shown all and only the service accounts linked to a Service Principal Name (SPN).

In the demo are present only 3 services: Active Directory Domain Services (AD DS), Kerberos Key Distribution Center (KDC), and `sql_svc`.
It has been done using the command: `ldapsearch -x -H 'ldap://192.168.56.10' -D 'srv_audit@vuln.local' -w 'Password123!' -b 'DC=vuln,DC=local' '(servicePrincipalName=*)'`.
As a result it is possible to see that `sql_svc` is in the `Helpdesk` group, indicating a misconfiguration.

![AS-REP execution](images/cyberdemo1.png)

#### Gathering informations about the `Domain Admins` group
To obtain information about the `Domain Admins` group, the utility `dacledit.py` has been used. This tool is specifically designed to query and modify Access Control Lists (ACLs) within Active Directory, and in this case, the action performed is to read the permissions of the target group.
The command is:
`dacledit.py -action read -target 'Domain Admins' -dc-ip 192.168.56.10 vuln.local/srv_audit:'Password123!'`

By reading the information, specifically at ACE[7] (Access Control Entry), which is the individual rule within the ACL that defines the permissions granted to a specific trustee, it can be seen that the group Helpdesk is present. This configuration grants the Helpdesk group full Write permissions over the Domain Admins group, exposing a critical security misconfiguration.

![AS-REP execution](images/cyberdemo1.png)

#### Discovery results
By exposing these two misconfigurations, it becomes clear that `sql_svc` is crucial for the continuation of the attack







> Note: Example page content from [GetGrav.org](https://learn.getgrav.org/17/content/markdown), included to demonstrate the portability of Markdown-based content

[^1]: [Markdown - John Gruber](https://daringfireball.net/projects/markdown/)
