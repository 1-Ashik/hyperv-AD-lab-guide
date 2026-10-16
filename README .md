# Hyper-V Home Lab — Build & Operations Guide

Ashik Khadka · built and documented October 2026 · Adelaide, South Australia

> **Note on sizes.** The memory and disk figures in this guide (6 GB for the server VM, 1–2 GB per nested VM, an 80 GB lab disk) are what fits a 16 GB laptop with limited free space. With more RAM or disk, raise them as you wish; the steps do not change.

## Contents

- [About this document](#about-this-document) · [Lab overview](#lab-overview) · [Prerequisites](#prerequisites)
- **Hyper-V (Parts 1–11)**
  - [Part 1 — Prepare the host laptop](#part-1--prepare-the-host-laptop)
  - [Part 2 — Create the Windows Server VM in VirtualBox](#part-2--create-the-windows-server-vm-in-virtualbox)
  - [Part 3 — Install Windows Server 2025 and the Hyper-V role](#part-3--install-windows-server-2025-and-the-hyper-v-role)
  - [Part 4 — Add the L: lab data disk](#part-4--add-the-l-lab-data-disk)
  - [Part 5 — Virtual switches](#part-5--virtual-switches)
  - [Part 6 — Build the nested VMs](#part-6--build-the-nested-vms)
  - [Part 7 — Connectivity tests](#part-7--connectivity-tests)
  - [Part 8 — Virtual disks](#part-8--virtual-disks)
  - [Part 9 — Checkpoints, export/import, template and clone](#part-9--checkpoints-exportimport-template-and-clone)
  - [Part 10 — Resources and tuning](#part-10--resources-and-tuning)
  - [Part 11 — PowerShell automation and PowerShell Direct](#part-11--powershell-automation-and-powershell-direct)

- **Active Directory (Parts 12–17)**
  - [Part 12 — Build the domain controller](#part-12-active-directory-build-the-domain-controller)
  - [Part 13 — Manage the domain from HYPERVLAB](#part-13--manage-the-domain-from-hypervlab)
  - [Part 14 — OUs, users and a group](#part-14--ous-users-and-a-group)
  - [Part 15 — Join WS-CORE02 to the domain](#part-15--join-ws-core02-to-the-domain)
  - [Part 16 — Group Policy: a logon banner](#part-16--group-policy-a-logon-banner)
  - [Part 17 — Join HYPERVLAB for GUI management](#part-17--join-hypervlab-for-gui-management)

- **Reference**
  - [Daily operations](#daily-operations)
  - [Troubleshooting and lessons learned](#troubleshooting-and-lessons-learned)
  - [Appendix A — Screenshots](#appendix-a--screenshots)
  - [Appendix B — Quick command reference](#appendix-b--quick-command-reference)
  - [Next steps — Phases 8–10](#next-steps--phases-810)

## About this document

This guide rebuilds a complete Hyper-V learning lab from a blank Windows 11 Home laptop, in the order that actually works, with every command and setting that was needed. Parts 1–11 cover Phases 1–7 of the lab plan (setup through PowerShell automation). Parts 12–17 add an Active Directory domain on top: a domain controller, users and groups, a member server, Group Policy and GUI management. Phases 8–10 are listed at the end as next steps.

**Who it is for.** Anyone who wants to learn Microsoft Hyper-V hands-on without Windows Pro or a second machine. It assumes basic Windows and command-line comfort, nothing more.

**Why it is built this way.** Windows 11 Home has no Hyper-V. The workaround is to run Windows Server 2025 as a VirtualBox virtual machine and install Hyper-V inside it (nested virtualization). Everything after Part 3 is then real Hyper-V, exactly as it runs in a workplace.

**Where commands run.** Three places are used throughout. Every command block states which one.

| Label | Machine | How to get a prompt |
|---|---|---|
| HOST | Your laptop (Windows 11 Home) | Start → type `PowerShell` → Run as administrator |
| HYPERVLAB | The Windows Server 2025 VM in VirtualBox (the Hyper-V host) | Inside the VM: Start → Terminal (Admin) |
| Inside a nested VM | WS-CORE01, WS-CORE02 or UBU01, running inside HYPERVLAB | Hyper-V Manager → right-click VM → Connect; on Server Core type `15` in sconfig for PowerShell |

**Conventions.** `Code like this` is typed exactly. Paths, names and IP addresses are the ones used in this build; change them consistently if you change them at all. Each part ends with a checkpoint: a VirtualBox snapshot you take only after a clean shutdown of HYPERVLAB.

**Screenshots.** Every part has its screenshots in place, taken during the October 2026 rebuild. Appendix A lists what each one shows and how to retake one.

**What you will have at the end.** One laptop running, in layers:

- **HYPERVLAB** — a Windows Server 2025 virtual machine in VirtualBox, with the Hyper-V role installed. This is the "server" you manage.
- Inside it, three small Hyper-V VMs on a private 10.20.20.0/24 network: **WS-CORE01** (Windows Server Core, later the domain controller for `lab.local`), **WS-CORE02** (Windows Server Core, a member server cloned from a template) and **UBU01** (Ubuntu Server).
- A working Active Directory domain: users, groups, OUs, a joined member server, a Group Policy that changes a server's sign-in screen, and the graphical admin tools on HYPERVLAB.

**How to use this guide.** Do the parts in order; each one assumes the ones before it. Every command block is labelled with where it runs (HOST, HYPERVLAB, or inside a named VM) — check the prompt before pressing Enter. Lines starting with `#` are explanations, not commands. Paste a block whole unless it contains `Read-Host` (a password prompt): then run it one line at a time, or the prompt will swallow the next line. When something fails, the Troubleshooting table at the end has every error met in two builds of this lab, with its fix. Take the VirtualBox snapshot named at the end of each part; it is your undo button for the whole server.

## Lab overview

The whole lab runs on one 16 GB laptop: VirtualBox hosts a Windows Server 2025 VM called HYPERVLAB, and Hyper-V inside HYPERVLAB hosts three small lab VMs on an isolated 10.20.20.0/24 network.

![Lab architecture — three layers, three switches, three VMs](images/lab-architecture.png)

Only `lab-internal` carries day-to-day traffic. `lab-external` exists for internet access during builds, and `lab-private` is only used to demonstrate full isolation. Because HYPERVLAB has 6 GB, memory is tight: UBU01 runs alone, and once the domain exists (Part 12 onwards) WS-CORE01 at 1280 MB plus WS-CORE02 at 768 MB is the most that fits.

**Where things live**

| Item | Location |
|---|---|
| Windows Server 2025 ISO (for HYPERVLAB) | HOST `C:\ISOs\` |
| HYPERVLAB VM files and snapshots | HOST `C:\VMs\WinServer2025\` (VirtualBox) |
| Launcher script | HOST desktop `start-lab.cmd` |
| ISOs used inside HYPERVLAB | HYPERVLAB `C:\ISOs\` |
| Nested VMs (configs + VHDX) | HYPERVLAB `L:\VMs\<VMName>\` |
| Read-only template | HYPERVLAB `L:\VMs\Templates\ws2025-core-template.vhdx` |
| Scripts | HYPERVLAB `L:\Scripts\New-LabVM.ps1` |
| Kali and Metasploitable (separate VirtualBox lab) | HOST `D:\` — not part of this guide |

**VirtualBox snapshots (restore points)**

| Snapshot | Taken after |
|---|---|
| `clean-base-hyperv` | Part 3 — Hyper-V role installed, test VM ran |
| `phase3-vms-built` | Part 7 — both VMs built, connectivity tests passed |
| `phase4-disks` | Part 8 |
| `phase5-checkpoints-clone` | Part 9 |
| `phase6-tuning` | Part 10 |
| `phase7-powershell` | Part 11 |
| `ad-dc-built` | Part 12 — domain controller promoted, health check passed |
| `before-hypervlab-join-domain` | Part 17.1 — before HYPERVLAB joins the domain (undo point) |
| `ad-domain-complete` | Part 17 — finished lab |

## Prerequisites

Everything below is free for a student; the only cost is disk space and about two days of time.

| Need | Used in this build | Notes |
|---|---|---|
| Laptop with VT-x and 16 GB RAM | Dell Inspiron 16 5630, i7-1360P, 16 GB | 8 GB is not enough. A hybrid P/E-core Intel CPU needs the affinity launcher in Part 2. |
| ~120 GB free disk on one drive | Host `C:` | VirtualBox files cannot be split across drives safely. Small second drives (like a 79 GB `D:`) fill up mid-build. |
| Windows 11 (Home is fine) | Windows 11 Home 25H2 | Pro/Education would allow Hyper-V directly and skip VirtualBox, but the nested method still teaches the same Hyper-V. |
| Oracle VirtualBox 7.x | 7.2 | Latest from virtualbox.org. Extension Pack not needed. Guest Additions are optional (Part 3.6, copy and paste into the server). |
| Windows Server 2025 Standard ISO + key | From Azure Dev Tools for Teaching (Education Hub), via UniSC login | Free, fully licensed. Evaluation ISO from microsoft.com/evalcenter works too (180-day). Avoid any file with `LOF_PACKAGES` in the name — that is a language pack, not Windows. |
| Ubuntu Server ISO | 26.04.1 LTS | ubuntu.com/download/server, about 3 GB. |
| Internet | Home broadband | Downloads total about 15 GB. |

**Downloads to do before starting**

1. VirtualBox installer → install on the laptop.
2. Windows Server 2025 ISO → save to `C:\ISOs\` on the laptop (about 6 GB). Note the product key in a text file.
3. Nothing else yet: the Ubuntu ISO and a second copy of the Server ISO are downloaded later *inside* HYPERVLAB.

**Education Hub route** (student): sign in at `azureforeducation.microsoft.com/devtools` with your university email → Software → *Windows Server 2025 Standard* → copy the key, click Download.

![Education Hub software catalogue with Windows Server 2025 Standard selected and the download panel open](images/P0-education-hub.png)

## Part 1 — Prepare the host laptop

VirtualBox can only pass virtualisation (nested VT-x) into a VM when Windows' own hypervisor is switched off. Without this step Hyper-V inside the server will install but never start a VM.

**1.1 Turn off Memory Integrity**

Settings → Privacy & security → Windows Security → Device security → Core isolation details → **Memory integrity: Off**.

![Core isolation page with Memory integrity Off](images/P1-01-memory-integrity.png)

**1.2 Remove the Windows hypervisor features**

Start → type *Turn Windows features on or off* → untick **Windows Hypervisor Platform** and **Virtual Machine Platform** → OK. (WSL2 stops working while these are off; nothing else is affected.)

![Windows Features with Virtual Machine Platform and Windows Hypervisor Platform unticked](images/P1-02-windows-features.png)

**1.3 Stop the hypervisor launching at boot** (HOST, PowerShell as administrator)

```powershell
bcdedit /set hypervisorlaunchtype off
```

Expect *The operation completed successfully.* Then **restart the laptop**.

**1.4 Verify** (HOST, PowerShell as administrator)

```powershell
bcdedit | findstr hypervisor
```

Must print `hypervisorlaunchtype    Off`. If it prints nothing or `Auto`, step 1.3 did not apply — run it again as administrator and reboot.

**1.5 Create folders**

```powershell
New-Item C:\ISOs, C:\VMs -ItemType Directory
```

Put the Windows Server 2025 ISO in `C:\ISOs\`.

**To reverse all of this** (for days you are not using the lab): `bcdedit /set hypervisorlaunchtype auto`, turn Memory Integrity back on, reboot. Reverse it again before the next lab session.

> **Why this matters.** Memory Integrity *is* Hyper-V — Windows runs itself inside a tiny hypervisor to protect the kernel. Only one hypervisor can own the CPU's VT-x at a time. Turning it off costs a small amount of host protection; keep Windows Update and Defender on and the real-world risk is low.

## Part 2 — Create the Windows Server VM in VirtualBox

This VM becomes HYPERVLAB. Two settings decide whether nested Hyper-V works at all: **Nested VT-x on** and **Paravirtualization Interface = None**. A third — launching pinned to the P-cores — decides whether it stays up.

**2.1 New VM** (VirtualBox → New)

| Field | Value |
|---|---|
| Name | `WinServer2025` |
| Folder | `C:\VMs` |
| ISO Image | `C:\ISOs\<windows server 2025>.iso` |
| OS Edition | Windows Server 2025 Standard **(Desktop Experience)** |
| Unattended Installation | **Untick** “Proceed with Unattended Installation” — install manually |
| Base memory | **6144 MB** (8192 crashed VirtualBox on this laptop — see Troubleshooting) |
| Processors | 4 |
| Enable EFI | Ticked |
| Hard disk | New, 60 GB, dynamic (not pre-allocated) |

![New VM wizard: name WinServer2025, folder C:\VMs, Server 2025 ISO, unattended install unticked](images/P2-01-wizard-name.png)

![New VM wizard hardware: 6144 MB, 4 CPUs, Use EFI ticked](images/P2-02-wizard-hardware.png)

![New VM wizard disk: new 60 GB VDI, not pre-allocated](images/P2-03-wizard-disk.png)

Finish, then open **Settings** on the new VM:

| Settings page | Change |
|---|---|
| System → Processor | Tick **Enable Nested VT-x/AMD-V** (greyed out = Part 1 did not take) |
| System → Acceleration | Paravirtualization Interface: **None** (Default makes Hyper-V inside refuse to start) |
| Display | Video memory 128 MB |
| Network → Adapter 1 | NAT |
| Audio | Untick Enable Audio |
| Shared Folders | None |

![Settings > System > Processor: 4 CPUs, Nested VT-x/AMD-V ticked](images/P2-04-settings-processor.png)

![Settings > System > Acceleration: Paravirtualization Interface None, Nested Paging ticked](images/P2-05-settings-acceleration.png)

![Settings > Display: 128 MB video memory, VBoxSVGA](images/P2-06-settings-display.png)

![Settings > Network: Adapter 1 NAT, cable connected](images/P2-07-settings-network.png)

![Settings > Audio: Enable Audio unticked](images/P2-08-settings-audio.png)

![Settings > Shared Folders empty and User Interface defaults](images/P2-09-settings-shared-ui.png)

**2.2 Confirm from the command line** (HOST)

```powershell
cd "C:\Program Files\Oracle\VirtualBox"                                            # VBoxManage lives here; it is VirtualBox's command line
.\VBoxManage modifyvm "WinServer2025" --nested-hw-virt on --paravirtprovider none   # force the two settings that matter, even if the GUI already shows them
.\VBoxManage showvminfo "WinServer2025" | findstr /i "nested paravirt memory cpus"  # print the VM's settings, keep only the four lines we care about
```

Want `Nested VT-x/AMD-V: enabled`, `Paravirt. Provider: None`, `Memory size: 6144MB`, `Number of CPUs: 4`.

![VBoxManage showvminfo: 6144 MB, 4 CPUs, Nested VT-x enabled, Paravirt provider None](images/P2-10-vboxmanage-check.png)

**2.3 The launcher — always start the VM this way**

On an Intel CPU with P-cores and E-cores (12th gen and later), VirtualBox's nested virtualisation crashes (“Guru Meditation”) when a vCPU lands on an E-core. Pin the VM to the P-cores at launch; it cannot be changed after launch.

Create `start-lab.cmd` on the desktop containing one line:

```cmd
start /affinity FF "" "C:\Program Files\Oracle\VirtualBox\VirtualBoxVM.exe" --startvm "WinServer2025"
```

`FF` = logical CPUs 0–7 = the four P-cores of an i7-1360P. Pin it to the taskbar and **never use the Start button in VirtualBox for this VM**. Verify after launch:

```powershell
Get-Process VirtualBoxVM | Select-Object Id, ProcessorAffinity   # expect 255
```

![Get-Process VirtualBoxVM showing ProcessorAffinity 255 on all three processes](images/P2-11-affinity-check.png)

## Part 3 — Install Windows Server 2025 and the Hyper-V role

**3.1 Install Windows** (start the VM with `start-lab.cmd`)

1. Press a key at *Press any key to boot from CD*.
2. Language/keyboard → Next → Install Now.
3. Product key: paste the Education Hub key (or *I don't have a product key* and activate later).
4. Image: **Windows Server 2025 Standard (Desktop Experience)**.
5. Accept licence → *Custom: Install Windows only* → select the 60 GB unallocated disk → Next. Windows creates its own partitions.
6. “Keep nothing” on the summary screen is correct for a blank disk → Install (10–15 min, two reboots).
7. Set the Administrator password. Log in with **Host key (Right Ctrl) + Del** when it asks for Ctrl+Alt+Del.

![Windows Server Setup: Select Image, Standard (Desktop Experience) highlighted](images/P3-01-select-image.png)

![Installing Windows Server progress screen](images/P3-02-installing.png)

**3.2 First-boot housekeeping** (HYPERVLAB, Server Manager → Local Server)

| Setting | Value |
|---|---|
| Computer name | `HYPERVLAB` (restart) |
| Time zone | Adelaide |
| Windows Update | Check and install now while NAT is connected |
| IE Enhanced Security Configuration | Off for Administrators |
| Activation | Settings → System → Activation → Change product key if not already active |

Guest Additions are optional: without them nothing pastes from the laptop into the server and every command is typed. Part 3.6 installs them with a one-way clipboard.

**3.3 Install Hyper-V** (HYPERVLAB, PowerShell as administrator)

```powershell
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart
```

The server restarts itself. A *Choose an option* recovery screen on the first reboot after this is normal — click **Continue**.

**3.4 Prove the hypervisor runs** (HYPERVLAB)

```powershell
Get-Service vmms, vmcompute          # both Running
bcdedit | findstr hypervisor         # hypervisorlaunchtype Auto
New-VM -Name Test -MemoryStartupBytes 512MB -Generation 2
Start-VM Test
Get-VM Test                          # State: Running = nested virtualisation works
Stop-VM Test -TurnOff
Remove-VM Test -Force
```

If `Start-VM` fails with *one of the Hyper-V components is not running*, the hypervisor did not start at boot: shut down and re-check Paravirtualization = None (Part 2) and `hypervisorlaunchtype Off` on the HOST (Part 1).

![Inside HYPERVLAB: vmms and vmcompute Running, hypervisorlaunchtype Auto, a test VM created, started and removed](images/P3-04-hyperv-test-vm.png)

**3.5 Checkpoint.** Shut HYPERVLAB down cleanly (Start → Power → Shut down). VirtualBox → Snapshots → Take → `clean-base-hyperv`. If you are doing 3.6 (Guest Additions), do it first and take this snapshot after it.

> **Shutdown Event Tracker.** Windows Server asks why it shut down unexpectedly after any hard stop (snapshot of a running VM, a crash). Type `lab` and OK. It is only a log entry.

**3.6 Guest Additions for copy and paste** (optional, HYPERVLAB running, no nested VMs)

Without Guest Additions nothing pastes from the laptop into the HYPERVLAB window, so every command is typed. One-way clipboard sharing fixes that and keeps the isolation: the server can receive the laptop's clipboard but never read it.

1. VirtualBox window menu: **Devices → Insert Guest Additions CD image**.
2. Inside HYPERVLAB: File Explorer → the *VirtualBox Guest Additions* DVD → run `VBoxWindowsAdditions.exe`, accept the defaults, let it reboot.
3. After the reboot, VirtualBox window menu: **Devices → Shared Clipboard → Host to Guest**. Leave **Drag and Drop** at Disabled; add no shared folders.
4. Test: copy a line on the laptop, right-click in HYPERVLAB's PowerShell window. It pastes.

The window also resizes and goes full-screen cleanly from now on. Re-run the 2.2 check once; nested VT-x and the paravirtualization setting are unchanged. Take the `clean-base-hyperv` snapshot after this step, not before, so every restore keeps it.

Pasting into a nested VM: use PowerShell Direct from HYPERVLAB (Part 11.4), where normal paste works. The vmconnect console only offers **Clipboard → Type clipboard text**, which is slow but fine for short commands.

## Part 4 — Add the L: lab data disk

The 60 GB system disk is for Windows and ISOs only; VMs, templates, exports and scripts go on a separate 80 GB disk called **L:**. In the original build this was added late, after the C: drive filled up during an export. Doing it now avoids that.

**4.1 Add the disk in VirtualBox** (HYPERVLAB shut down)

VirtualBox → WinServer2025 → Settings → Storage → select *Controller: SATA* → the “add hard disk” icon → **Create** → VDI → dynamically allocated → **80 GB** → Finish. In the *Hard Disk Selector*, click the new disk under *Not Attached*, then **Choose** → OK.

![Hard Disk Selector: WinServer2025_1.vdi (80 GB) under Not Attached](images/P4-01-media-manager.png)

![Settings > Storage: WinServer2025_1.vdi attached on SATA Port 2](images/P4-01b-storage-attached.png)

Start HYPERVLAB with `start-lab.cmd`.

**4.2 Bring it online as L:** (HYPERVLAB)

```powershell
Get-Disk                                                               # lists the disks; the new 80 GB one is Number 1 and shows RAW
Initialize-Disk -Number 1 -PartitionStyle GPT                          # write a partition table (GPT = the modern type)
New-Partition -DiskNumber 1 -UseMaximumSize -DriveLetter L             # one partition using the whole disk, called L:
Format-Volume -DriveLetter L -FileSystem NTFS -NewFileSystemLabel LabData   # format it NTFS with the label LabData
Get-Volume L                                                           # L: now shows about 79.98 GB free
```

**4.3 Make L: the default home for VMs** (HYPERVLAB)

```powershell
New-Item L:\VMs, L:\VMs\Templates, L:\Scripts, C:\ISOs -ItemType Directory
Set-VMHost -VirtualMachinePath L:\VMs -VirtualHardDiskPath L:\VMs
Get-VMHost | Select-Object VirtualMachinePath, VirtualHardDiskPath
```

![Initialize-Disk, New-Partition and Format-Volume producing L: LabData, 79.98 GB NTFS](images/P4-03-format-L.png)

From now on, any VM created without a path lands under `L:\VMs`.

## Part 5 — Virtual switches

A Hyper-V VM has no network until it is plugged into a virtual switch. Three types exist; this lab creates one of each.

| Switch | Type | Who is on it | Purpose here |
|---|---|---|---|
| `lab-external` | External — bound to the VM's real NIC | VMs + HYPERVLAB + outside world (VirtualBox NAT → internet) | Downloads, activation, updates |
| `lab-internal` | Internal — HYPERVLAB gets a virtual NIC | VMs + HYPERVLAB | The lab network, 10.20.20.0/24 |
| `lab-private` | Private — VMs only | VMs | Fully isolated tests |

Build them least-invasive first: Private changes nothing, Internal adds a NIC to the server, External re-plumbs the server's real NIC (its network blips for a few seconds).

**5.1 Create all three** (HYPERVLAB)

```powershell
New-VMSwitch -Name lab-private  -SwitchType Private                                   # VMs only
New-VMSwitch -Name lab-internal -SwitchType Internal                                  # VMs + HYPERVLAB (adds a vEthernet adapter to the server)
New-VMSwitch -Name lab-external -NetAdapterName Ethernet -AllowManagementOS $true     # bound to the server's real NIC; AllowManagementOS lets HYPERVLAB keep using that NIC too
Get-VMSwitch | Format-Table Name, SwitchType                                          # all three listed
```

(GUI equivalent: Hyper-V Manager → Virtual Switch Manager. For External pick the *Intel PRO/1000 MT Desktop Adapter* — that is VirtualBox's NAT adapter as the server sees it.)

![Virtual Switch Manager: lab-private, Private network](images/P5-01-switch-private.png)

![Virtual Switch Manager: lab-internal, Internal network](images/P5-01b-switch-internal.png)

![Virtual Switch Manager: lab-external, External network on the Intel PRO/1000 MT adapter, management OS sharing ticked](images/P5-01c-switch-external.png)

**5.2 Give the server an address on lab-internal** (HYPERVLAB)

```powershell
Get-NetAdapter                             # two new vEthernet adapters appear
New-NetIPAddress -InterfaceAlias "vEthernet (lab-internal)" -IPAddress 10.20.20.1 -PrefixLength 24
```

![Get-NetAdapter listing the vEthernet adapters, then New-NetIPAddress 10.20.20.1/24 on vEthernet (lab-internal)](images/P5-02-server-ip.png)

**5.3 Let the server answer pings** (HYPERVLAB)

Windows Server blocks inbound ping by default. Every “I can't ping the server” in this lab was this rule missing.

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4 In" -Protocol ICMPv4 -Direction Inbound -Action Allow
```

**5.4 Checkpoint.** `Get-VMSwitch` shows three switches.

![Allow ICMPv4 In rule created, then Get-VMSwitch showing lab-private, lab-internal and lab-external](images/P5-04-firewall-switches.png)

## Part 6 — Build the nested VMs

Two VMs: a Windows Server 2025 **Core** box (no desktop, managed by `sconfig` and PowerShell — the real job skill) and an Ubuntu Server box. Build them one at a time; HYPERVLAB's 6 GB cannot run both plus itself comfortably.

**6.1 Get the ISOs into HYPERVLAB**

The server needs its own copies (a VirtualBox-mounted ISO shows up as an unpacked DVD, which Hyper-V cannot boot). In the server's Edge, download to `C:\ISOs\`:

- Windows Server 2025 Standard from the Education Hub (same file as before), **or** the evaluation ISO from `microsoft.com/evalcenter` → pick the file containing `SERVER_EVAL_x64FRE_en-us`. Not the `LOF_PACKAGES` one.
- Ubuntu Server 26.04.x from `ubuntu.com/download/server`.

One download at a time; roughly 40 min each on NAT. Verify sizes afterwards (Windows ~6.4 GB, Ubuntu ~3 GB):

```powershell
Get-Item C:\ISOs\*.iso | Select-Object Name, Length
```

**6.2 WS-CORE01 — Windows Server Core** (HYPERVLAB, Hyper-V Manager → New → Virtual Machine)

| Wizard page | Value |
|---|---|
| Name / location | `WS-CORE01`, tick *Store in a different location* → `L:\VMs` |
| Generation | **2** |
| Memory | 2048 MB for the install (setup fails at 1 GB), **untick** Dynamic Memory |
| Network | `lab-external` (for install/activation; moved to lab-internal in Part 7) |
| Disk | New, `WS-CORE01.vhdx`, 40 GB |
| Install options | Install from bootable image → `C:\ISOs\<server>.iso` |

![New Virtual Machine Wizard: name WS-CORE01, stored in L:\VMs](images/P6-02a-wizard-name.png)

![Specify Generation: Generation 2](images/P6-02b-generation.png)

![Assign Memory, Dynamic Memory unticked. Shown at 1024 MB, which failed at install; use 2048 MB](images/P6-02c-memory.png)

![Configure Networking: lab-external](images/P6-02d-network.png)

![Connect Virtual Hard Disk: WS-CORE01.vhdx, 40 GB, under L:\VMs\WS-CORE01\Virtual Hard Disks](images/P6-02e-disk.png)

![Installation Options: install from the Server 2025 ISO](images/P6-02f-iso.png)

![Wizard summary before Finish](images/P6-02g-summary.png)

Then Settings → Processor → **2**. Connect → Start → click in the window and **tap spacebar repeatedly** to catch *Press any key*. If you see *Start PXE over IPv4* or a *Boot Summary*, you missed it: Action → Reset and try again.

![Windows Boot Manager after catching Press any key](images/P6-02h-boot-manager.png)

![Select setup option: Install Windows Server](images/P6-02i-setup-option.png)

![Installing Windows Server inside the nested VM](images/P6-02j-installing.png)

Install **Windows Server 2025 Standard** (the entry *without* Desktop Experience). Set the Administrator password; you land in `sconfig`.

![First boot: sconfig with the random WIN- computer name](images/P6-02k-sconfig-first.png)

In sconfig: `2` → `WS-CORE01` → restart. Then `15` for PowerShell.

![sconfig after the rename: computer name WS-CORE01](images/P6-02l-sconfig-renamed.png)

Once the password is set, shut the VM down and drop it to its running size: `Set-VMMemory WS-CORE01 -StartupBytes 1024MB`. The same applies to every Windows Server install in this lab: 2 GB to install, 1 GB to run. Ubuntu Server installs fine at its running size of 1.5 GB.

**6.3 UBU01 — Ubuntu Server** (same wizard)

| Wizard page | Value |
|---|---|
| Name / location | `UBU01`, `L:\VMs` |
| Generation | **2** |
| Memory | **1536 MB**, Dynamic Memory **off** (1 GB kernel-panics; dynamic memory starved it) |
| Network | `lab-internal` |
| Disk | New, 20 GB |
| Install options | Install from bootable image → Ubuntu ISO |

Settings → Security → Secure Boot template: **Microsoft UEFI Certificate Authority** (Ubuntu will not boot with the Windows template). Processor → 2. Or in PowerShell:

```powershell
Set-VMFirmware UBU01 -SecureBootTemplate MicrosoftUEFICertificateAuthority
Set-VM UBU01 -StaticMemory -MemoryStartupBytes 1536MB -ProcessorCount 2
```

![UBU01 settings: Secure Boot enabled with the Microsoft UEFI Certificate Authority template, 1536 MB, lab-internal](images/P6-03a-secure-boot.png)

Installer choices (keyboard only: arrows, Tab, Space, Enter):

1. English → US keyboard → *Ubuntu Server* (not minimized).
2. Network: `eth0` shows *autoconfiguration failed* (no DHCP on lab-internal — expected). Enter on eth0 → Edit IPv4 → **Manual** → Subnet `10.20.20.0/24`, Address `10.20.20.10`, gateway and DNS blank → Save → Done.
3. Proxy blank. Mirror: it cannot be checked — Done.
4. Storage: *Use an entire disk*, LVM ticked → Done → Done → Continue.
5. Profile: server name `ubu01`, a username and password (this build used `wick`).
6. Skip Ubuntu Pro. **Tick Install OpenSSH server.** No snaps.
7. Reboot Now → log in.

![Ubuntu installer: Ubuntu Server (not minimized)](images/P6-03b-ubuntu-type.png)

![Ubuntu installer: eth0 IPv4 set to Manual, subnet 10.20.20.0/24, address 10.20.20.10](images/P6-03c-ubuntu-net-edit.png)

![Ubuntu installer: eth0 static 10.20.20.10/24 confirmed](images/P6-03d-ubuntu-net-done.png)

![Ubuntu installer: guided storage, use the entire disk](images/P6-03e-ubuntu-storage.png)

![Ubuntu installer: profile, server name ubu01, username wick](images/P6-03f-ubuntu-profile.png)

![Ubuntu installer: Install OpenSSH server ticked](images/P6-03g-ubuntu-ssh.png)

![Ubuntu installer: installation complete, Reboot Now](images/P6-03h-ubuntu-complete.png)

If the static address did not stick, set it afterwards: `sudo nano /etc/netplan/50-cloud-init.yaml` with

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 10.20.20.10/24
```

then `sudo netplan apply`. Confirm with `ip a`.

**6.4 Memory rules that came out of this build**

| VM | Memory | Why |
|---|---|---|
| WS-CORE01 / WS-CORE02 | 1024 MB static | Core is happy at 1 GB. With the domain: 1280 MB for the DC and 768 MB for WS-CORE02 (Part 15) |
| UBU01 | 1536 MB static | Ubuntu 26.04 panics at 1 GB; Dynamic Memory shrank it to 512 MB and killed services |
| HYPERVLAB itself | ~3 GB of its 6 GB | Leaves ~3 GB: one nested VM at a time, two only for connectivity tests |

A *Kernel panic … deadlocked on memory* in Ubuntu or *Not enough memory to start* in Hyper-V both mean the same thing: stop the other VM first.

## Part 7 — Connectivity tests

Three tests prove what each switch type does. Expected results are the point; a ping that succeeds for the wrong reason is covered at the end.

**Test A — WS-CORE01 on ****lab-external** (inside WS-CORE01)

```powershell
Test-NetConnection 1.1.1.1        # True  — internet through VirtualBox NAT
Test-NetConnection 10.20.20.10    # should be False — UBU01 is on a different switch
```

![Test A inside WS-CORE01 on lab-external: 1.1.1.1 replies, 10.20.20.10 reachable](images/P7-testA.png)

**Test B — move WS-CORE01 to lab-internal**

HYPERVLAB:

```powershell
Connect-VMNetworkAdapter -VMName WS-CORE01 -SwitchName lab-internal
Get-VMNetworkAdapter WS-CORE01 | Select-Object SwitchName
```

Inside WS-CORE01:

```
New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 10.20.20.20 -PrefixLength 24
New-NetFirewallRule -DisplayName "Allow ICMPv4 In" -Protocol ICMPv4 -Direction Inbound -Action Allow
Test-NetConnection 10.20.20.10       # replies in 0–1 ms = UBU01
Test-NetConnection 10.20.20.1        # replies = HYPERVLAB
Test-NetConnection 1.1.1.1           # fails = no route off lab-internal
```

![Allow ICMPv4 In rule created inside WS-CORE01](images/P7-testB-firewall.png)

![Test B inside WS-CORE01 on lab-internal: 10.20.20.10 and 10.20.20.1 succeed, 1.1.1.1 fails](images/P7-testB-results.png)

Inside UBU01: `ping -c 3 10.20.20.20` replies.

![Inside UBU01: ip a shows 10.20.20.10/24 and ping 10.20.20.20 replies](images/P7-testB-ubuntu-ping.png)

**Test C — both on lab-private**

Optional but two minutes, and the only test that shows what a Private switch does: VMs on it see each other and nothing else, HYPERVLAB included. The failed test below is the expected result, not a fault. Both VMs stay running throughout.

HYPERVLAB:

```powershell
Connect-VMNetworkAdapter -VMName WS-CORE01 -SwitchName lab-private
Connect-VMNetworkAdapter -VMName UBU01     -SwitchName lab-private
Test-NetConnection 10.20.20.10    # DestinationHostUnreachable — the server is locked out
```

![Test C on HYPERVLAB: both VMs moved to lab-private, ping to 10.20.20.10 times out](images/P7-testC.png)

Inside WS-CORE01: `Test-NetConnection 10.20.20.10` still replies — VMs see each other, nothing else does.

Put both back on HYPERVLAB. This step is not optional: left on lab-private, nothing from Part 8 onwards can reach the VMs and it looks like a mystery fault later.

```powershell
Connect-VMNetworkAdapter -VMName WS-CORE01 -SwitchName lab-internal
Connect-VMNetworkAdapter -VMName UBU01     -SwitchName lab-internal
```

`Test-NetConnection 10.20.20.10` from HYPERVLAB succeeds again. Worth one screenshot: the failed test on HYPERVLAB beside the working ping inside WS-CORE01.

In Tests A to C, `Test-NetConnection` and `ping` are used rather than `Test-Connection`: on Windows PowerShell 5.1 the latter fails with *a non-recoverable error occurred during a database lookup* because there is no DNS in the lab until Part 12. *Destination host unreachable* from a VM's own address means the target VM is off or on a different switch, not a problem on the VM running the ping.

| From → To | lab-external | lab-internal | lab-private |
|---|---|---|---|
| VM → internet | Yes | No | No |
| VM → HYPERVLAB (10.20.20.1) | No | Yes | No |
| VM → other VM on same switch | Yes | Yes | Yes |
| HYPERVLAB → VM | Yes | Yes | No |

> **The false positive.** In Test A, `Test-NetConnection 10.20.20.10` returned *True* — with UBU01 switched off. VirtualBox's NAT forwarded the ping out to the internet and a device on the ISP's private network with that same address answered, 33 ms away. Clues: one hop, internet-scale latency, and the target VM was off. Same-switch replies are under 1 ms. A ping proves *something* answered, not that the right thing did — check the RTT, trace the route, or confirm the target saw the traffic.

**Checkpoint.** Shut everything down; VirtualBox snapshot `phase3-vms-built`.

## Part 8 — Virtual disks

Use **VHDX** always (64 TB, crash-resilient); VHD is legacy. **Dynamic** grows as used, **fixed** takes its full size up front, **differencing** is a child that stores only changes from a parent — the basis of checkpoints and templates.

**8.1 Hot-add a data disk** (HYPERVLAB, WS-CORE01 running)

```powershell
New-VHD -Path L:\VMs\WS-CORE01\data01.vhdx -SizeBytes 10GB -Dynamic   # create the disk file: 10 GB, grows as used
Add-VMHardDiskDrive -VMName WS-CORE01 -Path L:\VMs\WS-CORE01\data01.vhdx   # plug it into the running VM (no reboot needed)
Get-VMHardDiskDrive WS-CORE01                                          # two disks now: the OS disk at location 0, data01 at location 2
```

![New-VHD data01.vhdx 10 GB dynamic, added to WS-CORE01 at SCSI location 2](images/P8-01-new-vhd.png)

Inside WS-CORE01 (D: is taken by the DVD drive, so the data disk becomes E:):

```powershell
Get-Disk                                           # new disk is Number 1
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -DriveLetter E
Format-Volume -DriveLetter E -FileSystem NTFS -NewFileSystemLabel Data
Get-Volume
```

![Inside WS-CORE01: Get-Disk shows the 10 GB RAW disk, initialised GPT and partitioned as E:](images/P8-01b-init-disk.png)

![Format-Volume E: Data, Get-Volume listing C:, D: (DVD) and E:](images/P8-01c-format-E.png)

**8.2 Resize online** — two steps, disk then partition, same as in Azure

HYPERVLAB: `Resize-VHD -Path L:\VMs\WS-CORE01\data01.vhdx -SizeBytes 20GB`

Inside WS-CORE01:

```powershell
Resize-Partition -DriveLetter E -Size (Get-PartitionSupportedSize -DriveLetter E).SizeMax
Get-Volume E
```

![Resize-Partition E: to the supported maximum, now 19.95 GB](images/P8-02-resize.png)

**8.3 Detach and re-attach — data survives**

Inside WS-CORE01: `"hello" | Out-File E:\test.txt` then `Set-Disk -Number 1 -IsOffline $true`.

HYPERVLAB (check the ControllerLocation first with `Get-VMHardDiskDrive WS-CORE01`):

```powershell
Remove-VMHardDiskDrive -VMName WS-CORE01 -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 2
Add-VMHardDiskDrive -VMName WS-CORE01 -Path L:\VMs\WS-CORE01\data01.vhdx
```

Inside WS-CORE01: `Set-Disk -Number 1 -IsOffline $false; Get-Content E:\test.txt` → `hello`.

**8.4 Differencing disk** (HYPERVLAB, UBU01 shut down)

```powershell
Stop-VM UBU01
$parent = (Get-VMHardDiskDrive UBU01).Path
$parent                                              # echo it: the wizard puts disks under <VM>\Virtual Hard Disks\
New-VHD -Path L:\VMs\UBU01\UBU01-child.vhdx -ParentPath $parent -Differencing
Get-VMHardDiskDrive UBU01 | Set-VMHardDiskDrive -Path L:\VMs\UBU01\UBU01-child.vhdx
Get-VMHardDiskDrive UBU01 | Select-Object Path      # MUST show UBU01-child.vhdx before starting
Start-VM UBU01
```

![$parent echoed as L:\VMs\UBU01\Virtual Hard Disks\UBU01.vhdx, then New-VHD creating the differencing child](images/P8-04a-differencing.png)

![Set-VMHardDiskDrive to the child, Get-VMHardDiskDrive confirming it, Start-VM](images/P8-04b-attach-child.png)

Inside UBU01: `echo "child change" > ~/note.txt`. On HYPERVLAB, `Get-ChildItem L:\VMs\UBU01 | Select Name, Length` — the child is ~140 MB, the parent unchanged. Merge it back:

![Get-ChildItem L:\VMs\UBU01: the child is about 100 MB](images/P8-04c-child-size.png)

```powershell
Stop-VM UBU01
Merge-VHD -Path L:\VMs\UBU01\UBU01-child.vhdx
Get-VMHardDiskDrive UBU01 | Set-VMHardDiskDrive -Path $parent
Start-VM UBU01                                       # note.txt still there
```

![Merge-VHD confirmation, disk pointed back at $parent, Start-VM](images/P8-04d-merge.png)

> **Rule:** never start a VM on a parent while a child exists. Writing to the parent changes its ID and every child becomes unmergeable (*The chain of virtual hard disks is corrupted*). If it happens: discard the child (point the VM back at the parent, delete the child) or, if the child holds real data, `Set-VHD -Path <child> -ParentPath <parent> -IgnoreIdMismatch` then merge. The `Get-VMHardDiskDrive` check before `Start-VM` is what prevents it.

**8.5 Compact and convert** (HYPERVLAB, WS-CORE01 off)

```powershell
Optimize-VHD -Path L:\VMs\WS-CORE01\data01.vhdx -Mode Full              # compact: give back space freed inside the guest (dynamic disks grow but never shrink by themselves)
New-VHD -Path L:\VMs\test-dyn.vhdx -SizeBytes 2GB -Dynamic             # a 2 GB dynamic disk: the file is only 4 MB
Convert-VHD -Path L:\VMs\test-dyn.vhdx -DestinationPath L:\VMs\test-fixed.vhdx -VHDType Fixed   # copy it as a fixed disk: the file is the full 2 GB
Get-ChildItem L:\VMs\test-*.vhdx | Select-Object Name, Length           # 4 MB vs 2,151,677,952 bytes
Remove-Item L:\VMs\test-*.vhdx                                          # tidy up
```

![Optimize-VHD, then a 2 GB dynamic disk at 4 MB beside its fixed copy at 2,151,677,952 bytes](images/P8-05-compact-convert.png)

**Checkpoint.** Snapshot `phase4-disks`.

## Part 9 — Checkpoints, export/import, template and clone

**9.1 Checkpoints** are Hyper-V's snapshots, built on differencing disks. *Standard* saves disk and memory (instant; the VM wakes up never knowing it stopped — bad for databases and domain controllers). *Production* uses the guest's backup tooling (VSS / fsfreeze) and restores as a clean shutdown — the default and the one used at work.

HYPERVLAB, WS-CORE01 running:

```powershell
Set-VM WS-CORE01 -CheckpointType Standard
Checkpoint-VM -Name WS-CORE01 -SnapshotName "std-before-change"
Set-VM WS-CORE01 -CheckpointType Production
Checkpoint-VM -Name WS-CORE01 -SnapshotName "prod-before-change"
Get-VMCheckpoint WS-CORE01
$os = (Get-VMHardDiskDrive WS-CORE01 -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 0).Path
$os                                                  # now points at an .avhdx: each checkpoint is a differencing disk
Get-ChildItem (Split-Path $os)                       # WS-CORE01.vhdx plus one .avhdx per checkpoint
```

![9.1 — two checkpoints created; $os now points at an .avhdx and the folder shows one .avhdx per checkpoint](images/P9-01a-checkpoints-created.png)

![9.1 — Hyper-V Manager shows the checkpoint tree: std-before-change → prod-before-change → Now](images/P9-01b-checkpoint-tree.png)

Inside WS-CORE01: `"important" | Out-File C:\change.txt`. Then on HYPERVLAB:

```powershell
Restore-VMCheckpoint -Name "prod-before-change" -VMName WS-CORE01 -Confirm:$false
Start-VM WS-CORE01                                       # change.txt is gone
```

![9.1 — restore to prod-before-change, start, then remove all checkpoints](images/P9-01c-restore-delete.png)

Delete them and watch the merge (*Status: Merging disks*, then the `.avhdx` files disappear):

```powershell
Get-VMCheckpoint WS-CORE01 | Remove-VMCheckpoint
Get-VM WS-CORE01 | Select-Object Name, Status     # "Merging disks" for a few seconds, then "Operating normally"
Get-VMCheckpoint WS-CORE01                        # nothing listed
Get-ChildItem (Split-Path $os)                    # only WS-CORE01.vhdx: the .avhdx changes were folded in
```

![9.1 — after the merge: Hyper-V Manager shows no checkpoints](images/P9-01d-no-checkpoints.png)

Checkpoints left behind are the most common Hyper-V mess in real environments — chains 20 deep and storage full. Delete them when the change is confirmed good.

**9.2 Export and import.** An export is a full portable copy: config + disks + checkpoints. Import with `-Copy -GenerateNewId` is a clone; without `-GenerateNewId` it is a restore. This is also the correct way to move a VM to a new folder — never move VM folders by hand.

```powershell
Stop-VM WS-CORE01
Export-VM -Name WS-CORE01 -Path L:\Exports
Get-ChildItem L:\Exports\WS-CORE01 -Recurse -Filter *.vhdx | Select-Object FullName, Length   # expect BOTH disks: WS-CORE01.vhdx and data01.vhdx
Remove-VM WS-CORE01 -Force                               # removes the VM definition only; the disk files stay on L:
Rename-Item L:\VMs\WS-CORE01 L:\VMs\WS-CORE01-old         # move the old disks aside or Import-VM refuses: "data01.vhdx already exists"
$vmcx = Get-ChildItem L:\Exports\WS-CORE01 -Recurse -Filter *.vmcx | Select-Object -ExpandProperty FullName
$vmcx                                                    # always echo a variable before using it
Import-VM -Path $vmcx -Copy -GenerateNewId -VirtualMachinePath L:\VMs\WS-CORE01 -VhdDestinationPath "L:\VMs\WS-CORE01\Virtual Hard Disks"
Get-VM WS-CORE01
Get-VMHardDiskDrive WS-CORE01 | Select-Object ControllerLocation, Path   # both disks, now under the new folder
Start-VM WS-CORE01                                       # log in once to prove it boots
Remove-Item L:\VMs\WS-CORE01-old -Recurse -Force          # only after it boots
Remove-Item L:\Exports -Recurse -Force                   # only after it boots
```

![9.2 — export holds both disks; old folder renamed; import completes; both disks now under the new Virtual Hard Disks folder](images/P9-02-export-import.png)

An export needs free space equal to the VM; on a full disk it fails with *not enough disk space* (that is what forced the L: drive in Part 4).

**9.3 Sysprep a template.** Sysprep strips the machine-specific identity (SID, name, activation) so a disk can be cloned many times. It also resets the source VM — WS-CORE01 loses its name and IP and must be set up again afterwards.

Inside WS-CORE01:

```powershell
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
# /generalize = strip the machine identity (SID, name, activation)   /oobe = next boot runs first-run setup   /shutdown = power off when done
```

![9.3 — sysprep running inside WS-CORE01 (generalize phase)](images/P9-03a-sysprep.png)

HYPERVLAB, once WS-CORE01 is **Off** (copying a running disk gives a corrupt template):

```powershell
$os = (Get-VMHardDiskDrive WS-CORE01 -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 0).Path
$os                                                  # must end in WS-CORE01.vhdx: no checkpoints left, VM Off
New-Item L:\VMs\Templates -ItemType Directory -Force
Copy-Item $os L:\VMs\Templates\ws2025-core-template.vhdx
Set-ItemProperty L:\VMs\Templates\ws2025-core-template.vhdx -Name IsReadOnly -Value $true
```

![9.3 — what happens if the 9.2 leftovers are still on L: — Copy-Item: not enough space on the disk](images/P9-03b-copy-no-space.png)

![9.3 — Get-PSDrive L: 14.6 GB free; delete WS-CORE01-old and L:\Exports; 59 GB free](images/P9-03c-free-space.png)

**9.4 Clone from the template** — a differencing disk off the read-only master:

```powershell
New-Item L:\VMs\WS-CORE02 -ItemType Directory
New-VHD -Path L:\VMs\WS-CORE02\WS-CORE02.vhdx -ParentPath L:\VMs\Templates\ws2025-core-template.vhdx -Differencing   # child disk: stores only WS-CORE02's changes, reads everything else from the template
New-VM -Name WS-CORE02 -Generation 2 -MemoryStartupBytes 1024MB -VHDPath L:\VMs\WS-CORE02\WS-CORE02.vhdx -SwitchName lab-internal -Path L:\VMs   # the VM, using that child disk, on lab-internal
Set-VM WS-CORE02 -StaticMemory -ProcessorCount 2
Start-VM WS-CORE02
Get-ChildItem L:\VMs\WS-CORE02, L:\VMs\Templates | Select-Object Name, Length   # clone is a few hundred MB; the template is untouched
```

WS-CORE02 boots into first-run setup in about a minute instead of a 15-minute install. Set its password, then in sconfig `2` → `WS-CORE02` → restart. Then inside WS-CORE02's PowerShell, give it its address and let it answer ping (the clone came up blank — no IP, firewall blocking ping):

```powershell
New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 10.20.20.30 -PrefixLength 24
New-NetFirewallRule -DisplayName "Allow ICMPv4 In" -Protocol ICMPv4 -Direction Inbound -Action Allow
```

![9.4 — inside WS-CORE02: New-NetIPAddress 10.20.20.30](images/P9-04a-core02-ip.png)

![9.4 — inside WS-CORE02: Allow ICMPv4 In firewall rule created](images/P9-04b-core02-icmp.png)

**9.5 Restore WS-CORE01** after sysprep — it also came up blank. Set password → sconfig `2` → `WS-CORE01` → restart, then inside WS-CORE01's PowerShell:

```powershell
New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 10.20.20.20 -PrefixLength 24
New-NetFirewallRule -DisplayName "Allow ICMPv4 In" -Protocol ICMPv4 -Direction Inbound -Action Allow
ping 10.20.20.30                                  # replies = WS-CORE02 is up and reachable
```

![9.5 — inside WS-CORE01: ping 10.20.20.30 gets four replies from WS-CORE02](images/P9-05-core01-ping.png)

**Checkpoint.** Snapshot `phase5-checkpoints-clone`.

## Part 10 — Resources and tuning

All on HYPERVLAB, WS-CORE02 as the test subject, one nested VM running.

**10.1 Dynamic memory, done properly.** Fine for Windows guests when the minimum is sensible; the Ubuntu crash in Part 6 was a 512 MB minimum.

```powershell
Stop-VM WS-CORE02
Set-VMMemory WS-CORE02 -DynamicMemoryEnabled $true -MinimumBytes 768MB -StartupBytes 1024MB -MaximumBytes 1536MB -Buffer 20
Start-VM WS-CORE02
Get-VM WS-CORE02 | Select-Object MemoryAssigned, MemoryDemand, MemoryStatus
```

![10.1 — dynamic memory set 768 MB – 1.5 GB on WS-CORE02](images/P10-01-dynamic-memory.png)

(The buffer lives on `Set-VMMemory`, not `Set-VM`. `Get-Help Set-VMMemory -Parameter *` lists what each cmdlet accepts.)

**10.2 CPU reserve, l****i****m****i****t** **a****n****d** **w****e****i****g****h****t**

```powershell
Set-VMProcessor WS-CORE02 -Reserve 10 -Maximum 50 -RelativeWeight 200
Get-VMProcessor WS-CORE02 | Select-Object Count, Reserve, Maximum, RelativeWeight
```

![10.2 — CPU reserve 10, maximum 50, weight 200 on WS-CORE02](images/P10-02-cpu-weight.png)

Reserve = guaranteed %, Maximum = cap, RelativeWeight = who wins under contention (default 100; production 200, test boxes 50).

**10.3 Boot order and automatic start/stop**

```powershell
Get-VMFirmware WS-CORE02 | Select-Object -ExpandProperty BootOrder
Stop-VM WS-CORE02                                        # start/stop actions can only be changed while the VM is Off
Set-VM WS-CORE01 -AutomaticStartAction Start -AutomaticStartDelay 0  -AutomaticStopAction ShutDown
Set-VM WS-CORE02 -AutomaticStartAction StartIfRunning -AutomaticStartDelay 60 -AutomaticStopAction ShutDown
Get-VM | Select-Object Name, AutomaticStartAction, AutomaticStartDelay, AutomaticStopAction
Start-VM WS-CORE02
```

![10.3 — boot order, then automatic start/stop actions set with WS-CORE02 Off](images/P10-03-start-stop.png)

Real hosts: domain controllers first with delay 0, everything else staggered; stop action *ShutDown*, not *Save*, for anything with a database.

**10.4 Integration services**

```powershell
Get-VMIntegrationService WS-CORE02
Get-VMIntegrationService UBU01
```

![10.4 — integration services: WS-CORE02 all OK, UBU01 missing the status column; Guest Service Interface False on both](images/P10-04-integration.png)

Integration services are the small agents inside a guest that let Hyper-V talk to it (heartbeat, time sync, clean shutdown, KVP = the guest reporting its IP and name, VSS, file copy). WS-CORE02: all `OK` except **Guest Service Interface** `Enabled: False` — that one is off by default on every VM. UBU01: same, plus some *No Contact* / *Not working* because Ubuntu needs the `hyperv-daemons` package and UBU01 has no internet. Expected, not a fault. Side effect: `Get-VM | Get-VMNetworkAdapter | Select-Object VMName, IPAddresses` is blank for UBU01 (no KVP agent to report it) and for any VM that is Off — look inside the VM instead.

**10.5 Enhanced Session Mode.** RDP over VMBus: resolution picker, clipboard, drive redirection. Needs the host setting, the user setting, and RDP on in the guest (sconfig `7` on Core).

```powershell
Set-VMHost -EnableEnhancedSessionMode $true
```

Hyper-V Manager → Hyper-V Settings → User → Enhanced Session Mode → tick *Use enhanced session mode*. Result in this build: worked on a fresh template clone (WS-CORE03), never on WS-CORE02. Expect it to be fussy on Server Core and reliable on Desktop Experience guests.

**10.6 Nested virtualisation flag — look, don't use**

```powershell
Stop-VM WS-CORE02
Set-VMProcessor WS-CORE02 -ExposeVirtualizationExtensions $true
Get-VMProcessor WS-CORE02 | Select-Object ExposeVirtualizationExtensions
Set-VMProcessor WS-CORE02 -ExposeVirtualizationExtensions $false
```

![10.6 — nested virtualisation exposed to WS-CORE02, then turned back off](images/P10-05-nested-virt.png)

On real hardware the full recipe for a nested host is: this flag, static memory, and `Set-VMNetworkAdapter -MacAddressSpoofing On`. Here it would be a third layer under VirtualBox and will crash it — set it back to `$false`.

**Checkpoint.** Snapshot `phase6-tuning`.

## Part 11 — PowerShell automation and PowerShell Direct

Everything in Parts 5–10 was already PowerShell; this part turns it into scripts, which is what separates clicking through Hyper-V Manager from managing many hosts.

**11.1 Know the module**

```powershell
Get-Command -Module Hyper-V | Measure-Object        # ~240 cmdlets
Get-Command -Module Hyper-V -Noun VM*
Get-Help New-VM -Examples
```

All Verb-Noun. Guess the noun (`VMSwitch`, `VMHardDiskDrive`, `VMCheckpoint`) and `Get-Command -Noun X` finds the rest.

**11.2 Pipelines and bulk work**

```powershell
Get-VM | Where-Object State -eq Running | Select-Object Name, Uptime, MemoryAssigned        # only the running VMs, three columns
Get-VM | Get-VMNetworkAdapter | Select-Object VMName, SwitchName, IPAddresses              # which switch each VM is on, and the IP it reports
Get-VM | Get-VMHardDiskDrive | Get-VHD | Select-Object Path, VhdType, @{n='GB';e={[math]::Round($_.FileSize/1GB,2)}}   # every disk file with its type and real size on L:
Get-VM WS-CORE* | Checkpoint-VM -SnapshotName "bulk-test"                                  # one checkpoint on every WS-CORE VM at once
Get-VM WS-CORE* | Get-VMCheckpoint | Remove-VMCheckpoint                                   # and remove them all again
```

![11.2 — one-liners: running VMs, adapters and IPs (UBU01 and the Off VM blank), disk sizes, bulk checkpoint](images/P11-01-oneliners.png)

![11.2 — bulk remove of the checkpoints](images/P11-01b-bulk-remove.png)

![11.2 — Hyper-V Manager: no checkpoints left](images/P11-01c-no-checkpoints.png)

**11.3 Script a full VM build** — save as `L:\Scripts\New-LabVM.ps1` (HYPERVLAB, Notepad). With Guest Additions (Part 3.6) you can copy the script from the laptop and paste it into Notepad; in Save As set the type to All Files so it is not saved as .txt.

```powershell
param(
    [Parameter(Mandatory)] [string]$Name,
    [string]$SwitchName = "lab-internal",
    [int64]$MemoryMB = 1024,
    [int]$Cpu = 2,
    [string]$Template = "L:\VMs\Templates\ws2025-core-template.vhdx"
)

$vmPath  = "L:\VMs\$Name"
$vhdPath = "$vmPath\$Name.vhdx"

if (Get-VM -Name $Name -ErrorAction SilentlyContinue) {
    throw "VM '$Name' already exists."
}

New-Item -Path $vmPath -ItemType Directory -Force | Out-Null
New-VHD -Path $vhdPath -ParentPath $Template -Differencing | Out-Null
New-VM -Name $Name -Generation 2 -MemoryStartupBytes ($MemoryMB * 1MB) -VHDPath $vhdPath -SwitchName $SwitchName -Path "L:\VMs" | Out-Null
Set-VM -Name $Name -StaticMemory -ProcessorCount $Cpu -AutomaticStopAction ShutDown -CheckpointType Production
Start-VM -Name $Name
Get-VM -Name $Name | Select-Object Name, State, MemoryAssigned, ProcessorCount
```

What the script does, top to bottom: `param(...)` names the inputs — only `-Name` is required, the rest have defaults (switch, 1024 MB, 2 CPUs, the template path). It refuses to run if a VM of that name already exists. Then it is Part 9.4 in five lines: make the folder, make a differencing disk off the template, create the VM on that disk, set static memory / CPUs / Production checkpoints, start it, and print the result. `| Out-Null` just hides the chatter from each command.

![11.3 — Notepad, Save As L:\Scripts\New-LabVM.ps1 (type All Files or quote the name so it is not saved as .txt)](images/P11-03a-notepad-save.png)

Run it:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
L:\Scripts\New-LabVM.ps1 -Name WS-CORE03
```

![11.3 — one command builds and starts WS-CORE03](images/P11-03b-run-script.png)

![11.3 — Hyper-V Manager: WS-CORE03 running beside the others](images/P11-03c-core03-running.png)

A full server from the template in one command. Note the parameter is `$SwitchName`, not `$Switch` — `$switch` is a reserved PowerShell variable and comes out empty. A failed half-run leaves a folder behind; delete `L:\VMs\<Name>` before retrying.

The script starts the VM, but a template clone boots into first-run setup and waits there. Before 11.4, open WS-CORE03's console in Hyper-V Manager, set the Administrator password and let it reach the login screen — until then there is no password, so PowerShell Direct answers *The credential is invalid*. (Memory: WS-CORE03 needs 1 GB free; stop WS-CORE02 first if `Start-VM` complains.)

**11.4 PowerShell Direct** — manage a VM from the host over VMBus, no network needed. Build the credential without the pop-up dialog (it hides behind Terminal):

```powershell
# ONE LINE AT A TIME. Pasting the block whole feeds the next lines into Read-Host as the password.
$pw   = Read-Host "WS-CORE03 password" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential("Administrator", $pw)
Invoke-Command -VMName WS-CORE03 -Credential $cred -ScriptBlock { hostname; Get-NetIPAddress -AddressFamily IPv4 | Select-Object IPAddress }
Enter-PSSession -VMName WS-CORE03 -Credential $cred     # prompt becomes [WS-CORE03]:  — exit to leave
```

![11.4 — PowerShell Direct from HYPERVLAB: hostname and IPs of WS-CORE03 (169.254 = APIPA), then Enter-PSSession](images/P11-04-psdirect.png)

The result in this build showed `169.254.144.241` — APIPA. Any address starting 169.254 means “I asked DHCP, nobody answered, I made one up.” It is the first thing to look for on any “no network” ticket. Fix it from HYPERVLAB's PowerShell window, without opening WS-CORE03's console — the command runs inside WS-CORE03 over PowerShell Direct:

```powershell
exit                                                     # if the prompt still says [WS-CORE03]: — $cred only exists on HYPERVLAB
Invoke-Command -VMName WS-CORE03 -Credential $cred -ScriptBlock { New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 10.20.20.40 -PrefixLength 24 }
Invoke-Command -VMName WS-CORE03 -Credential $cred -ScriptBlock { Get-NetIPAddress -AddressFamily IPv4 | Select-Object IPAddress }   # 10.20.20.40
```

**11.5 Clean up** — three Windows VMs is more than the lab needs:

```powershell
Stop-VM WS-CORE03 -TurnOff
Remove-VM WS-CORE03 -Force
Remove-Item L:\VMs\WS-CORE03 -Recurse -Force
Get-VM                                      # UBU01, WS-CORE01, WS-CORE02
```

If you see `>>` in PowerShell, a bracket or quote is still open — Ctrl+C and retype.

**Checkpoint.** Snapshot `phase7-powershell`.

## Part 12 — Active Directory: build the domain controller

WS-CORE01 becomes the domain controller (DC) for a new domain, `lab.local` (short name `LAB`), with DNS on the same server. It is Server Core, so the build is two PowerShell commands: install the role, then promote. Keep UBU01 off throughout.

| Item | Value |
|---|---|
| Domain (DNS name) | `lab.local` |
| Short (NetBIOS) name | `LAB` |
| Domain controller + DNS | WS-CORE01, 10.20.20.20 |
| Domain admin account | `LAB\Administrator` (same password WS-CORE01's local Administrator had) |

**12.1 Give WS-CORE01 fixed memory** (HYPERVLAB)

Memory can only be changed while the VM is off.

```powershell
Stop-VM UBU01, WS-CORE02, WS-CORE01 -ErrorAction SilentlyContinue   # all three Off (ignore errors for VMs already Off)
Set-VMMemory WS-CORE01 -DynamicMemoryEnabled $false -StartupBytes 1280MB   # the DC: fixed 1280 MB
Set-VMMemory WS-CORE02 -DynamicMemoryEnabled $false -StartupBytes 768MB    # the member server: fixed 768 MB (used from Part 15)
Get-VM | Select-Object Name, State, MemoryStartup
```

All three show *Off*, WS-CORE01 `1342177280` (1280 MB) and WS-CORE02 `805306368` (768 MB). These are the only two sizes that let the DC and the member server run together on a 6 GB HYPERVLAB (Part 15.1 explains why). The first build used 2 GB here — that is what the screenshot shows — and had to drop to 1280 MB later.

![Get-VM output: all three VMs Off, WS-CORE01 at 2 GB](images/AD-01-memory.png)

**12.2 Safety checkpoint, then start** (HYPERVLAB)

```powershell
Checkpoint-VM -Name WS-CORE01 -SnapshotName "pre-AD"   # restore point: undo the whole DC build with one Restore-VMCheckpoint
Get-VMCheckpoint WS-CORE01                             # pre-AD listed
Start-VM WS-CORE01
Get-VM WS-CORE01 | Select-Object Name, State, MemoryAssigned
```

In `Checkpoint-VM`, `-Name` is the VM and `-SnapshotName` is the checkpoint. Swapping them gives *cannot bind parameter*.

![pre-AD checkpoint listed, WS-CORE01 Running with 2 GB](images/AD-02-checkpoint.png)

**12.3 Connect and confirm name and IP** (HYPERVLAB → inside WS-CORE01)

A DC's name and IP are very hard to change later, so check them before promoting. Run the lines one at a time: pasted together, the password prompt swallows the next line.

```powershell
# ONE LINE AT A TIME (Read-Host)
$pw = Read-Host "WS-CORE01 password" -AsSecureString                                   # type WS-CORE01's Administrator password; it is stored encrypted in $pw
$cred = New-Object System.Management.Automation.PSCredential("Administrator", $pw)     # username + password packed into one credential object
Enter-PSSession -VMName WS-CORE01 -Credential $cred                                   # PowerShell Direct: the prompt changes to [WS-CORE01]:
hostname
Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias Ethernet | Select-Object IPAddress, PrefixLength
```

Expect `WS-CORE01` and `10.20.20.20` / `24`. This guide builds the credential with `Read-Host` because the `Get-Credential` popup fails on HYPERVLAB (see Troubleshooting).

![Inside WS-CORE01: hostname WS-CORE01, IP 10.20.20.20/24](images/AD-03-name-ip.png)

**12.4 Point DNS at itself** (inside WS-CORE01)

Active Directory depends on DNS, and this server is about to be the lab's DNS server.

```powershell
Set-DnsClientServerAddress -InterfaceAlias Ethernet -ServerAddresses 10.20.20.20
Get-DnsClientServerAddress -InterfaceAlias Ethernet -AddressFamily IPv4
```

![DNS server address on WS-CORE01 set to 10.20.20.20](images/AD-04-dns.png)

**12.5 Install the AD DS role** (inside WS-CORE01)

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Get-WindowsFeature AD-Domain-Services, RSAT-AD-PowerShell
```

Want `Success True`, `Restart Needed No`, and both features `[X]`. This only copies the software; the server is not a DC yet.

![AD DS role install: Success True, both features installed](images/AD-05-role-install.png)

**12.6 Promote to domain controller** (inside WS-CORE01)

The first line sets the Directory Services Restore Mode (DSRM) password, used only to repair a broken DC. Make one up (8+ characters, mixed case and a number) and record it in your lab notes.

```powershell
# ONE LINE AT A TIME (Read-Host)
$dsrm = Read-Host "DSRM password" -AsSecureString
Install-ADDSForest -DomainName lab.local -DomainNetbiosName LAB -InstallDns -SafeModeAdministratorPassword $dsrm -Force
# -DomainName = the DNS name of the new domain   -DomainNetbiosName = its short name   -InstallDns = make this server the DNS server too
# -SafeModeAdministratorPassword = the DSRM password from the line above   -Force = do not ask for confirmation
```

It runs 5–10 minutes. Two yellow *DNS delegation* warnings are normal in a lab. It ends with `Status: Success`.

![Install-ADDSForest finishing with Status Success and the two normal DNS delegation warnings](images/AD-06-promote.png)

The reboot normally follows by itself. If the prompt still responds, run `Restart-Computer -Force`. Then wait about 5 minutes: the first boot as a DC is slow.

**12.7 Sign in with the domain account** (HYPERVLAB)

The local Administrator of WS-CORE01 has become the domain's Administrator. Same password, new name: `LAB\Administrator`.

```powershell
# ONE LINE AT A TIME (Read-Host)
$pw = Read-Host "LAB admin password" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential("LAB\Administrator", $pw)   # same password as before, domain account name now
Enter-PSSession -VMName WS-CORE01 -Credential $cred
whoami                                                                                 # lab\administrator
(Get-ADDomain).DNSRoot                                                                 # lab.local
```

Expect `lab\administrator` and `lab.local`. `DNSRoot` is one word; typed as `DNS.Root` it prints nothing, as in the capture below.

![Signed in to WS-CORE01 as lab\administrator](images/AD-08-domain-login.png)

**12.8 Health check** (inside WS-CORE01)

```powershell
Get-Service NTDS, DNS, Kdc, Netlogon, ADWS | Select-Object Name, Status
Get-ADDomainController | Select-Object Name, IPv4Address, IsGlobalCatalog
Resolve-DnsName _ldap._tcp.dc._msdcs.lab.local -Type SRV
```

All five services *Running*, the DC at 10.20.20.20 as a Global Catalog, and an SRV record pointing to `ws-core01.lab.local`. The SRV lookup is the exact question a machine asks when it joins the domain, so if it answers here, joins will work.

![Health check: five services Running, WS-CORE01 is a Global Catalog, SRV record resolves](images/AD-09-healthcheck.png)

**Local account or domain account.** The part before the backslash says where the account lives.

| Account | Stored on | Works on |
|---|---|---|
| `LAB\Administrator` | The domain (WS-CORE01) | Every machine joined to `lab.local` |
| `WS-CORE02\Administrator` | WS-CORE02 only | WS-CORE02 only |
| `HYPERVLAB\Administrator` | HYPERVLAB only | HYPERVLAB only, even when the DC is off |

**Checkpoint.** `Remove-VMCheckpoint -VMName WS-CORE01 -Name "pre-AD"` once the health check passes (never leave old checkpoints on a DC), shut everything down, VirtualBox snapshot `ad-dc-built`.

## Part 13 — Manage the domain from HYPERVLAB

WS-CORE01 has no desktop, so the graphical admin tools (RSAT) run on HYPERVLAB and connect to the DC over `lab-internal`. These are only the remote controls; the domain itself stays on WS-CORE01.

**13.1 Install the admin tools** (HYPERVLAB)

```powershell
Install-WindowsFeature RSAT-AD-Tools, RSAT-DNS-Server, GPMC
Get-WindowsFeature RSAT-AD-Tools, RSAT-DNS-Server, GPMC
```

| Feature | Tool | Used for |
|---|---|---|
| `RSAT-AD-Tools` | Active Directory Users and Computers (`dsa.msc`) + the AD PowerShell module | Users, groups, computers, OUs |
| `RSAT-DNS-Server` | DNS Manager | The `lab.local` DNS records |
| `GPMC` | Group Policy Management (`gpmc.msc`) | Policies |

![RSAT install on HYPERVLAB: Success True, three features installed](images/AD-10-rsat.png)

Do not tick *Active Directory Domain Services* under Add Roles and Features on HYPERVLAB. That would make HYPERVLAB a DC; the DC is WS-CORE01.

**13.2 Tell HYPERVLAB where ****`lab.local`**** is** (HYPERVLAB)

HYPERVLAB's normal DNS is on the VirtualBox NAT side and knows nothing about the domain. This rule sends only `lab.local` lookups to the DC; everything else resolves as before.

```powershell
Add-DnsClientNrptRule -Namespace ".lab.local", "lab.local" -NameServers 10.20.20.20   # only names ending in lab.local go to the DC
Clear-DnsClientCache
Resolve-DnsName ws-core01.lab.local                                                   # 10.20.20.20 = HYPERVLAB can now find the DC by name
```

![DNS rule added; ws-core01.lab.local resolves to 10.20.20.20](images/AD-11-dns-rule.png)

**13.3 Open Active Directory Users and Computers** (HYPERVLAB)

```powershell
runas /netonly /user:LAB\Administrator "mmc dsa.msc /server=ws-core01.lab.local"
```

Type the domain admin password; nothing shows while you type. `runas /netonly` is needed while HYPERVLAB is not a domain member: it opens the tool with the domain account for anything done over the network. After Part 17 the tool opens normally from the Tools menu.

![Users and Computers connected to lab.local, WS-CORE01 listed under Domain Controllers](images/AD-12a-aduc-dc.png)

Everything in the tree except the domain name, WS-CORE01 and the Administrator account is created by Windows for every new domain.

| Default folder | Holds |
|---|---|
| Builtin | Standard groups such as Administrators and Backup Operators |
| Computers | Machines that join without a chosen OU |
| Domain Controllers | The DCs |
| Users | Built-in accounts and groups: Administrator, Domain Admins, Domain Users |

![The default Users container of a new domain](images/AD-12b-aduc-users.png)

**If the window opens with “Naming information cannot be located … The logon attempt failed”**, the DC was found but the password was wrong. Close it and run the command again.

## Part 14 — OUs, users and a group

Each object type is created twice, once by clicking and once by command, so both ways are on record. Work on HYPERVLAB with WS-CORE01 running; wait about 3 minutes after the DC starts before opening the tools.

| Object | Name | Where | Made with |
|---|---|---|---|
| OU | `Staff` | `lab.local` | GUI |
| OU | `Servers` | `lab.local` | PowerShell |
| User | Alex Tran (`atran`) | `Staff` | GUI |
| User | Priya Shah (`pshah`) | `Staff` | PowerShell |
| Group | `Staff-Team` (Global, Security) | `Staff` | GUI |

The PowerShell steps need the domain credential in `$cred`. Set it once per window:

```powershell
# ONE LINE AT A TIME (Read-Host)
$pw = Read-Host "LAB admin password" -AsSecureString
$cred = New-Object System.Management.Automation.PSCredential("LAB\Administrator", $pw)
```

**14.1 Create the ****`Staff`**** OU** (Users and Computers)

An OU (Organisational Unit) is a folder inside the domain. Group Policy is applied to OUs, which is the real reason they exist. Right-click **lab.local** → **New** → **Organizational Unit**.

![Right-click lab.local, New, Organizational Unit](images/AD-14-new-menu.png)

Name it `Staff` and leave **Protect container from accidental deletion** ticked → OK.

![New Object dialog: OU named Staff, protection ticked](images/AD-14a-new-ou.png)

![Staff OU now in the tree, empty](images/AD-14b-ou-created.png)

To delete a protected OU: **View → Advanced Features**, then the OU's **Properties → Object** tab, untick the protection, then Delete. Without that, Delete answers *Access is denied*.

**14.2 Create the ****`Servers`**** OU** (HYPERVLAB, PowerShell)

```powershell
New-ADOrganizationalUnit -Name "Servers" -Path "DC=lab,DC=local" -Server ws-core01.lab.local -Credential $cred
Get-ADOrganizationalUnit -Filter * -Server ws-core01.lab.local -Credential $cred | Select-Object Name, DistinguishedName
```

`DC=lab,DC=local` is how Active Directory writes `lab.local`: each part of the name becomes a `DC=` piece. No output from the first line means success.

![Servers OU created by command; three OUs listed with their distinguished names](images/AD-15a-ou-powershell.png)

Press **F5** in Users and Computers and `Servers` appears beside `Staff`.

![Both OUs, Servers and Staff, in the tree](images/AD-15b-both-ous.png)

**14.3 Create a user** (Users and Computers)

Right-click **Staff** → **New** → **User**. First name `Alex`, last name `Tran`, logon name `atran`.

![New user dialog: Alex Tran, logon name atran](images/AD-16a-new-user.png)

Set a password (8+ characters, mixed case and a number). *User must change password at next logon* is unticked here only to make lab test sign-ins simpler; in a real job leave it ticked so the admin never knows the user's password.

![Password page with the four account options](images/AD-16b-user-password.png)

![Summary page before Finish: Alex Tran, atran@lab.local](images/AD-16c-user-finish.png)

![Alex Tran listed in the Staff OU](images/AD-16d-user-created.png)

**14.4 Create a second user** (HYPERVLAB, PowerShell)

```powershell
# ONE LINE AT A TIME (Read-Host)
$upw = Read-Host "Password for new user" -AsSecureString
New-ADUser -Name "Priya Shah" -GivenName Priya -Surname Shah -SamAccountName pshah -UserPrincipalName pshah@lab.local -Path "OU=Staff,DC=lab,DC=local" -AccountPassword $upw -Enabled $true -Server ws-core01.lab.local -Credential $cred
# -Name = display name   -SamAccountName = logon name   -UserPrincipalName = email-style logon   -Path = which OU   -Enabled $true = can sign in
# -Server / -Credential = which DC to talk to and as whom (needed because HYPERVLAB is not in the domain yet)
Get-ADUser -Filter * -SearchBase "OU=Staff,DC=lab,DC=local" -Server ws-core01.lab.local -Credential $cred | Select-Object Name, SamAccountName, Enabled
```

`-SamAccountName` is the logon name (`LAB\pshah`). `-Enabled $true` matters: without it PowerShell creates the account disabled. Type the first letters of a parameter and press **Tab** to complete it; `-UserPrincipleName` (wrong spelling) is rejected.

![Priya Shah created by command; both users listed and enabled](images/AD-17a-user-powershell.png)

![Both users in the Staff OU after refresh](images/AD-17b-two-users.png)

To remove a user: `Remove-ADUser -Identity pshah -Server ws-core01.lab.local -Credential $cred`.

**14.5 Create a group and add both users** (Users and Computers)

Permissions are given to groups, not to people one by one. Right-click **Staff** → **New** → **Group**, name `Staff-Team`, scope **Global**, type **Security**.

![New group dialog: Staff-Team, Global, Security](images/AD-18a-new-group.png)

Double-click the group → **Members** → **Add** → type `atran; pshah` → **Check Names** (both become underlined) → OK.

![Adding Alex Tran and Priya Shah on the Members tab](images/AD-18b-group-members.png)

Check it from PowerShell:

```powershell
Get-ADGroupMember "Staff-Team" -Server ws-core01.lab.local -Credential $cred | Select-Object Name, SamAccountName
```

![Get-ADGroupMember lists both users](images/AD-18c-group-check.png)

## Part 15 — Join WS-CORE02 to the domain

Joining makes WS-CORE02 a member server: domain accounts can sign in to it and domain policies apply to it. The DC keeps a record of it (a computer account) in the `Servers` OU.

**15.1 Memory sizing that fits a 6 GB server** (HYPERVLAB)

Two nested VMs must fit inside about 2 GB in total: Hyper-V keeps a reserve for HYPERVLAB itself, so with the DC at 2 GB `Start-VM WS-CORE02` failed with *Not enough memory in the system* even with 1.47 GB showing free. If you set 1280 MB and 768 MB in Part 12.1 this is already done — just check with `Get-VM | Select-Object Name, MemoryStartup` and start both. Otherwise:

```powershell
Stop-VM WS-CORE01
Set-VMMemory WS-CORE01 -StartupBytes 1280MB
Set-VMMemory WS-CORE02 -StartupBytes 768MB
Start-VM WS-CORE01
Start-VM WS-CORE02
Get-VM | Select-Object Name, State, MemoryAssigned
```

| VM | Memory | Note |
|---|---|---|
| WS-CORE01 (DC) | 1280 MB | Plenty for a Server Core DC with a handful of users |
| WS-CORE02 (member) | 768 MB | Server Core with no roles |
| UBU01 | Off | No room for a third VM |

![Both servers Running: WS-CORE01 at 1280 MB, WS-CORE02 at 768 MB, UBU01 Off](images/AD-19a-both-running.png)

Wait about 3 minutes for the DC's services before going on.

**15.2 Connect with the local account and check name and IP** (HYPERVLAB → inside WS-CORE02)

WS-CORE02 is not in the domain yet, so it only knows its own local `Administrator`. New variable names keep the domain login in `$cred` intact.

```powershell
# ONE LINE AT A TIME (Read-Host)
$pw2 = Read-Host "WS-CORE02 local password" -AsSecureString
$local2 = New-Object System.Management.Automation.PSCredential("Administrator", $pw2)   # WS-CORE02's own local Administrator
Enter-PSSession -VMName WS-CORE02 -Credential $local2                                  # prompt becomes [WS-CORE02]:
hostname
Get-NetIPAddress -AddressFamily IPv4 -InterfaceAlias Ethernet | Select-Object IPAddress, PrefixLength
```

![Inside WS-CORE02 before the join: hostname WS-CORE02, IP 10.20.20.30/24](images/AD-19b-core02-before-join.png)

**15.3 Point DNS at the DC** (inside WS-CORE02)

A machine finds its domain through DNS. Wrong DNS is the most common reason a join fails.

```powershell
Set-DnsClientServerAddress -InterfaceAlias Ethernet -ServerAddresses 10.20.20.20   # ask the DC for all name lookups
Resolve-DnsName lab.local                                                           # answers 10.20.20.20 = DNS works
ping 10.20.20.20                                                                    # replies = the DC is reachable
```

![WS-CORE02 resolves lab.local to 10.20.20.20 and pings the DC](images/AD-20-core02-dns.png)

**15.4 Join** (inside WS-CORE02)

The join is approved with a domain account, not WS-CORE02's local one: only the domain can create the computer account.

```powershell
# ONE LINE AT A TIME (Read-Host). You are inside WS-CORE02, so $cred from HYPERVLAB does not exist here: build the domain credential again.
$dpw = Read-Host "LAB admin password" -AsSecureString
$dcred = New-Object System.Management.Automation.PSCredential("LAB\Administrator", $dpw)
Add-Computer -DomainName lab.local -OUPath "OU=Servers,DC=lab,DC=local" -Credential $dcred -Restart   # join lab.local, land in the Servers OU, reboot
```

`-OUPath` puts the computer straight into `Servers` instead of the default Computers folder. The server restarts and the session drops back to HYPERVLAB.

![Add-Computer run inside WS-CORE02; the session returns to HYPERVLAB as it restarts](images/AD-21-join-command.png)

**15.5 Prove it: sign in with the domain account** (HYPERVLAB)

`LAB\Administrator` was never created on WS-CORE02, yet it works there. WS-CORE02 asks the DC to check the login.

```powershell
Enter-PSSession -VMName WS-CORE02 -Credential $cred
whoami
(Get-CimInstance Win32_ComputerSystem).Domain
```

![Signed in to WS-CORE02 as lab\administrator; the machine reports domain lab.local](images/AD-22-core02-domain-login.png)

**15.6 See the computer account** (HYPERVLAB)

```powershell
Get-ADComputer -Filter * -Server ws-core01.lab.local -Credential $cred | Select-Object Name, DistinguishedName
```

![Two computer accounts: WS-CORE01 under Domain Controllers, WS-CORE02 under Servers](images/AD-23a-computers-ps.png)

![WS-CORE02 in the Servers OU in Users and Computers](images/AD-23b-servers-ou.png)

**Who can sign in where.**

| Account | WS-CORE01 (DC) | WS-CORE02 (member) |
|---|---|---|
| `LAB\Administrator` | Yes, full control | Yes, full control |
| `LAB\atran`, `LAB\pshah` | Blocked: only admins sign in to a DC | Yes, as standard users |
| `WS-CORE02\Administrator` | Does not exist there | Yes, this machine only |

## Part 16 — Group Policy: a logon banner

A Group Policy Object (GPO) is a setting made once on the domain that every machine in the linked OU picks up by itself. This one, `Servers-Banner`, shows a warning on the sign-in screen of every server in the `Servers` OU.

It is built with commands on the DC because Group Policy Management will not connect from a machine outside the domain: at this stage it fails with *No mapping between account names and security IDs was done*, even with `runas /netonly`. Part 17 fixes that.

**16.1 Create the policy, add the settings, link it** (HYPERVLAB → inside WS-CORE01)

```powershell
Enter-PSSession -VMName WS-CORE01 -Credential $cred                        # the GP cmdlets run on the DC
New-GPO -Name "Servers-Banner"                                             # an empty policy object
$key = "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System"   # the registry key Windows reads for the sign-in message
Set-GPRegistryValue -Name "Servers-Banner" -Key $key -ValueName legalnoticecaption -Type String -Value "LAB.LOCAL"            # message title
Set-GPRegistryValue -Name "Servers-Banner" -Key $key -ValueName legalnoticetext -Type String -Value "Authorised use only"     # message body
New-GPLink -Name "Servers-Banner" -Target "OU=Servers,DC=lab,DC=local"    # attach it to the Servers OU: every computer in there gets it
```

| Command | GUI equivalent |
|---|---|
| `New-GPO` | Create a GPO (empty, applies to nobody yet) |
| `Set-GPRegistryValue` × 2 | *Interactive logon: Message title* and *Message text* under Security Options |
| `New-GPLink` | *Link an Existing GPO* on the `Servers` OU |

![New-GPO creates Servers-Banner; the first setting is saved](images/AD-24a-gpo-created.png)

![Second setting saved, then New-GPLink attaches the policy to the Servers OU (Enabled True)](images/AD-24b-gpo-linked.png)

**16.2 Apply it on WS-CORE02 and check** (HYPERVLAB → inside WS-CORE02)

Machines refresh policy by themselves about every 90 minutes and at restart. `gpupdate /force` fetches it now.

```powershell
exit                                                      # leave the WS-CORE01 session
Enter-PSSession -VMName WS-CORE02 -Credential $cred       # into the member server
gpupdate /force                                           # fetch and apply policy now instead of waiting
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" | Select-Object legalnoticecaption, legalnoticetext   # the two values the policy wrote
gpresult /r /scope computer                               # which policies this computer applied
```

The two values were never typed on WS-CORE02, so finding them there proves the policy delivered them.

![gpupdate succeeds; both banner values are present on WS-CORE02](images/AD-25a-policy-values.png)

![gpresult: Servers-Banner and Default Domain Policy under Applied Group Policy Objects](images/AD-25b-gpresult.png)

**16.3 See it on screen** (HYPERVLAB)

```powershell
vmconnect localhost WS-CORE02
```

**Action → Ctrl+Alt+Delete**. The message appears before the password prompt. If the console is already signed in, type `logoff` there first.

![The banner on WS-CORE02's sign-in screen: LAB.LOCAL, Authorised use only](images/AD-25c-logon-banner.png)

If a local admin changes a setting that a domain policy controls, the next refresh puts it back. The domain wins.

## Part 17 — Join HYPERVLAB for GUI management

HYPERVLAB joins `lab.local` so Group Policy Management and Server Manager work normally, with no `runas`. It goes into its own `Management` OU with policy inheritance blocked, so domain policies do not land on the Hyper-V host.

| What changes | What does not |
|---|---|
| `LAB\Administrator` can sign in to HYPERVLAB and has admin rights there | HYPERVLAB still starts, stops and checkpoints every VM, the DC included |
| The AD tools open straight from the Tools menu | `HYPERVLAB\Administrator` still works, with or without the DC |
| A domain sign-in needs the DC running (or a cached earlier sign-in) | No domain GPO applies while inheritance is blocked |

The host now depends on a DC that it hosts. A real site avoids that loop with a DC on separate hardware; here the local Administrator is the way in if the DC is down.

**17.1 Snapshot first.** Stop both VMs, `Stop-Computer`, then VirtualBox snapshot `before-hypervlab-join-domain`. Restoring it undoes this whole part.

**17.2 Create the ****`Management`**** OU and block inheritance** (HYPERVLAB → inside WS-CORE01)

```powershell
Enter-PSSession -VMName WS-CORE01 -Credential $cred
New-ADOrganizationalUnit -Name "Management" -Path "DC=lab,DC=local"          # the OU HYPERVLAB will live in
Set-GPInheritance -Target "OU=Management,DC=lab,DC=local" -IsBlocked Yes       # domain policies stop at this OU's door
exit
```

`New-ADOrganizationalUnit` takes `-Name`; `-Target` belongs to `Set-GPInheritance`. Want `GpoInheritanceBlocked : Yes`.

![Management OU created and GpoInheritanceBlocked set to Yes](images/AD-27-block-inheritance.png)

**17.3 Join** (HYPERVLAB, plain prompt)

```powershell
Add-Computer -DomainName lab.local -OUPath "OU=Management,DC=lab,DC=local" -Credential $cred -Restart
```

HYPERVLAB restarts inside the same VirtualBox window, so the P-core affinity is kept. The sign-in screen now offers **Administrator** (the local account) and **Other user**. Sign in as the local account first and check:

```powershell
Get-VM | Select-Object Name, State
(Get-CimInstance Win32_ComputerSystem).Domain
```

![After the join: both servers Running and HYPERVLAB reports domain lab.local](images/AD-28-hypervlab-joined.png)

**17.4 Sign in with the domain account.** Sign out → **Other user** → `LAB\Administrator` + the domain admin password. The first sign-in builds a new, empty profile; files from the local account are still under `C:\Users\Administrator\`.

**17.5 Group Policy Management** (HYPERVLAB as `LAB\Administrator`)

```powershell
gpmc.msc
```

Expand **Forest: lab.local → Domains → lab.local**. `Servers-Banner` sits under **Servers**, and **Management** carries the blue mark of Block Inheritance. The banner values are on the **Settings** tab under Administrative Templates, as *Extra Registry Settings*.

![Group Policy Management: Servers-Banner linked under Servers, blue Block Inheritance mark on Management](images/AD-29-gpmc-working.png)

**17.6 Add both servers to Server Manager** (HYPERVLAB as `LAB\Administrator`)

**Manage → Add Servers → Active Directory** tab → **Find Now** → select WS-CORE01 and WS-CORE02 → arrow → OK.

![Add Servers: Find Now lists WS-CORE01, WS-CORE02 and HyperVLab from Active Directory](images/AD-30a-add-servers.png)

**All Servers** then shows three machines *Online*, and the left menu gains **AD DS** and **DNS** from WS-CORE01.

![Server Manager All Servers: HYPERVLAB, WS-CORE01 and WS-CORE02 Online, AD DS and DNS in the menu](images/AD-30b-server-manager.png)

*Performance counters not started* only means the optional graphs are off. The `Security-SPP` error 8198 on WS-CORE02 is Windows activation failing, expected with no route to Microsoft.

**Checkpoint.** Stop WS-CORE02, then WS-CORE01, `Stop-Computer`, VirtualBox snapshot `ad-domain-complete`.

## Daily operations

**Starting a lab session**

1. HOST: confirm `bcdedit | findstr hypervisor` says `Off` (if you turned the host back to normal, set `off` and reboot first).
2. Double-click `start-lab.cmd`. Never the Start button in VirtualBox.
3. Sign in to HYPERVLAB. After Part 17 choose Other user and sign in as LAB\Administrator to use the AD tools; HYPERVLAB\Administrator always works, even with the DC off. Open Hyper-V Manager or Terminal (Admin).
4. Start **one** nested VM: `Start-VM UBU01` or `Start-VM WS-CORE01`. Two only for network tests. With the domain built: check WS-CORE01 is running (it may start with the host), wait about 3 minutes for its services, then start WS-CORE02 if needed and keep UBU01 off.

**Ending a lab session**

1. `Stop-VM <name>` for each running nested VM (or `Get-VM | Where State -eq Running | Stop-VM`).
2. HYPERVLAB: Start → Power → Shut down. Wait for the VirtualBox window to close. With the domain, stop member servers before the DC (WS-CORE02, then WS-CORE01).
3. Optional: VirtualBox → Snapshots → Take, named for what you just finished.
4. Optional, for non-lab days: HOST `bcdedit /set hypervisorlaunchtype auto`, Memory Integrity on, reboot.

**Rules that keep it working**

| Rule | Why |
|---|---|
| Clean shutdown before every snapshot | Snapshots of a running server trigger recovery screens, the Shutdown Event Tracker and a corrupt Server Manager settings file next boot |
| Never move VM folders by hand | Use `Move…` in VirtualBox for HYPERVLAB, Export/Import in Hyper-V for nested VMs |
| `Get-VMHardDiskDrive` before any disk change | Catches wrong paths and parent/child mistakes before they cost a rebuild |
| Echo a variable (`$vmcx`) before using it | A variable keeps its old value; the WS-CORE01 export path was reused for UBU01 by mistake |
| Check sizes after any download | A 6 GB ISO is right; a wrong file or a cut-off download boots to *Boot Summary* |
| Stay inside the memory budget | 6 GB HYPERVLAB: UBU01 alone, or WS-CORE01 (1280 MB) + WS-CORE02 (768 MB). Anything more fails to start |
| No checkpoints left on the domain controller | Reverting a DC to an old checkpoint rolls the directory back; take one before a risky change and remove it once the change is confirmed |
| Check the prompt before every AD command | [WS-CORE01] means inside the DC, a plain prompt means HYPERVLAB; Install-ADDSForest on the wrong one would make HYPERVLAB the DC |
| Nothing else open inside HYPERVLAB while a nested VM runs | Edge alone took 1.1 GB and left a 1 GB VM unable to start; Hyper-V needs about 2 GB free to start a 1 GB VM. Finish downloads in Edge before starting any nested VM, then close Edge |

**Lab address book**

| Machine | Address | Role | Login |
|---|---|---|---|
| HYPERVLAB (lab-internal leg) | 10.20.20.1 | Hyper-V host, management machine, member of `lab.local` (Management OU) | `LAB\Administrator`, or `HYPERVLAB\Administrator` (local) |
| UBU01 | 10.20.20.10 | Ubuntu Server, not domain-joined | wick |
| WS-CORE01 | 10.20.20.20 | Domain controller + DNS for `lab.local` | `LAB\Administrator` |
| WS-CORE02 | 10.20.20.30 | Member server (Servers OU) | `LAB\Administrator`, or `WS-CORE02\Administrator` (local) |
| Spare for clones | 10.20.20.40 onwards | — | — |

## Troubleshooting and lessons learned

Every one of these happened during the build. Newest-build-order first.

| Symptom | Cause | Fix |
|---|---|---|
| No “Hyper-V” in Windows Features | Windows 11 Home | Nested method in this guide, or upgrade to Pro/Education |
| *A hypervisor has been detected* + nested VM won't start | VirtualBox's default paravirtualisation presents a fake Hyper-V to the guest | VirtualBox → Acceleration → Paravirtualization Interface = **None** |
| Server boots to *Choose an option* recovery screen | Hard power-off or crash the boot before | Click Continue |
| VirtualBox **Guru Meditation**, `VERR_VMM_RING0_ASSERTION` on VCPU3 | Nested virt crashes when a vCPU runs on an E-core (hybrid Intel CPU) | Launch with `start /affinity FF`; keep RAM at 6 GB; fallback is 2 vCPUs, then VMware Workstation Pro |
| `I/O cache … VERR_DISK_FULL`, VM paused | Host drive holding the VDI is full | Free space (move ISOs), resume; then move the VM to a bigger drive with VirtualBox → Move |
| VM viewer tiny, no View menu, won't resize | Saved zoom level in `%APPDATA%\Microsoft\Windows\Hyper-V\Client\1.0\vmconnect*.config` | Delete that file (search with `Get-ChildItem $env:USERPROFILE -Recurse -Force -Filter vmconnect*`) and reconnect |
| Nested VM shows *Start PXE over IPv4* / *Boot Summary* | Missed the *Press any key* prompt, or the ISO is wrong/incomplete | Reset and tap spacebar; check ISO size; avoid the `LOF_PACKAGES` ISO |
| Ubuntu *Out of memory: Killed process* flood | Dynamic Memory shrank it to 512 MB | Static memory: `Set-VM UBU01 -StaticMemory -MemoryStartupBytes 1536MB` |
| Ubuntu *KERNEL PANIC … deadlocked on memory* | Not enough free RAM on HYPERVLAB for the VM's allocation | Stop the other nested VM; 1.5 GB for Ubuntu |
| *Not enough memory in the system to start the virtual machine* | Same as above, Hyper-V's version | Same |
| Can't ping HYPERVLAB from a VM | Windows Server firewall blocks ICMP by default | `New-NetFirewallRule -DisplayName "Allow ICMPv4 In" -Protocol ICMPv4 -Direction Inbound -Action Allow` |
| Ping to 10.20.20.10 succeeds from the wrong switch | VirtualBox NAT sent it to the internet; an ISP device answered | Check RTT (<1 ms = same switch), confirm target is actually up |
| `New-Partition … DriveLetter D` fails *access path already in use* | D: is the VM's DVD drive | Use E:, or remove the DVD drive |
| `Merge-VHD`: *chain of virtual hard disks is corrupted … identifiers* | VM ran on the parent after the child was created (child stayed ~4 MB) | Discard child, or `Set-VHD -IgnoreIdMismatch` then merge; always verify `Get-VMHardDiskDrive` shows the child before `Start-VM` |
| `Export-VM`: *not enough disk space* | 60 GB C: full | Add the L: disk (Part 4), export there |
| `Import-VM` *ObjectNotFound* | `$vmcx` still held the previous VM's path | Re-run the `Get-ChildItem … *.vmcx` line and echo `$vmcx` |
| `Set-VMHardDiskDrive` *path is invalid* | Typo (`.chdx`) | List the folder first, copy the exact name |
| Script: *Missing an argument for parameter SwitchName* | `$Switch` is a reserved PowerShell variable / missing value | Name it `$SwitchName` |
| `Get-Credential` returns nothing | Dialog hidden behind Terminal | Build it with `Read-Host -AsSecureString` + `New-Object PSCredential` |
| PowerShell prompt stuck at `>>` | Unclosed bracket or quote | Ctrl+C |
| Nested VM IP is `169.254.x.x` | APIPA — no DHCP on lab-internal, no static set | `New-NetIPAddress` (from the console or via PowerShell Direct) |
| `Get-VMNetworkAdapter … IPAddresses` blank | VM off, or guest has no KVP integration agent (Ubuntu without hyperv-daemons) | Check inside the VM instead |
| Server Manager *error in a user settings file* | Settings file corrupted by an unclean stop | OK to restore defaults |
| Enhanced Session never offered on WS-CORE02 | Unresolved; Core guest | Worked on a fresh clone; expect it on Desktop Experience guests |
| Not enough memory to start a 1 GB VM, nothing else running, HYPERVLAB has 6 GB | Programs inside HYPERVLAB are using the memory (Edge, Defender's first scan) | Get-Process \| Sort-Object WorkingSet64 -Descending \| Select-Object -First 8 to find them; close Edge; want 2 GB+ free before Start-VM |
| Windows Server installation has failed, twice, at the same point | VM has 1 GB; setup needs more than the installed OS does | Stop-VM -TurnOff, Set-VMMemory -StartupBytes 2048MB, delete the leftover partitions at the disk screen, install, then back to 1024MB |
| Test-Connection: a non-recoverable error occurred during a database lookup | PowerShell 5.1 does a reverse DNS lookup before pinging; the lab has no DNS until Part 12 | Use ping 10.20.20.x or Test-NetConnection 10.20.20.x instead |
| New-VHD / Copy-Item: '…\UBU01.vhdx' is not an existing virtual hard disk file | The wizard stores disks under L:\VMs\<VM>\Virtual Hard Disks\, not L:\VMs\<VM>\ | (Get-VMHardDiskDrive <VM>).Path gives the real path; never type a disk path from memory |
| Split-Path: cannot bind argument to parameter 'Path' because it is null | $os is empty: with a checkpoint present the VM's disk path is an .avhdx, so a filter on WS-CORE01.vhdx matches nothing | Pick the disk by slot: Get-VMHardDiskDrive <VM> -ControllerType SCSI -ControllerNumber 0 -ControllerLocation 0 |
| Import-VM: the file 'L:\VMs\WS-CORE01\data01.vhdx' already exists | Remove-VM deletes only the VM definition; the original .vhdx files are still in the folder and Import-VM -Copy will not overwrite them | Keep the export. Rename-Item L:\VMs\WS-CORE01 L:\VMs\WS-CORE01-old, import again, boot it, then delete the -old folder |
| Copy-Item (9.3 template): There is not enough space on the disk | L: still holds the 9.2 leftovers — L:\Exports and L:\VMs\WS-CORE01-old are two extra copies of the same disks | Get-PSDrive L to see free space; delete both leftover folders (the imported VM has already booted), then run the copy again |
| Set-VM -AutomaticStartAction: failed to modify settings, 0x80041001, InvalidState | The VM is running; automatic start/stop actions are only editable while it is Off | Stop-VM <VM>, run the Set-VM line again, Start-VM <VM> |
| Pasted a block with Read-Host; password prompt filled itself with the next command lines | Read-Host reads the next pasted line as its input, so $pw holds a command, not the password | Ctrl+C (exit first if inside a session), then run the block one line at a time |
| Invoke-Command / Enter-PSSession -VMName: The credential is invalid (PSSessionStateBroken) | The clone is still at first-run setup — no Administrator password exists yet (or the password was mistyped) | Open the VM's console, set the password, then redo the $pw / $cred lines |

**Active Directory (Parts 12–17).**

| Symptom | Cause | Fix |
|---|---|---|
| `Checkpoint-VM`: *cannot bind parameter* | `-Name` given the checkpoint's name | `Checkpoint-VM -Name <VM> -SnapshotName "<checkpoint>"` |
| `Get-Credential`: *missing mandatory parameters: Credential* | The login popup cannot show or was closed | `Read-Host -AsSecureString` + `New-Object PSCredential`, as in Part 12.3 |
| `New-Object`: *Cannot find an overload for "PSCredential" and the argument count: "2"* | `$pw` is empty: the `Read-Host` line did not run, or lines were pasted together | Run the lines one at a time |
| Session stays open after `Install-ADDSForest` reports Success | The automatic reboot did not fire | `Restart-Computer -Force` inside the DC |
| *Naming information cannot be located … The logon attempt failed* | DC found, wrong password in `runas` | Close the window, run it again; test the password with `Invoke-Command -VMName WS-CORE01 -Credential $cred -ScriptBlock { whoami }` |
| Looking for Group Policy in Users and Computers | Different tool | `gpmc.msc` is Group Policy Management; `dsa.msc` is Users and Computers |
| Group Policy Management: *No mapping between account names and security IDs was done* | The machine running it is not a domain member | Build the GPO with commands on the DC (Part 16), or join HYPERVLAB (Part 17) |
| *A parameter cannot be found that matches parameter name* | Misspelt parameter (`-UserPrincipleName`) or one from another command (`-Target` on `New-ADOrganizationalUnit`) | Type the first letters and press Tab |
| `Start-VM WS-CORE02`: *Not enough memory*, even with 1.4 GB free | Hyper-V reserves memory for HYPERVLAB itself | DC 1280 MB + member 768 MB (Part 15.1) |
| Deleting an OU: *Access is denied* | *Protect from accidental deletion* is on | View → Advanced Features → Properties → Object tab → untick |
| Domain sign-in or join fails right after the DC starts | AD services take about 3 minutes to come up | Wait, then retry; check with `Resolve-DnsName lab.local` |
| `Start-VM`: *already in the specified state* after a host boot | The VM's automatic start action started it | Nothing to fix; `Get-VM` with `AutomaticStartAction` shows the setting |

## Appendix A — Screenshots

**How to capture.** Take every screenshot from the **laptop**, so it lands in the laptop's clipboard and can be pasted straight into this doc: press **Win + Shift + S**, drag over the VirtualBox window (or the part that matters), then click just below the step in the doc and press Ctrl + V. Capture the whole window including its title bar — the title shows which machine you are on. Blur or crop passwords and product keys.

The 17 key screenshots below are all placed, along with the extra ones taken around them (82 images for Parts 1–11, 43 for Parts 12–17). Local copies are named by step, `P1-01-memory-integrity.png` … `P11-04-psdirect.png` and `AD-01` … `AD-30b`, so one can be retaken and swapped without touching the rest.

| # | Where in doc | What it must show | Status |
|---|---|---|---|
| 1 | Prerequisites | Education Hub software list with *Windows Server 2025 Standard* visible | Placed |
| 2 | Part 1.1 | Core isolation page with Memory integrity **Off** | Placed |
| 3 | Part 1.2 | Windows Features list with Windows Hypervisor Platform and Virtual Machine Platform **unticked** | *Placed* |
| 4 | Part 2.1 | VirtualBox New VM dialog: name, OS edition dropdown, unattended **unticked** | Placed (wizard, hardware and disk pages) |
| 5 | Part 2.1 | VirtualBox Settings → System → Acceleration showing Paravirtualization **None**, and Processor tab with Nested VT-x ticked (two shots) | Placed (all six Settings pages) |
| 6 | Part 3.1 | Windows Setup *Select Image* screen with Standard (Desktop Experience) highlighted | Placed |
| 7 | Part 3.4 | HYPERVLAB PowerShell: `Get-Service vmms, vmcompute` both Running and `Get-VM Test` Running | Placed |
| 8 | Part 4.1 | VirtualBox Hard Disk Selector with the 80 GB VDI attached | Placed (Not Attached and attached states) |
| 9 | Part 5.1 | Virtual Switch Manager with all three switches listed | Placed (one per switch) |
| 10 | Part 5.4 | Get-VMSwitch output with the three switches | Placed |
| 11 | Part 6.2 | Hyper-V New VM wizard *Specify Name and Location* with `L:\VMs` | Placed (every wizard page; memory page shows 1024 MB, text says 2048 MB) |
| 12 | Part 6.2 | WS-CORE01 at the sconfig menu | Placed (before and after the rename) |
| 13 | Part 6.3 | Ubuntu installer network page with eth0 = 10.20.20.10/24 static | *Placed (every installer page*) |
| 14 | Part 6.3 | UBU01 `ip a` showing eth0 10.20.20.10/24 | Placed (with the Test B ping, under Part 7) |
| 15 | Part 7 Test B | WS-CORE01 `Test-Connection` to .10 and .1 with 0–1 ms, 1.1.1.1 failing | Placed (Tests A, B and C, plus Part 8 steps) |
| 16 | Part 9.4 | Hyper-V Manager VM list showing UBU01, WS-CORE01, WS-CORE02; folder listing with clone size vs template size | Placed (WS-CORE02 IP and ICMP rule; VM list and clone sizes under Part 11.1) |
| 17 | Part 11.4 | `Invoke-Command -VMName` output showing hostname and the 169.254 address | Placed |

**Tip for the next rebuild.** Keep a `Screenshots` folder on the laptop and name files `01-education-hub.png`, `02-memory-integrity.png` … in this order. Capturing as you go takes seconds; reconstructing later takes a rebuild.

## Appendix B — Quick command reference

All on HYPERVLAB unless marked.

| Task | Command |
|---|---|
| HOST: hypervisor off / on | `bcdedit /set hypervisorlaunchtype off` · `… auto` (reboot) |
| HOST: launch the server pinned | `start /affinity FF "" "C:\Program Files\Oracle\VirtualBox\VirtualBoxVM.exe" --startvm "WinServer2025"` |
| HOST: VirtualBox settings by CLI | `VBoxManage modifyvm "WinServer2025" --memory 6144 --cpus 4 --nested-hw-virt on --paravirtprovider none` |
| Is Hyper-V running? | `Get-Service vmms, vmcompute` |
| List VMs / switches / disks | `Get-VM` · `Get-VMSwitch` · `Get-VMHardDiskDrive <vm>` |
| Start / stop / hard off | `Start-VM <vm>` · `Stop-VM <vm>` · `Stop-VM <vm> -TurnOff` |
| Move a VM to another switch | `Connect-VMNetworkAdapter -VMName <vm> -SwitchName <switch>` |
| Memory (static) | `Set-VM <vm> -StaticMemory -MemoryStartupBytes 1536MB` |
| Memory (dynamic) | `Set-VMMemory <vm> -DynamicMemoryEnabled $true -MinimumBytes 768MB -StartupBytes 1024MB -MaximumBytes 1536MB -Buffer 20` |
| CPUs | `Set-VM <vm> -ProcessorCount 2` |
| Secure Boot for Linux | `Set-VMFirmware <vm> -SecureBootTemplate MicrosoftUEFICertificateAuthority` |
| New dynamic VHDX | `New-VHD -Path <file> -SizeBytes 10GB -Dynamic` |
| Attach / detach a disk | `Add-VMHardDiskDrive -VMName <vm> -Path <file>` · `Remove-VMHardDiskDrive -VMName <vm> -ControllerType SCSI -ControllerNumber 0 -ControllerLocation <n>` |
| Grow a disk | `Resize-VHD -Path <file> -SizeBytes 20GB` then inside: `Resize-Partition` |
| Differencing disk / merge | `New-VHD -Path <child> -ParentPath <parent> -Differencing` · `Merge-VHD -Path <child>` |
| Compact / convert | `Optimize-VHD -Path <file> -Mode Full` · `Convert-VHD -Path <src> -DestinationPath <dst> -VHDType Fixed` |
| Checkpoints | `Checkpoint-VM -Name <vm> -SnapshotName <n>` · `Restore-VMCheckpoint` · `Remove-VMCheckpoint` · `Set-VM <vm> -CheckpointType Production` |
| Export / import | `Export-VM -Name <vm> -Path L:\Exports` · `Import-VM -Path <vmcx> -Copy -GenerateNewId -VirtualMachinePath <p> -VhdDestinationPath <p>` |
| Sysprep (inside a Windows VM) | `C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown` |
| PowerShell Direct | `Invoke-Command -VMName <vm> -Credential $cred -ScriptBlock { … }` · `Enter-PSSession -VMName <vm> -Credential $cred` |
| Credential without dialog | `$pw = Read-Host -AsSecureString; $cred = New-Object System.Management.Automation.PSCredential("Administrator",$pw)` |
| Allow ping in Windows | `New-NetFirewallRule -DisplayName "Allow ICMPv4 In" -Protocol ICMPv4 -Direction Inbound -Action Allow` |
| Static IP (Windows) | `New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 10.20.20.x -PrefixLength 24` |
| Static IP (Ubuntu) | edit `/etc/netplan/50-cloud-init.yaml`, `sudo netplan apply` |
| Server Core menu | `sconfig` · `15` = PowerShell · `2` = rename · `7` = Remote Desktop · `8` = network · `14` = shut down |
| Find a cmdlet | `Get-Command -Module Hyper-V -Noun VM*` · `Get-Help <cmdlet> -Examples` |

**Active Directory.** Signed in to HYPERVLAB as `LAB\Administrator` these run as shown. From a non-domain session add `-Server ws-core01.lab.local -Credential $cred`.

| Task | Command |
|---|---|
| Domain summary | `Get-ADDomain` |
| DC health | `Get-Service NTDS, DNS, Kdc, Netlogon, ADWS` |
| List OUs | `Get-ADOrganizationalUnit -Filter *` |
| New OU | `New-ADOrganizationalUnit -Name "X" -Path "DC=lab,DC=local"` |
| New user | `New-ADUser -Name "First Last" -SamAccountName flast -Path "OU=Staff,DC=lab,DC=local" -AccountPassword $upw -Enabled $true` |
| Users in an OU | `Get-ADUser -Filter * -SearchBase "OU=Staff,DC=lab,DC=local"` |
| Reset a password | `Set-ADAccountPassword flast -Reset -NewPassword $upw` |
| Remove a user | `Remove-ADUser -Identity flast` |
| Group members | `Get-ADGroupMember "Staff-Team"` |
| Add to a group | `Add-ADGroupMember "Staff-Team" -Members flast` |
| List computers | `Get-ADComputer -Filter *` |
| Join a machine | `Add-Computer -DomainName lab.local -OUPath "OU=Servers,DC=lab,DC=local" -Credential $dcred -Restart` |
| New GPO + link | `New-GPO -Name "X"`, then `New-GPLink -Name "X" -Target "OU=Servers,DC=lab,DC=local"` |
| Refresh policy now | `gpupdate /force` |
| What policy applied | `gpresult /r /scope computer` |
| Open the GUI tools | `dsa.msc` (Users and Computers), `gpmc.msc` (Group Policy), `dnsmgmt.msc` (DNS) |

## Next steps — Phases 8–10

Parts 1–11 cover what a support, systems or junior infrastructure role expects of Hyper-V day to day, and Parts 12–17 add the Active Directory basics the same roles expect. The remaining three phases are the differentiators and all need a second Hyper-V host, so they start together. The domain that Phase 9 needs now exists.

| Phase | Topics | Needs |
|---|---|---|
| 8 — Backup and recovery | Export as backup and restore; Windows Server Backup of a VM; **Hyper-V Replica** between two hosts | Second nested host `WS-HV02` |
| 9 — Enterprise features | Two nested Hyper-V hosts; Failover Clustering basics; Live Migration; Windows Admin Center | Same, plus shared storage (iSCSI target on HYPERVLAB) and a domain |
| 10 — Cloud link | Compare Hyper-V and Azure VMs; upload a VHDX to Azure and boot it; Azure Arc overview | Azure for Students subscription (free, no card) |

**Active Directory follow-ons** (no extra hardware):

- More Group Policy through the GUI editor: a password policy, a mapped drive for `Staff`
- A file share on WS-CORE02 with permissions given to `Staff-Team`
- Sign in to WS-CORE02 as Alex Tran to see a standard user's limits
- A second domain controller, built through Server Manager's wizard from HYPERVLAB
- Join UBU01 to the domain

Known constraints going in: HYPERVLAB has 6 GB, so a second nested host (2 GB) plus a tiny VM inside it (1 GB) is the whole budget — UBU01 stays off. Clustering also needs the DC running (1280 MB), which this budget does not cover, so expect Replica hands-on and full clustering mostly as theory unless more memory is found. VirtualBox's nested virtualisation on this CPU is stable only with the affinity launcher; if Phase 9 pushes it over, the fallback is VMware Workstation Pro (free for personal use), converting the existing VDI rather than rebuilding.
