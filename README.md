# Windows Active Directory and Security Homelab

I've worked with Active Directory in enterprise IT, but I had never built a domain myself from nothing. This lab was my way of doing that: one domain controller, one Windows 11 workstation, a set of Group Policy security controls, and a test for every control to prove it actually works.

**Tech used:** VirtualBox 7.2, Ubuntu, Windows Server 2025, Windows 11 Enterprise, Active Directory, DNS, Group Policy, PowerShell, Windows Firewall, UEFI Secure Boot, TPM 2.0

## Architecture

```mermaid
flowchart TB
    NET((Internet))
    subgraph HOST["Host: HP ENVY x360, Ubuntu, 16 GB RAM, Secure Boot on"]
        subgraph LABNET["VirtualBox NAT Network 'LabNet' 192.168.50.0/24, DHCP off"]
            GW["NAT Gateway<br/>192.168.50.1"]
            DC["DC01<br/>Windows Server 2025<br/>AD DS + DNS<br/>192.168.50.10"]
            CL["CLIENT01<br/>Windows 11 Enterprise<br/>Domain joined<br/>192.168.50.20"]
        end
    end
    NET --- GW
    GW --- DC
    GW --- CL
    CL -- "DNS, Kerberos, LDAP, Group Policy" --> DC
    DC -- "DNS forwarders<br/>1.1.1.1 / 8.8.8.8" --> GW
```

| Machine | OS | Role | IP | vCPU / RAM / Disk |
|---|---|---|---|---|
| Host | Ubuntu on an HP ENVY x360 (16 GB) | Runs VirtualBox | N/A | N/A |
| DC01 | Windows Server 2025 Standard Evaluation (Desktop Experience) | Domain controller, DNS, Global Catalog | 192.168.50.10 | 2 / 3 GB / 60 GB |
| CLIENT01 | Windows 11 Enterprise Evaluation | Workstation joined to the domain (UEFI, TPM 2.0, Secure Boot) | 192.168.50.20 | 2 / 4 GB / 64 GB |

Both VMs sit on an isolated VirtualBox NAT network. I turned DHCP off and used static IPs so the client could never pick up the wrong DNS server. The domain is `lab.local` (NetBIOS name `LAB`).

For DNS, CLIENT01 only uses DC01. DC01 points to itself first and loopback second, and forwards anything it can't answer (like microsoft.com) to 1.1.1.1 and 8.8.8.8.

## How I Built It

1. Installed VirtualBox on Ubuntu from Oracle's repository. My laptop has Secure Boot on, so the VirtualBox kernel modules were blocked at first. I enrolled a Machine Owner Key with `mokutil` instead of turning Secure Boot off.
2. Created the LabNet NAT network with `VBoxManage` and disabled DHCP.
3. Built both VMs from the command line with `VBoxManage`. The Windows 11 VM got UEFI, TPM 2.0 and Secure Boot since Windows 11 requires them.
4. Installed Windows Server 2025, then used PowerShell to set the static IP, hostname and time zone.
5. Installed AD DS and promoted DC01 to the first domain controller in a new forest, `lab.local`, with DNS on the same server.
6. Checked that the SRV record clients use to find the DC (`_ldap._tcp.dc._msdcs.lab.local`) resolved, and set up DNS forwarders for internet names.
7. Created OUs, test users, security groups and a separate admin account for myself. I also ran `redircmp` so new computers land in my own OU instead of the default Computers container.
8. Installed Windows 11, pointed its DNS at DC01, joined it to the domain with my admin account, and logged in as a regular domain user.
9. Set up Group Policy for passwords, account lockout, screen lock, the firewall and local admin rights.
10. Tested every control and saved screenshots as evidence.

## Active Directory Layout

- **LAB-Users**
  - **IT:** John Smith (`jsmith`, member of IT-Helpdesk), Sabesen Admin (`adm.sabesen`, member of Domain Admins)
  - **Staff:** Alice Wong (`awong`) and Maya Patel (`mpatel`), both members of Staff
- **LAB-Computers:** CLIENT01
- **LAB-Groups:** IT-Helpdesk and Staff (global security groups)
- **Domain Controllers:** DC01

