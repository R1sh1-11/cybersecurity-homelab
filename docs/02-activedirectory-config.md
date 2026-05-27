# 02 - Active Directory Setup & Client Configuration

## Overview
This phase transforms the bare Windows Server 2022 VM into a functioning Active Directory 
environment with a domain controller, intentionally vulnerable users, and a domain-joined 
client machine. This is the core infrastructure that all future attacks and detections will 
run against (hopefully with no errors).

---

## Part 1 — Installing Active Directory Domain Services

### What is AD DS?
Active Directory Domain Services is the Windows role that enables a server to manage 
users, computers, and authentication for an entire network. Without this role installed, 
the server is just a regular Windows machine. Installing AD DS gives it the software it 
needs to become a Domain Controller.

### Steps
1. Opened Server Manager → clicked **Add Roles and Features**
2. Selected **Role-based installation**
3. Checked **Active Directory Domain Services**
4. Clicked through and installed
5. After install, clicked the yellow flag → **Promote this server to a domain controller**

### Configuration Choices
- **New forest** — created a brand new AD environment from scratch
- **Root domain:** `lab.local` — internal domain name, doesn't exist on the public internet
- **NetBIOS name:** `LAB` — short name used in `LAB\username` format
- **DSRM password** — emergency recovery password set in case AD breaks

### Why These Choices Matter
A forest is the top-level container for the entire AD environment. Every company has one. 
`lab.local` uses `.local` to signal it's internal only. The NetBIOS name `LAB` is what 
you'll see prefixed on every account: `LAB\jbond`, `LAB\Administrator`, etc.

![AD DS and DNS Roles](../screenshots/ad_usersandcomputers.png)
*Server Manager showing AD DS and DNS roles active — Tools menu open showing Active Directory Users and Computers available*

![lab.local Domain](../screenshots/lab.local.png)
*Active Directory Users and Computers showing lab.local domain successfully created*

![NetBIOS Configuration](../screenshots/netbios_domainname.png)
*NetBIOS domain name set to LAB*

---

## Part 2 — Creating AD Users

### Why We Create Users
Real corporate networks have hundreds of user accounts with varying levels of access. 
Attackers enumerate these accounts looking for weak passwords or misconfigured 
permissions. I am intentionally creating vulnerable accounts to practice real attack 
techniques against them.

### Users Created
| Username | Full Name | Password | Notes |
|---|---|---|---|
| jbond | James Bond | Password123! | Weak password — password spray target |
| pparker | Peter Parker | spidermanrocks12! | Weak password |
| sconnor | Sarah Connor | John@456! | Weak password |
| svcbackup | Backup Services | Backingup123! | Service account — Kerberoasting target |

All accounts set to **Password never expires**.

### Registering an SPN on svcbackup
A Service Principal Name (SPN) tells Kerberos that an account runs a specific service. 
Any domain user can request a Kerberos ticket for an account with an SPN. That ticket is 
encrypted with the account's password hash, which means an attacker can grab it offline and 
crack it without ever touching the account directly. There is no lockout and no alerts. All of this equals kerberoasting.

```powershell
setspn -a HTTP/labbackup.lab.local svcbackup
```

### Screenshots
![AD Users List](../screenshots/createdusersinad.png)
*All users created in Active Directory Users and Computers — jbond, pparker, sconnor, svcbackup visible*

![SPN Registration](../screenshots/setspnobject.png)
*SPN successfully registered on svcbackup — "Updated object" confirms it worked*

---

## Part 3 — Network Configuration

### Why Internal Network?
Both VMs were originally on NAT, meaning they had separate connections to the internet 
and couldn't see each other. Switching to VirtualBox Internal Network (`labnetwork`) 
creates a private virtual switch between them, which is completely isolated from the internet.

### Static IP Assignment
| Machine | IP Address | Subnet Mask | DNS |
|---|---|---|---|
| Domain Controller (DC) | 192.168.10.10 | 255.255.255.0 | 127.0.0.1 (itself) |
| Windows Client | 192.168.10.11 | 255.255.255.0 | 192.168.10.10 (DC) |
| Kali (not yet, will be implemented next) | 192.168.10.12 | 255.255.255.0 | 192.168.10.10 (DC) |

