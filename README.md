# Windows Server 2019 AD Lab Infrastructure

I'm currently working as an IT Executive, mostly handling desktop support and user issues. I started this lab because I wanted to actually understand how the Active Directory infrastructure behind our office network works — not just reset passwords and close tickets, but know what's happening on the server side.

I built everything on an old Dell PowerEdge T110 II I had access to, running Windows Server 2019. The domain is called `sohan.lab`, and I set up 6 client VMs — 5 in VMware and 1 in Hyper-V, just to see both platforms — to simulate real users joining the domain, logging in, and running into the kind of problems you'd actually see in an office.

This isn't a copy-paste tutorial writeup — most of what's below includes the actual mistakes I made and how I figured out what was wrong, because that's honestly where most of the learning happened.
<img width="1366" height="767" alt="sys info" src="https://github.com/user-attachments/assets/68cbc219-bcc5-44a5-87b4-c20f597216ab" />
<img width="1366" height="768" alt="vmware" src="https://github.com/user-attachments/assets/fa2da0b7-e5c6-4812-895a-ed46494bc63f" />
<img width="1358" height="728" alt="hyper v" src="https://github.com/user-attachments/assets/ab4ca7b6-9078-4bf6-a5e5-9ae3c123946f" />


## What's in here

- [Getting the server up](#getting-the-server-up)
- [Setting up AD — domain, OUs, users](#setting-up-ad--domain-ous-users)
- [Permissions (NTFS + Share)](#permissions-ntfs--share)
- [Group Policy](#group-policy)
- [Fine-grained password policy](#fine-grained-password-policy)
- [Backup and restore](#backup-and-restore)
- [Auditing](#auditing)
- [DNS](#dns)
- [PowerShell](#powershell)
- [FSRM — quotas and file screening](#fsrm--quotas-and-file-screening)
- [Print server](#print-server)
- [What I'd say I actually learned](#what-id-say-i-actually-learned)


## Getting the server up

Installed Windows Server 2019 Standard on the T110 II. This box is from around 2011, so the installer didn't have drivers for the PERC RAID controller out of the box — had to grab Dell's driver pack separately and load it mid-install or the disk just wouldn't show up.

Confirmed everything was healthy afterward with `dcdiag` once AD was up.

<img width="1366" height="768" alt="server manager dashboard" src="https://github.com/user-attachments/assets/104161e2-492c-4c2f-8075-b276f75b2d1d" />
<img width="1366" height="768" alt="Domain controller" src="https://github.com/user-attachments/assets/7c779ebd-a8cf-48b9-b363-44705faa85fa" />
<img width="1366" height="764" alt="DCdiag 1" src="https://github.com/user-attachments/assets/5c9e29d4-278e-435d-9682-00a8fd89a909" />
<img width="1366" height="763" alt="DC diag 2" src="https://github.com/user-attachments/assets/5142e541-05d0-47f2-9738-95c58027402f" />


## Setting up AD — domain, OUs, users

Promoted the server to a new forest, `sohan.lab`. Then built out a basic department structure to mirror what a small office might actually look like:

- **IT** — sohan, hridoy, tuhin
- **HR** — mim, Akib, Shamim
- **Management** — alamin, kamal
- **SCM** —

Also made a security group, `IT-Staff`, with sohan, hridoy and tuhin in it — this becomes relevant later for permissions.

Verified the domain controller was healthy with `dcdiag` (all green) and checked the structure in ADUC.
<img width="1366" height="763" alt="DC diag 2" src="https://github.com/user-attachments/assets/ed2312da-79fd-4ede-935f-b16ae214246b" />
<img width="1366" height="768" alt="computers" src="https://github.com/user-attachments/assets/ab74644a-506a-4d11-9a28-81f3ad3b8c23" />
<img width="1364" height="765" alt="AD stracture sohan lab" src="https://github.com/user-attachments/assets/59f56bd9-aa2c-4828-b76a-2dfecd129c4f" />
<img width="1366" height="767" alt="IT OU" src="https://github.com/user-attachments/assets/4c0bad3f-3b0f-4590-b372-2b9ed7e78a1c" />
<img width="1366" height="768" alt="IT Group" src="https://github.com/user-attachments/assets/0c378ad4-4d8a-47fc-9a6f-455c82d01a47" />
<img width="1366" height="767" alt="HR OU" src="https://github.com/user-attachments/assets/dce71804-e7cf-46b2-9a43-88c310964fd0" />


## Permissions (NTFS + Share)

Set up a shared folder (`E:\IT share`) and configured both NTFS and Share permissions on it:

- Share permission: Everyone → Read, IT-Staff → Full Control
- NTFS permission: IT-Staff → Full Control

This is where I actually confused myself for a bit. I knew the rule — whichever permission (share or NTFS) is more restrictive wins — but when I tested it, hridoy (in IT-Staff) could write files just fine. I expected it to be blocked because I thought Everyone's "Read" at the share level would cap everything.

Turned out I'd forgotten I'd *also* given IT-Staff Full Control at the share level specifically (not just Everyone = Read). So there was no conflict — IT-Staff had Full Control on both layers, which is why it worked. Once I actually checked the share permissions properly instead of assuming, it made sense.

Tested both ways: hridoy/tuhin could create and edit files, mim (HR, not in the group) got Access Denied.

<img width="1366" height="768" alt="Client access1" src="https://github.com/user-attachments/assets/692d0a54-3dfe-417c-8cef-95fbd6d0c971" />
<img width="807" height="604" alt="SMB 1" src="https://github.com/user-attachments/assets/18e5dae4-e73a-42bc-9aa8-be95ecf22032" />
<img width="1366" height="765" alt="mim access denied it share " src="https://github.com/user-attachments/assets/a6bd3a9a-1d79-4db9-ad1f-d05a6d143ffe" />
<img width="1366" height="768" alt="mim  client" src="https://github.com/user-attachments/assets/ef49da61-8e8a-492e-8a29-a0d69288e8e0" />
<img width="491" height="583" alt="IT share" src="https://github.com/user-attachments/assets/5c0047ff-39ed-4c30-a4b9-752c9ef5a5f1" />

## Group Policy

Built a handful of GPOs scoped to different OUs:

| What | Does what | Where |
|---|---|---|
| IT-Wallpaper-Policy | custom wallpaper | IT |
| HR-Wallpaper-Policy | custom wallpaper | HR |
| Management-Wallpaper-Policy | custom wallpaper | Management |
| IT-Drive-Mapping | maps `Z:` to the IT-Documents share | IT |
| Default Domain Policy | password rules, lockout | whole domain |
| Screen-Lock-Policy | 10 min idle lock | IT |
| Restrict-Control-Panel | blocks Settings/Control Panel | IT |
| Logon Hours | time-based login restriction | per-user, via Account tab |

### The GPO that wouldn't apply

Made a new GPO, linked it to an OU, ran `gpupdate /force` on the client — nothing happened. Checked `gpresult /r` and saw it listed under "Filtered" instead of "Applied."

Went into the GPO's Scope tab and found the problem: **Security Filtering didn't have "Authenticated Users" in it.** No idea exactly how it got removed, but without it, nobody actually had permission to read and apply the policy — it existed and was linked correctly, it just had no one allowed to use it.

Added Authenticated Users back in, and it started working immediately. After that I just got in the habit of checking Security Filtering right after creating any new GPO, before troubleshooting anything else.

### The password policy that silently ignored my setting

Set minimum password length to 4 in Default Domain Policy, but left "Password must meet complexity requirements" enabled. New users still couldn't set a 4-character password — Windows was enforcing a 6-character minimum anyway because complexity was on, regardless of what I'd typed for length. Disabled complexity and the 4-character minimum actually took effect. Small thing, but good reminder that these settings interact with each other, not just individually.
<img width="1364" height="768" alt="HR wallparper" src="https://github.com/user-attachments/assets/4d317ca2-8cd9-4d30-847d-e301711448a6" />
<img width="1366" height="766" alt="gpo" src="https://github.com/user-attachments/assets/24a6c240-3844-4e09-9248-d31eb7033f7f" />
<img width="868" height="639" alt="logon script" src="https://github.com/user-attachments/assets/9ae7beed-a6ce-409b-9e8a-ce46a67ce38e" />
<img width="1364" height="766" alt="IT wallpaper" src="https://github.com/user-attachments/assets/26f4fc1b-994f-4750-a3a4-a95729acd419" />



## Fine-grained password policy

The Default Domain Policy only lets you set one password rule for the entire domain — so if HR needed stricter rules than everyone else, that's not possible with a regular GPO. Fine-Grained Password Policy (PSO) solves this, but it only targets **users or groups**, not OUs, which tripped me up at first since every other policy I'd built was OU-based.

Made `HR-Strict-Policy` — 12 character minimum, complexity on, 5 password history — applied directly to an HR security group. Confirmed it was working via "Resultant Password Settings" on the user, and tested that a short password got rejected for mim specifically while regular domain users were unaffected.

<img width="1357" height="737" alt="HR auth" src="https://github.com/user-attachments/assets/580ced98-2b29-460c-aab8-f810337c5110" />


## Backup and restore

Set up Windows Server Backup and took a System State backup — this covers the AD database, SYSVOL, registry, and boot files, basically everything you'd need to bring a DC back from nothing.

To actually test it meant something, I deleted a test user on purpose, then restored from backup using Directory Services Restore Mode (DSRM). Had to boot into DSRM using the recovery password I'd set way back during the original domain promotion — good thing I'd written it down somewhere, because that's not something you can guess later.

Since this is a single-DC setup, a non-authoritative restore was enough — there's no second DC to "win" the replication conflict and overwrite the restored data. In a real multi-DC environment I'd have needed `ntdsutil` to mark the restore as authoritative so it actually sticks across all the DCs.

<img width="1366" height="765" alt="backup dahsboard" src="https://github.com/user-attachments/assets/3e408759-28d3-4a78-aea7-6fb870bbf4f8" />
<img width="1364" height="767" alt="backup cmd out put" src="https://github.com/user-attachments/assets/a4d39a8a-45c9-4c09-9324-e9c5fa720153" />


## Auditing

Turned on auditing for account management and directory service changes on the Default Domain Controllers Policy, then checked Event Viewer under Security logs.

The two event IDs I actually paid attention to:
- `4720` — user created
- `4726` — user deleted

Tested it by creating and deleting a user and watching both events show up with the right timestamp and the account that made the change.

<img width="1364" height="740" alt="event" src="https://github.com/user-attachments/assets/766cc775-bd7b-4ca1-be1f-161c0a3c3c78" />


## DNS

Checked the Forward Lookup Zone (created automatically when AD was set up), manually added a Reverse Lookup Zone, and set Forwarders to `8.8.8.8` so the DC could resolve external domains.

### The client that couldn't find the domain

A client machine threw "Non-existent domain" trying to resolve `sohan.lab`. Checked `ipconfig /all` and the client's DNS was pointed at `8.8.8.8` directly — not the domain controller. Made sense once I thought about it: Google's DNS has literally never heard of `sohan.lab`, it only knows public internet domains. 

The actual rule I took away from this: a domain-joined machine's DNS should always be the DC, never a public resolver. The DC can still resolve public stuff like google.com because of the Forwarders setting — so the chain ends up being client → DC → forwarder → internet, not client → internet directly.

Fixed the client's DNS, flushed the cache, and both `sohan.lab` and `google.com` resolved fine after that.

<img width="717" height="636" alt="clinet cmd" src="https://github.com/user-attachments/assets/e3e50c6f-67ea-4383-9af7-c86a42ffd9c8" />
<img width="1366" height="768" alt="reverse lookup zone" src="https://github.com/user-attachments/assets/73b6428d-561c-460a-9880-c17c00f8a3f2" />
<img width="413" height="477" alt="forwarder" src="https://github.com/user-attachments/assets/96ad184b-f30c-42a5-ae64-8ee255331350" />
<img width="1362" height="767" alt="forward lookup zone" src="https://github.com/user-attachments/assets/a10860d7-ff13-4706-a7dc-14c547fa960f" />


## PowerShell

Used this mainly for bulk user creation, since doing the same thing 10+ times through the GUI wizard gets old fast.

```powershell
# look up users in a specific OU
Get-ADUser -Filter * -SearchBase "OU=IT,DC=sohan,DC=lab" | Select Name,SamAccountName

# create one user
New-ADUser -Name "James Smith" -SamAccountName "jsmith" `
  -Path "OU=SCM,DC=sohan,DC=lab" `
  -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
  -Enabled $true

# bulk create from a CSV
Import-Csv "C:\Temp\users.csv" | ForEach-Object {
    New-ADUser -Name $_.Name -SamAccountName $_.SamAccountName `
      -Path "OU=SCM,DC=sohan,DC=lab" `
      -AccountPassword (ConvertTo-SecureString "P@ssw0rd123" -AsPlainText -Force) `
      -Enabled $true
}
```

Honestly, for a single user the GUI is faster — PowerShell only pays off once you're doing something repetitive. The CSV import made 10 users in about the time it'd take to fill out the first GUI form. Also useful for stuff that's basically impossible to do cleanly in the GUI, like checking who hasn't logged in recently:

```powershell
Get-ADUser -Filter * -Properties LastLogonDate | Select Name,LastLogonDate | Sort LastLogonDate
```

<img width="1347" height="726" alt="bulk user creat" src="https://github.com/user-attachments/assets/5246ebd6-2be4-4953-bc2c-58544630252a" />
<img width="678" height="694" alt="all user and computer" src="https://github.com/user-attachments/assets/f760ba1c-7654-48c3-a6dc-189dfead52dc" />
<img width="1366" height="346" alt="user add" src="https://github.com/user-attachments/assets/a391c5de-37e3-4c1d-9236-b49579061174" />
<img width="963" height="395" alt="get user name ps" src="https://github.com/user-attachments/assets/da07e722-c166-4112-bed9-011024dd6ba1" />
<img width="1350" height="373" alt="bulk user create , pass reset" src="https://github.com/user-attachments/assets/e972183c-c295-4a70-aeb9-06c001d14e03" />


## FSRM — quotas and file screening

Put a 5GB hard quota on the IT-Documents share — hard meaning writes actually get blocked once it's hit, as opposed to a soft quota which just sends a warning but lets you keep going.

Also set up file screening to block audio/video files (mp3, mp4, etc.) from being saved there. Tested it by trying to copy an mp3 in — got blocked and also set up e-mail for notification, while a regular docx went through fine.

<img width="913" height="520" alt="file screeening" src="https://github.com/user-attachments/assets/b13aa0c4-4675-4d3b-ba54-027c24b19e9a" />
<img width="493" height="604" alt="emial conf" src="https://github.com/user-attachments/assets/d4f42f5b-df6c-4455-b933-ec14b700c099" />
<img width="1419" height="318" alt="quota" src="https://github.com/user-attachments/assets/40199e75-fd83-4f94-97b2-c7f086f02677" />
<img width="1366" height="726" alt="quota management" src="https://github.com/user-attachments/assets/337d6698-bdb4-4f1d-aedd-706b91f4a69c" />
<img width="1366" height="727" alt="file screeneing" src="https://github.com/user-attachments/assets/382c17fa-c843-46a1-81b0-74633abdd635" />


## Print server

Installed the Print and Document Services role. Didn't have a spare physical printer lying around, so I set one up using a generic driver and a local port instead of an actual printer connection — still lets you test the real part that matters, which is driver sharing and how a client pulls the driver automatically from the server instead of needing it installed manually.

Connected from a client via `\\SOHAN-DC\IT-Printer` and confirmed it showed up in Devices and Printers without me touching the client machine's driver setup at all — that's the actual point of a print server.

<img width="801" height="555" alt="client end" src="https://github.com/user-attachments/assets/e0fec5bf-f30b-4026-b813-4df018ac01d3" />
<img width="1366" height="766" alt="dc print server" src="https://github.com/user-attachments/assets/6a7f910f-23e6-4bb8-8cd5-111b0b6cea20" />


## What I'd say I actually learned

If I had to summarize what I can actually *do* now versus just recognize from a slide:

- Build out an AD structure from scratch — forest, OUs, users, groups
- Reason about NTFS vs Share permissions instead of just copying settings
- Troubleshoot a GPO that isn't applying (Security Filtering is usually the first thing I check now)
- Set up and actually test a disaster recovery scenario, not just configure backup and hope
- Diagnose a DNS problem from the client side instead of assuming the server is broken
- Use PowerShell for the stuff that's actually painful to do by hand

None of this makes me a senior admin, but it's a real jump from where I started — which was just resetting passwords and not knowing what was happening behind the GUI.


**Environment:** Dell PowerEdge T110 II · Windows Server 2019 Standard · 6 client VMs (5 on VMware Workstation, 1 on Hyper-V) · Domain: `sohan.lab`