I do admin work with `adm.sabesen` rather than the built in Administrator account, so actions are tied to a named person in the logs.

## Security Controls

| Control | Where it's set | Setting |
|---|---|---|
| Password policy | Default Domain Policy | 12 characters minimum, complexity on, 24 password history, 90 day max age, 1 day min age |
| Account lockout | Default Domain Policy | Lock after 10 bad attempts for 15 minutes, counter resets after 15 minutes |
| Screen lock | LAB-Workstation-Security GPO on LAB-Computers | Locks after 300 seconds idle |
| Windows Firewall | LAB-Workstation-Security GPO on LAB-Computers | On for all three profiles, inbound blocked, outbound allowed, enforced by policy |
| Local admin rights | LAB-Workstation-Security GPO on LAB-Computers | Only the IT-Helpdesk group is added to local Administrators, so Staff users stay standard users |
| Separate admin account | Active Directory | `adm.sabesen` used for privileged work |
| Platform security | CLIENT01 and the host | UEFI, TPM 2.0 and Secure Boot on the client, Secure Boot left on for the host |

A few notes on these choices:

Password and lockout settings for domain accounts only work in a GPO linked to the whole domain, which is why they're in the Default Domain Policy and not in the workstation GPO.

I went with a lockout threshold of 10 because it's Microsoft's current default. It still stops password guessing, especially with a 12 character minimum, and it causes fewer accidental lockouts. CIS recommends 5 or lower if you want to be stricter.

NIST SP 800-63B actually recommends against forcing password changes on a schedule unless there's a sign of compromise. I set 90 days anyway because a lot of organizations still require it.

## Testing

| # | Control | What I did | Expected | Result | Evidence |
|---|---|---|---|---|---|
| 1 | GPO delivery | Ran `gpresult /r /scope computer` on CLIENT01 | Both GPOs listed as applied | Pass | 20-gpresult-client.png |
| 2 | Firewall | Checked `Get-NetFirewallProfile -PolicyStore ActiveStore`, then tried ping and port 3389 from DC01 | All profiles on with inbound blocked, network type DomainAuthenticated, ping and RDP blocked | Pass | 18-test-firewall-domain-profile.png, 18b-firewall-blocks-inbound.png |
| 3 | Screen lock | Checked `InactivityTimeoutSecs` in the registry, then left the VM idle | Locks after 5 minutes | Pass | 17-test-screen-lock.png |
| 4 | Password policy | Tried to reset awong's password to a 10 character password with no symbol | Rejected | Pass | 15c-test-weak-password-rejected.png |
| 5 | Least privilege | Logged in as awong (Staff) and opened Terminal as admin | UAC asks for admin credentials | Pass | 19-test-least-privilege.png |
| 6 | Account lockout | Entered 10 wrong passwords for `LAB\mpatel` | Account locks, Event 4740 logged on DC01, I unlock it with `Unlock-ADAccount` | Pass | 16-test-account-lockout.png, 16b-lockout-event-and-unlock.png |
| 7 | Domain login | Logged in as `LAB\jsmith`, ran `whoami`, `$env:LOGONSERVER` and `nltest /dsgetdc:lab.local` | Shows lab\jsmith, logon server DC01, DC found | Pass | 12-domain-user-login-whoami.png |
| 8 | DNS and DC discovery | SRV lookup, LDAP port 389 test and an internet name lookup from CLIENT01 | All succeed | Pass | 13-client-dns-dc-tests.png |

## Problems I Ran Into

**VirtualBox wouldn't start its kernel modules.** Running `vboxconfig` failed with `modprobe vboxdrv failed`. Secure Boot was rejecting the modules because they were signed with a key my firmware didn't trust yet. I imported the key with `mokutil --import`, enrolled it in MOK Manager on reboot, and rebuilt the modules. Secure Boot stayed on the whole time.

**DC01's DNS only pointed at loopback.** After the promotion, its DNS client was set to just 127.0.0.1. I changed it to its own IP first and 127.0.0.1 second, which is the usual setup for a domain controller.