**Why the DC points DNS at itself (`127.0.0.1`):** The DC runs its own DNS server. 
`127.0.0.1` is the loopback address, which means "ask yourself." Since the DC IS the DNS 
server for `lab.local`, it resolves its own domain internally.

**Why the client points DNS at the DC (`192.168.10.10`):** The client has no DNS server 
of its own. When it needs to find `lab.local` to join the domain, it asks the DC. Without 
this setting, the client would search the internet for `lab.local`, find nothing, and fail.

### Screenshots
![DC Static IP](../screenshots/localnetconfig.png)
*Domain Controller configured with static IP 192.168.10.10*

![Client Static IP](../screenshots/clientnetconfig.png)
*Client configured with static IP 192.168.10.11 and DNS pointing to DC*

---

## Part 4 — Windows Client Setup

### Why a Separate Client VM?
The Domain Controller is the brain of the network. In real environments, employees don't 
log directly into the DC, but they use workstations (laptops, desktops) that are joined to 
the domain. Attacks like Pass-the-Hash typically start by compromising a workstation, 
then using stolen credentials to move laterally to the DC. You need both machines to 
simulate that attack chain.

### What Went Wrong — Cloning the DC
The first attempt was to clone the existing Windows Server 2022 VM to save time. This failed 
because cloning a Domain Controller creates a duplicate machine with the same AD identity. 
When the clone tried to log in, AD rejected it with a trust relationship error. It saw 
two machines claiming to be the same DC, which is impossible.

**Fix:** Deleted the clone and did a fresh Windows Server 2022 install on a new VM. 
This time, I only installed the OS: no AD DS, no domain promotion. Just a plain
Windows machine that could join the domain as a workstation (this time, the fix didn't take too long thankfully).

### Joining the Domain
```powershell
# Run on the client VM
Add-Computer -DomainName "lab.local" -Credential LAB\Administrator -Restart
```

This command contacts the DC via DNS, authenticates with Domain Administrator credentials, 
registers the client as a computer object in AD, and reboots to apply the change.

### Verifying the Domain Join
```powershell
whoami
```

### Screenshots
![Ping Success](../screenshots/pingsuccess.png)
*Client successfully pinging DC at 192.168.10.10 — 0% packet loss confirms network connectivity*

![whoami Output](../screenshots/clientadmin.png)
*whoami returns lab\administrator which confirms the client is authenticated against the domain, not locally*

---

## Part 5 — Problems & Fixes

| Problem | Root Cause | Fix |
|---|---|---|
| Cloned VM couldn't log in — trust relationship error | Cloning a DC creates duplicate AD identity | Fresh OS install instead of clone |
| DC showed IP 169.254.x.x after reboot | Static IP settings didn't persist after reboot | Re-entered static IP manually |
| Client couldn't ping DC | DC had lost its static IP | Fixed DC IP, ping succeeded immediately |
| Client ping showed replies from 192.168.10.11 (itself) | DC was offline | Booted DC first, then retried ping |

---

## What I Learned
- You cannot clone a Domain Controller — AD treats it as a duplicate identity and rejects it
- DNS is the foundation of AD — the entire domain join process depends on DNS resolving `lab.local`
- Static IPs are non-negotiable on a DC — if the IP changes, nothing on the network can find it
- The `169.254.x.x` address range means Windows gave up trying to get an IP and assigned itself one automatically (APIPA) — always a sign something is wrong with network config
- `whoami` returning `lab\administrator` is proof of domain authentication working end-to-end
- Perhaps the most important takeaway - how to change the size of my Virtual Machine's screen 1.5 times its initial size

---

## Current Lab State
- ✅ Domain Controller running `lab.local` at `192.168.10.10`
- ✅ AD users created with weak passwords (jbond, pparker, sconnor, svcbackup)
- ✅ svcbackup registered with SPN — ready for Kerberoasting
- ✅ Windows Client joined to domain at `192.168.10.11`
- ✅ Both VMs on isolated `labnetwork` internal network
- ✅ Snapshot taken: "Lab Complete - Pre-Attack"
