# Phase 2 — Central Management

Phase 1 gave ForgeLine a working core infrastructure. We had Active Directory, redundant Domain Controllers, DNS, centralized administration, file services and domain-joined workstations.

The goal of Phase 2 was to make that environment **centrally manageable**.

Instead of configuring every workstation manually, configuration is now distributed through Active Directory and Group Policy. This saves A LOT of time, keeps machines consistent and reduces the chances of configuration errors.

The main goals were:

- Centralize workstation configuration using Group Policy
- Separate computer and user configuration
- Centrally manage Windows Firewall, Defender and Windows Update
- Automatically map resources based on AD group membership
- Configure domain account policies
- Deploy Windows LAPS for local administrator password management
- Validate policies on multiple users and workstations
- Learn how to troubleshoot Group Policy and AD-related issues

---

## Management Architecture

Phase 2 builds on the OU structure created in Phase 1.

```text
corp.forgeline.test
│
├── Domain Controllers
│   ├── DC01
│   └── DC02
│
└── ForgeLine
    ├── Admins
    ├── Groups
    ├── Servers
    ├── Users
    │   ├── Abdullah
    │   └── TestUser
    │
    └── Workstations
        ├── CLIENT01
        └── CLIENT02
```

MGMT01 remains the central management server.

Group Policy Management was added to MGMT01 so that GPOs can be managed remotely instead of logging directly into a Domain Controller.

```powershell
Install-WindowsFeature GPMC
```

The management model now looks roughly like this:

```text
                         MGMT01
                            │
                     GPMC / RSAT
                            │
                            ↓
                   Active Directory
                            │
                     Group Policies
                            │
             ┌──────────────┴──────────────┐
             ↓                             ↓
          Users                       Workstations
             │                             │
             │                      ┌──────┴──────┐
             │                      ↓             ↓
             │                  CLIENT01      CLIENT02
             │
             └── User-specific configuration
```

---

# Group Policy Design

A **Group Policy Object (GPO)** is basically a collection of settings that can be managed centrally.

Creating a GPO alone does nothing. It has to be linked to the correct location in Active Directory.

For example:

```text
ForgeLine
└── Workstations
    │
    ├── FL-Workstation-Firewall
    ├── FL-Workstation-Defender
    ├── FL-Workstation-Updates
    └── FL-Workstation-LAPS
         │
         ├── CLIENT01
         └── CLIENT02
```

This is exactly why the OU structure from Phase 1 was important. It was not mere decoration.

## Computer vs User Configuration

One of the most important things I learned while working with Group Policy is the difference between computer and user configuration.

**Computer Configuration follows the computer.**

```text
Workstations OU
      ↓
Computer GPO
      ↓
CLIENT01
```

The settings apply to CLIENT01 regardless of which user logs into the machine.

**User Configuration follows the user.**

```text
Users OU
   ↓
User GPO
   ↓
Abdullah
```

This becomes especially useful when a resource should only appear for users belonging to a specific department or security group.

---

# Workstation Policies

Instead of putting every workstation setting into one massive GPO, ForgeLine uses separate GPOs based on their purpose.

```text
FL-Workstation-Firewall
FL-Workstation-Defender
FL-Workstation-Updates
FL-Workstation-LAPS
```

All of these are linked to:

```text
ForgeLine\Workstations
```

This keeps the configuration easier to understand and troubleshoot.

---

## Windows Firewall

`FL-Workstation-Firewall` centrally manages Windows Defender Firewall on domain workstations.

The Domain Profile is enforced as enabled through Group Policy.

```text
FL-Workstation-Firewall
        ↓
ForgeLine\Workstations
        ↓
 ┌──────┴──────┐
 ↓             ↓
CLIENT01    CLIENT02
```

### Validation

After Group Policy processing:

```powershell
gpupdate /force
Get-NetFirewallProfile
```

The Domain firewall profile was confirmed to be enabled.

The source of the configuration can also be checked using:

```cmd
gpresult /r
```

This confirms that the workstation received `FL-Workstation-Firewall` rather than relying on a manually configured local setting.

