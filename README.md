[README1.md](https://github.com/user-attachments/files/28369309/README1.md)
# Active Directory Home Lab

A hands-on home lab built inside Oracle VirtualBox, walking through the full setup of a Windows Server 2022 environment — from bare server to a functioning domain with DNS, DHCP, NAT routing, and hundreds of users created automatically via PowerShell. This lab simulates what a real small corporate network looks like from the ground up.

---

## What This Lab Covers

- Setting up a Windows Server VM with two network adapters, one internet-facing and one internal
- Assigning a static IP to the internal network adapter
- Installing and promoting Active Directory Domain Services (AD DS)
- Creating a new forest and root domain
- Creating Organizational Units (OUs) and a dedicated domain admin account
- Installing Remote Access and configuring NAT so internal clients can reach the internet through the server
- Installing DHCP, configuring a scope, and authorizing the DHCP server in Active Directory
- Setting up a Windows 10 client VM, joining it to the domain, and verifying connectivity
- Bulk-creating domain users from a names list using a PowerShell script
- Logging into the client as one of the newly created users to verify everything works end to end

---

## Lab Environment

| Component | Details |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Server OS | Windows Server 2022 |
| Client OS | Windows 10 |
| Network Adapters | Ethernet (internet-facing) + Internal Network (lab clients) |
| Domain | danielledomain.com (lab environment only) |

---

## Step-by-Step Walkthrough

### 1. Installing Windows Server and Configuring Network Adapters

I installed Windows Server 2022 in VirtualBox and configured it with two network adapters. The first, Ethernet, connects to the internet through VirtualBox's NAT. The second, Internal Network, is what lab client machines will connect to, an isolated network that only exists inside VirtualBox.

![Network Connections](Screenshot%20%28307%29.png)

From Network Connections I identified both adapters, then opened the IPv4 properties on the Internal Network adapter and assigned it a static IP address. The server needs a fixed address because it will act as the DNS server and default gateway for everything on the internal network.

![Static IP Configuration](Screenshot%20%28308%29.png)

---

### 2. Installing Active Directory Domain Services

From Server Manager, I went to Manage > Add Roles and Features and selected Active Directory Domain Services. The wizard prompted me to also install the required management tools, including Group Policy Management, AD DS and AD LDS Tools, Active Directory Administrative Center, and the Active Directory module for PowerShell.

![Add Features Prompt](Screenshot%20%28310%29.png)

![Installation Progress](Screenshot%20%28311%29.png)

Once installation finished, a yellow notification flag appeared in Server Manager prompting me to promote the server to a domain controller.

---

### 3. Promoting the Server to a Domain Controller

I clicked the notification and launched the Active Directory Domain Services Configuration Wizard. I selected "Add a new forest" and entered `danielledomain.com` as the root domain name.

![Deployment Configuration](Screenshot%20%28314%29.png)

On the DNS Options page there was a warning that a DNS delegation couldn't be created. That's expected for a brand new forest with no parent DNS zone above it, so I left "Create DNS delegation" unchecked and moved on.

![DNS Options](Screenshot%20%28315%29.png)

The prerequisites check passed and the server rebooted as a domain controller.

---

### 4. Creating an Organizational Unit and Admin Account

After the reboot I opened Active Directory Users and Computers from the Tools menu. Rather than using the built-in Administrator account for day-to-day work, I created a dedicated admin account.

First I right-clicked the domain, went to New > Organizational Unit, and created one called ADMINS to keep elevated accounts separate from regular users.

![New OU Menu](Screenshot%20%28317%29.png)

![Creating ADMINS OU](Screenshot%20%28318%29.png)

Inside the ADMINS OU I created a new user with an `a-` prefix on the logon name, a common naming convention to make admin accounts easy to identify at a glance. I then opened their properties and added them to the Domain Admins group.

![Creating Admin User](Screenshot%20%28319%29.png)

![Adding to Domain Admins](Screenshot%20%28320%29.png)

To confirm everything worked, I signed out and logged back in with the new admin account. The login screen showed it authenticating against the domain.

![Domain Login](Screenshot%20%28321%29.png)

---

### 5. Adding Remote Access and Configuring NAT

The next step was to add the Remote Access role and configure NAT, which allows internal clients to access the internet through the server.

From Server Manager I added the Remote Access role and selected Routing as the role service.

![Remote Access Role](Screenshot%20%28322%29.png)

![Routing Features](Screenshot%20%28323%29.png)

After installation I opened Routing and Remote Access from the Tools menu, right-clicked the server, and chose "Configure and Enable Routing and Remote Access."

![Routing and Remote Access](Screenshot%20%28324%29.png)

I selected Network Address Translation (NAT) as the configuration type, then picked the Ethernet adapter as the public interface.

![NAT Configuration](Screenshot%20%28325%29.png)

![Selecting Public Interface](Screenshot%20%28326%29.png)

Once the wizard finished, routing was configured on the server.

![Routing Configured](Screenshot%20%28327%29.png)

---

### 6. Installing and Configuring DHCP

I added the DHCP Server role so that internal clients would automatically receive their IP address, subnet mask, default gateway, and DNS server rather than needing everything entered manually.

![DHCP Server Role](Screenshot%20%28328%29.png)

During scope setup I configured the Domain Name and DNS Servers step, pointing clients to the domain name and the server's internal IP as their DNS server.

![DHCP DNS Settings](Screenshot%20%28331%29.png)

After creating the scope, I ran the DHCP Post-Install configuration wizard and authorized the DHCP server in Active Directory using my domain admin credentials. DHCP servers have to be authorized before Active Directory will allow them to hand out leases.

![DHCP Authorization](Screenshot%20%28341%29.png)

---

### 7. Bulk User Creation with PowerShell

With the infrastructure in place, I used a PowerShell script to populate the domain with users from a plain text list of names (`names.txt`).

![names.txt file](Screenshot%20%28334%29.png)

I opened PowerShell ISE as Administrator and loaded the `1_CREATE_USERS` script.

![Opening the script](Screenshot%20%28335%29.png)

The script loops through each name, builds a username from the first initial and last name, assigns a default password, and creates the account in Active Directory under the `_Users` OU. Running it you can watch the accounts being created in real time.

![Script running](Screenshot%20%28339%29.png)

> Script credit: [Josh Madakor](https://github.com/joshmadakor1/AD_PS). The names list and PowerShell scripts in this repo are his work.

---

### 8. Setting Up the Windows 10 Client and Joining the Domain

I spun up a Windows 10 VM set to use the Internal Network adapter, then ran `ipconfig /renew` to pull a DHCP lease from the server. The output confirmed it received an address in the right range with the domain DNS suffix and the server set as the default gateway.

![ipconfig /renew output](Screenshot%20%28343%29.png)

I then pinged the domain by name, which resolved and came back with 0% packet loss, confirming DNS and routing were both working.

![Ping test](Screenshot%20%28344%29.png)

To join the domain I went to System Properties > Computer Name > Change, selected Domain, typed in `danielledomain.com`, and entered my admin credentials. The confirmation message came back: "Welcome to the danielledomain.com domain."

![Joining the domain](Screenshot%20%28346%29.png)

Back on the server, the DHCP console showed the client's lease had appeared under Address Leases, confirming the server had issued an IP to `CLIENT1`.

![DHCP Lease](Screenshot%20%28347%29.png)

---

### 9. Logging In as a Domain User

As a final check I logged into the Windows 10 client as `arettig`, one of the accounts created by the PowerShell script. The machine authenticated against the domain and loaded a fresh user profile, confirming the whole lab is working end to end.

![Domain user login](Screenshot%20%28349%29.png)

---

## Key Concepts Practiced

- Active Directory structure: forests, domains, OUs, users, and groups
- Why the server needs a static IP rather than a DHCP-assigned one
- How NAT allows multiple internal clients to share one public IP to reach the internet
- How DHCP automates IP assignment and passes DNS and gateway info to clients
- Least-privilege principle: dedicated admin account rather than using built-in Administrator
- How DNS ties the whole environment together, name resolution is what makes domain joining work
- Automating repetitive AD tasks with PowerShell

---

## Script Credit

User creation script and names list by [Josh Madakor](https://github.com/joshmadakor1/AD_PS).

---

## Notes

- All screenshots containing IP addresses show internal private addresses within the VirtualBox lab environment only
- No real credentials or sensitive data are stored in this repository
