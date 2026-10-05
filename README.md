# Windows Server 2019 AD Lab

I'm currently working as an IT Support Executive, mostly handling desktop support and user issues. I started this lab because I wanted to actually understand how the Active Directory infrastructure behind our office network works — not just reset passwords and close tickets, but know what's happening on the server side.

I built everything on an old Dell PowerEdge T110 II I had access to, running Windows Server 2019. The domain is called `sohan.lab`, and I set up 6 client VMs — 5 in VMware and 1 in Hyper-V, just to see both platforms — to simulate real users joining the domain, logging in, and running into the kind of problems you'd actually see in an office.

This isn't a copy-paste tutorial writeup — most of what's below includes the actual mistakes I made and how I figured out what was wrong, because that's honestly where most of the learning happened.

---

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

---

## Getting the server up

Installed Windows Server 2019 Standard on the T110 II. This box is from around 2011, so the installer didn't have drivers for the PERC RAID controller out of the box — had to grab Dell's driver pack separately and load it mid-install or the disk just wouldn't show up.

Confirmed everything was healthy afterward with `dcdiag` once AD was up.

**Screenshots:** server spec, OS version, dcdiag output (all passed)

---

## Setting up AD — domain, OUs, users

Promoted the server to a new forest, `sohan.lab`. Then built out a basic department structure to mirror what a small office might actually look like:

- **IT** — sohan, hridoy, tuhin
- **HR** — mim
- **Management** — alamin

Also made a security group, `IT-Staff`, with hridoy and tuhin in it — this becomes relevant later for permissions.

Verified the domain controller was healthy with `dcdiag` (all green) and checked the structure in ADUC.

**Screenshots:** OU layout, user list, IT-Staff group members

---

## Permissions (NTFS + Share)

Set up a shared folder (`D:\IT-Documents`) and configured both NTFS and Share permissions on it:

- Share permission: Everyone → Read, IT-Staff → Full Control
- NTFS permission: IT-Staff → Full Control

This is where I actually confused myself for a bit. I knew the rule — whichever permission (share or NTFS) is more restrictive wins — but when I tested it, hridoy (in IT-Staff) could write files just fine. I expected it to be blocked because I thought Everyone's "Read" at the share level would cap everything.

Turned out I'd forgotten I'd *also* given IT-Staff Full Control at the share level specifically (not just Everyone = Read). So there was no conflict — IT-Staff had Full Control on both layers, which is why it worked. Once I actually checked the share permissions properly instead of assuming, it made sense.

Tested both ways: hridoy/tuhin could create and edit files, mim (HR, not in the group) got Access Denied.

**Screenshots:** Security tab, Sharing tab, successful file creation, access denied for mim

---

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

**Screenshots:** GPO list, wallpaper per department, mapped drive

---

## Fine-grained password policy

The Default Domain Policy only lets you set one password rule for the entire domain — so if HR needed stricter rules than everyone else, that's not possible with a regular GPO. Fine-Grained Password Policy (PSO) solves this, but it only targets **users or groups**, not OUs, which tripped me up at first since every other policy I'd built was OU-based.

Made `HR-Strict-Policy` — 12 character minimum, complexity on, 5 password history — applied directly to an HR security group. Confirmed it was working via "Resultant Password Settings" on the user, and tested that a short password got rejected for mim specifically while regular domain users were unaffected.

**Screenshots:** PSO list, resultant password settings for mim

---

## Backup and restore

Set up Windows Server Backup and took a System State backup — this covers the AD database, SYSVOL, registry, and boot files, basically everything you'd need to bring a DC back from nothing.

To actually test it meant something, I deleted a test user on purpose, then restored from backup using Directory Services Restore Mode (DSRM). Had to boot into DSRM using the recovery password I'd set way back during the original domain promotion — good thing I'd written it down somewhere, because that's not something you can guess later.

Since this is a single-DC setup, a non-authoritative restore was enough — there's no second DC to "win" the replication conflict and overwrite the restored data. In a real multi-DC environment I'd have needed `ntdsutil` to mark the restore as authoritative so it actually sticks across all the DCs.

**Screenshots:** backup success, wbadmin version list

---

## Auditing

Turned on auditing for account management and directory service changes on the Default Domain Controllers Policy, then checked Event Viewer under Security logs.

The two event IDs I actually paid attention to:
- `4720` — user created
- `4726` — user deleted

Tested it by creating and deleting a user and watching both events show up with the right timestamp and the account that made the change.

**Screenshots:** filtered event log (4720/4726)

---

## DNS

Checked the Forward Lookup Zone (created automatically when AD was set up), manually added a Reverse Lookup Zone, and set Forwarders to `8.8.8.8` so the DC could resolve external domains.

### The client that couldn't find the domain

A client machine threw "Non-existent domain" trying to resolve `sohan.lab`. Checked `ipconfig /all` and the client's DNS was pointed at `8.8.8.8` directly — not the domain controller. Made sense once I thought about it: Google's DNS has literally never heard of `sohan.lab`, it only knows public internet domains. 

The actual rule I took away from this: a domain-joined machine's DNS should always be the DC, never a public resolver. The DC can still resolve public stuff like google.com because of the Forwarders setting — so the chain ends up being client → DC → forwarder → internet, not client → internet directly.

Fixed the client's DNS, flushed the cache, and both `sohan.lab` and `google.com` resolved fine after that.

**Screenshots:** forward/reverse zones, forwarders config, nslookup results

---

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

**Screenshots:** Get-ADUser output, bulk import result

---

## FSRM — quotas and file screening

Put a 5GB hard quota on the IT-Documents share — hard meaning writes actually get blocked once it's hit, as opposed to a soft quota which just sends a warning but lets you keep going.

Also set up file screening to block audio/video files (mp3, mp4, etc.) from being saved there. Tested it by trying to copy an mp3 in — got blocked, while a regular docx went through fine.

**Screenshots:** quota config, file screen blocking media files

---

## Print server

Installed the Print and Document Services role. Didn't have a spare physical printer lying around, so I set one up using a generic driver and a local port instead of an actual printer connection — still lets you test the real part that matters, which is driver sharing and how a client pulls the driver automatically from the server instead of needing it installed manually.

Connected from a client via `\\SOHAN-DC\IT-Printer` and confirmed it showed up in Devices and Printers without me touching the client machine's driver setup at all — that's the actual point of a print server.

**Screenshots:** Print Management console, printer on client

---

## What I'd say I actually learned

If I had to summarize what I can actually *do* now versus just recognize from a slide:

- Build out an AD structure from scratch — forest, OUs, users, groups
- Reason about NTFS vs Share permissions instead of just copying settings
- Troubleshoot a GPO that isn't applying (Security Filtering is usually the first thing I check now)
- Set up and actually test a disaster recovery scenario, not just configure backup and hope
- Diagnose a DNS problem from the client side instead of assuming the server is broken
- Use PowerShell for the stuff that's actually painful to do by hand

None of this makes me a senior admin, but it's a real jump from where I started — which was just resetting passwords and not knowing what was happening behind the GUI.

---

**Environment:** Dell PowerEdge T110 II · Windows Server 2019 Standard · 6 client VMs (5 on VMware Workstation, 1 on Hyper-V) · Domain: `sohan.lab`