---

## Microsoft Defender

Microsoft Defender is centrally managed using:

```text
FL-Workstation-Defender
```

The policy ensures that Defender and real-time protection remain enabled on ForgeLine workstations.

The configuration applies to:

```text
ForgeLine\Workstations
├── CLIENT01
└── CLIENT02
```

### Validation

After processing Group Policy:

```cmd
gpupdate /force
```

Defender status can be checked using:

```powershell
Get-MpComputerStatus
```

This allows the security configuration to be controlled centrally rather than depending on the local configuration of every workstation.

---

## Windows Update

Automatic update behavior is centrally configured through:

```text
FL-Workstation-Updates
```

The configured update behavior is:

```text
Automatic Updates: Enabled
Mode: Auto download and schedule the install
Schedule: Every day
Time: 03:00
```

ForgeLine does not currently use WSUS.

The workstations still retrieve their updates from Microsoft, while Group Policy controls how those updates are handled.

```text
Microsoft Update
       ↑
       │
CLIENT01 / CLIENT02
       ↑
       │
FL-Workstation-Updates
```

This gives us centralized update behavior without introducing another infrastructure service before it is actually needed.

---

# Automatic Finance Drive Mapping

The Finance drive mapping combines several things built during Phase 1 and Phase 2:

```text
Active Directory users
        +
Security Groups
        +
Group Policy
        +
Item-Level Targeting
        +
SMB / NTFS permissions
```

A user belonging to:

```text
GG_Finance_Users
```

automatically receives:

```text
F: → \\FILE01\Finance
```

The configuration is stored in:

```text
FL-User-DriveMappings
```

and linked to:

```text
ForgeLine\Users
```

This is a **user policy** because the drive should follow the user, not the workstation.

Item-level targeting checks membership of:

```text
FORGELINE\GG_Finance_Users
```

The resulting logic is:

```text
FL-User-DriveMappings
        ↓
F: → \\FILE01\Finance
        ↓
Item-Level Targeting
        ↓
User ∈ GG_Finance_Users?
       /          \
     YES           NO
      ↓             ↓
   Map F:       Don't map
```

## Validation

`Abdullah` is a member of:

```text
GG_Finance_Users
```

After login on CLIENT01:

```text
Finance (F:) → \\FILE01\Finance
```

appears automatically.

The mapping can also be verified using:

```cmd
net use
```

TestUser is **not** a member of `GG_Finance_Users`.

When TestUser logs into CLIENT02:

```text
Finance F: → NOT mapped
```

This gives us both a positive and negative policy test:

```text
Abdullah
└── Finance group
    └── F: mapped ✓


TestUser
└── Domain Users only
    └── F: not mapped ✓
```

The drive mapping itself is not the security boundary.

The NTFS permissions configured during Phase 1 still determine whether the user can actually access the files.

Therefore:

```text
GPO targeting
↓
Controls whether the drive appears automatically


NTFS authorization
↓
Controls whether access is actually allowed
```

Even if TestUser manually attempts:

```text
\\FILE01\Finance
```

access is still denied.

---

# Domain Account Policy

Domain account security was also reviewed during this phase.

The domain password policy includes:

```text
Minimum password length: 14
Password complexity: Enabled
Reversible encryption: Disabled
```

An important distinction here is that domain password policy is a **domain-level configuration**.

It is not simply a user GPO linked to:

```text
ForgeLine\Users
```

The scope is:

```text
Domain Account Policy
        ↓
corp.forgeline.test
        ↓
Domain Users
```

This was another useful example of why understanding GPO scope matters instead of just creating policies and hoping Windows interprets our intentions telepathically.

---

# Windows LAPS

One of the most important additions in Phase 2 was Windows LAPS.

**LAPS = Local Administrator Password Solution**

The problem is simple.

If every workstation uses the same local administrator credentials:

```text
CLIENT01\ADMIN → SamePassword
CLIENT02\ADMIN → SamePassword
CLIENT03\ADMIN → SamePassword
```