**The Windows 11 install froze on a black screen.** The VM said it was running, but there was no picture and the virtual disk had stopped growing. `free -h` showed only about 14 GiB usable on the host, and it was using swap with both VMs set to 4 GB. I shut DC01 down while Windows 11 finished installing, switched the client's graphics controller to VMSVGA, and dropped DC01 to 3 GB so both VMs could run together.

**`gpupdate` failed with "lack of network connectivity to a domain controller."** DC01 was off. I could still log in as my admin account because Windows caches domain credentials, but Group Policy needs to actually reach a DC. I started DC01, confirmed port 389 was reachable with `Test-NetConnection`, and ran `gpupdate /force` again.

**My laptop has no Right Ctrl key**, which is VirtualBox's default Host key. I remapped it to Right Alt.

## What I Learned

DNS really is the foundation of Active Directory. Clients find the domain controller through SRV records, so one wrong DNS setting breaks domain joins, logins and Group Policy all at once.

Where you link a GPO matters. Domain password policies only work at the domain level, and the default Computers container can't have GPOs linked to it at all, which is why I redirected new computers to my own OU.

Setting a policy and having it work aren't the same thing. Running `gpresult`, checking the firewall profile and finding Event 4740 in the logs is how I actually knew each control was in effect.

Giving local admin to a group like IT-Helpdesk is much cleaner than adding people one at a time, and it's easy to audit later.

Some problems that look like one thing are really another. The black screen looked like a graphics issue, but the real cause was the host running out of memory.

You don't have to turn off a security feature to get something working. Enrolling a key let me keep Secure Boot on.

Snapshots need to stay in sync. If I restored only one VM to an older snapshot, the machine account password would no longer match, and I'd get the classic "trust relationship failed" error.

## What I'd Add Next

- Windows LAPS so each machine gets its own rotating local admin password
- A fine grained password policy with stricter rules for admin accounts
- BitLocker through Group Policy, with recovery keys stored in AD
- Joining my Linux machine to the domain with realmd and SSSD
- A file share with group based permissions
- Centralized logging with alerts on events like 4740 (lockout) and 4625 (failed logon)
- Separate admin accounts for workstations and for the domain

## Screenshots

Everything is in [`/screenshots`](./screenshots), numbered in the order I built things.

| File | Shows |
|---|---|
| 01-vbox-modules-loaded | VirtualBox kernel modules loaded after the MOK fix |
| 02-labnet-natnetwork | Lab network with DHCP off |
| 04-dc01-server-manager | Windows Server 2025 installed |
| 05-dc01-static-ip-hostname | Static IP and hostname |
| 06-dc01-promotion-complete | Domain created, DC and SRV record verified |
| 07-dns-manager-lab-local | DNS zone and records created by AD |
| 08-aduc-ou-structure | OU structure |
| 09-users-and-groups | Group membership |
| 10b-client01-secureboot-tpm | Windows reporting UEFI and Secure Boot on |
| 11-client01-domain-joined | CLIENT01 joined to lab.local |
| 12-domain-user-login-whoami | Domain user login checked by DC01 |
| 13-client-dns-dc-tests | DNS and DC lookups from the client |
| 13b-client01-in-lab-computers-ou | CLIENT01 in the LAB-Computers OU |
| 14-gpmc-gpos-linked | Workstation GPO linked to LAB-Computers |
| 15-password-lockout-policy | Password and lockout settings in effect |
| 15b-client01-gpupdate-success | Group Policy applied on CLIENT01 |
| 15c-test-weak-password-rejected | Weak password rejected |
| 16-test-account-lockout | Lockout message on the client |
| 16b-lockout-event-and-unlock | Event 4740 and unlocking the account |
| 17-test-screen-lock | Automatic screen lock |
| 18-test-firewall-domain-profile | Firewall on with the domain profile active |
| 18b-firewall-blocks-inbound | Ping and RDP from DC01 blocked |
| 19-test-least-privilege | Standard user blocked from running as admin |
| 20-gpresult-client | GPOs applied to the client |

## Licensing

Everything here is free. VirtualBox and Ubuntu are open source, and I used Microsoft's official evaluation versions of Windows Server 2025 (180 days) and Windows 11 Enterprise (90 days), which are meant for testing and labs.