compromising one local administrator credential may make it possible to reuse those credentials against other workstations.

That is horrible.

Windows LAPS solves this by managing a different local administrator password for every machine.

```text
CLIENT01\ADMIN → unique password A
CLIENT02\ADMIN → unique password B
```

The passwords can also be automatically rotated and backed up to Active Directory.

---

## Active Directory Integration

ForgeLine uses the modern Windows LAPS implementation included with current Windows versions rather than legacy Microsoft LAPS.

The Active Directory schema was extended using:

```powershell
Update-LapsADSchema
```

The Workstations OU was then delegated permission allowing its computer accounts to update their own LAPS information:

```powershell
Set-LapsADComputerSelfPermission `
    -Identity "OU=Workstations,OU=ForgeLine,DC=corp,DC=forgeline,DC=test"
```

A dedicated GPO was created:

```text
FL-Workstation-LAPS
```

and linked to:

```text
ForgeLine\Workstations
```

The overall flow is:

```text
CLIENT01 / CLIENT02
        ↓
FL-Workstation-LAPS
        ↓
Generate unique local admin password
        ↓
Windows LAPS
        ↓
Back up password information to AD
```

---

## Managing a Custom Local Administrator

During implementation I noticed that LAPS was managing the built-in:

```text
Administrator
```

account.

That account was disabled on my clients.

The local administrator account I actually use is:

```text
ADMIN
```

The LAPS policy was therefore configured with:

```text
Administrator account name: ADMIN
```

Both workstations use the same standardized local administrator **username**:

```text
CLIENT01\ADMIN
CLIENT02\ADMIN
```

but they do not share passwords.

```text
CLIENT01\ADMIN → unique password
CLIENT02\ADMIN → different unique password
```

This distinction is important:

```text
Same username       ✓
Same password       ✗
```

**Lesson learned:** when a LAPS policy manages a custom local administrator account, that account name needs to be consistent across the computers targeted by that policy.

---

## LAPS Validation

Policy processing was triggered on the clients using:

```powershell
gpupdate /force
Invoke-LapsPolicyProcessing
```

The stored LAPS information can be retrieved from an authorized management session using:

```powershell
Get-LapsADPassword -Identity CLIENT01
Get-LapsADPassword -Identity CLIENT02
```

CLIENT01 and CLIENT02 returned different local administrator passwords.

This confirmed that LAPS was independently managing the local `ADMIN` account on each workstation.

```text
             Active Directory
                    ↑
             Windows LAPS
              ↑           ↑
              │           │
        CLIENT01       CLIENT02
        ADMIN: A       ADMIN: B
```

---

# Troubleshooting: LAPS Schema Update Failure

Not everything worked immediately.

While preparing Active Directory for Windows LAPS:

```powershell
Update-LapsADSchema
```

failed with:

```text
DirectoryOperationException: An operation error occurred.
```

The account permissions were correct, so I investigated Active Directory itself.

DC02 had been powered off to save resources, and DC01 had restarted while DC02 was unavailable.

Although DC01 still appeared as the Schema Master FSMO role holder, replication with its partner was not healthy.

I checked replication using:

```powershell
repadmin /replsummary
repadmin /showrepl DC01
```

The Schema partition still showed an old RPC replication failure.

Replication of the Schema partition was then forced from DC02 to DC01:

```powershell
repadmin /replicate DC01 DC02 "CN=Schema,CN=Configuration,DC=corp,DC=forgeline,DC=test"
```

After replication was restored, the schema modification could proceed.

The troubleshooting path was basically:

```text
Update-LapsADSchema fails
        ↓
Check permissions
        ↓
Permissions are correct
        ↓
Inspect AD replication
        ↓
Schema partition replication failure
        ↓
DC02 brought online
        ↓
Force DC02 → DC01 replication
        ↓
Replication healthy
        ↓
Schema update succeeds
```

### Lesson Learned

Before making important Active Directory changes, especially **schema changes**, replication health should be verified first.

```powershell
repadmin /replsummary
```

A Domain Controller appearing to own an FSMO role does not necessarily mean it is currently in a healthy state to perform every operation associated with that role.

This issue also demonstrated why replication health matters even in a small two-DC lab.

---

# Group Policy Troubleshooting

Another major part of working with Group Policy is knowing how to prove where a configuration came from when something doesn't work.

Two commands became especially useful during this phase.

Force Group Policy processing:

```cmd
gpupdate /force
```

Show applied policies:

```cmd
gpresult /r
```

For a more detailed report:

```cmd
gpresult /h C:\gpresult.html
```

The HTML report provides considerably more information about the user and computer policies that were processed.

My basic GPO troubleshooting process is now:

```text
Policy not working
       ↓
1. Is the user/computer in the correct OU?
       ↓
2. Is the GPO linked to the correct OU?
       ↓
3. Is it User or Computer Configuration?
       ↓
4. gpupdate /force
       ↓
5. gpresult
       ↓
6. Event Viewer if necessary
```

This is much better than randomly changing settings until Windows eventually gives up and does what I wanted.

---

# Validation

Phase 2 was tested using two workstations and two users with different authorization requirements.

## CLIENT01 — Abdullah

```text
Domain user authentication              ✓
Workstation GPO processing              ✓
Windows Firewall policy                 ✓
Microsoft Defender policy               ✓
Windows Update policy                   ✓
Finance F: drive mapped                 ✓
Finance share access                    ✓
LAPS-managed local ADMIN                ✓
```

Finance Modify permissions were also tested through the mapped drive:

```cmd
echo Phase2-Test > F:\phase2-test.txt
type F:\phase2-test.txt
del F:\phase2-test.txt
```

The create, read and delete operations succeeded.

---

## CLIENT02 — TestUser

```text
Domain user authentication              ✓
Workstation GPO processing              ✓
Windows Firewall policy                 ✓
Microsoft Defender policy               ✓
Windows Update policy                   ✓
Finance F: drive mapped                 ✗
Finance share access                    ✗
LAPS-managed local ADMIN                ✓
```

This is intentional.

CLIENT02 receives the workstation policies because the **computer** belongs to:

```text
ForgeLine\Workstations
```

TestUser does not receive the Finance mapping because the **user** is not a member of:

```text
GG_Finance_Users
```

This validates both computer-based and user/group-based central management.

---

# Phase 2 Result

Phase 1 gave ForgeLine working infrastructure.

Phase 2 made that infrastructure centrally manageable.

```text
                         MGMT01
                            │
                      GPMC / RSAT
                            │
                            ↓
                    Active Directory
                            │
                      Group Policy
                            │
          ┌─────────────────┴─────────────────┐
          ↓                                   ↓
     Workstations                            Users
          │                                   │
   ┌──────┴──────┐                    Group targeting
   ↓             ↓                           │
CLIENT01      CLIENT02                GG_Finance_Users
   │             │                           │
   ├─ Firewall   ├─ Firewall                 ↓
   ├─ Defender   ├─ Defender          F: → \\FILE01\Finance
   ├─ Updates    ├─ Updates
   └─ LAPS       └─ LAPS
```

Implemented and validated:

```text
Central Group Policy management          ✓
Computer-based policy management         ✓
User-based policy management             ✓
Windows Firewall policy                  ✓
Microsoft Defender policy                ✓
Windows Update policy                    ✓
Automatic Finance drive mapping          ✓
Security-group-based targeting           ✓
Domain password policy                   ✓
Windows LAPS                             ✓
Unique local administrator passwords     ✓
AD-backed LAPS management                ✓
GPO troubleshooting                      ✓
Positive authorization testing           ✓
Negative authorization testing           ✓
AD replication troubleshooting           ✓
```

ForgeLine now has a central management layer instead of relying on manually configured workstations.

The next phase will replace the current flat VMware network with a more realistic network architecture.

```text
Phase 3 — Network Architecture

Router / Firewall
DHCP
Multiple subnets
Network segmentation
Management network
Server network
Client network
Inter-network firewall rules
```