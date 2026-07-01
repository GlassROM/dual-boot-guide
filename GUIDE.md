Conservative Windows + Linux Dual-Boot Guide

Separate disks, separate ESPs, fallback Linux boot path, no Windows chainloading

This material is provided for informational and educational purposes only. It is not legal, security, compliance, enterprise deployment, or incident-response advice.

Product names, project names, company names, logos, and trademarks mentioned in this article belong to their respective owners.

This article is independent. It is not affiliated with, sponsored by, approved by, or endorsed by any operating-system vendor, Linux distribution, bootloader project, firmware vendor, hardware vendor, standards body, or software publisher mentioned.

For production, regulated, enterprise, or attested environments, follow the applicable vendor documentation and internal change-control process.

⸻

1. Scope

Modern dual boot is a boot-integrity design problem on systems that use Secure Boot, TPM, BitLocker, device encryption, Credential Guard, VBS, anti-cheat, firmware attestation, or enterprise security baselines.

The conservative layout is:

Disk 0: Windows disk
  EFI System Partition
  Microsoft Reserved Partition
  Windows partition
  Windows Recovery partition
Disk 1: Linux disk
  EFI System Partition
  Linux root partition
  optional /home or swap

Each operating system owns its own EFI System Partition.

Windows starts through Windows Boot Manager. Linux starts through its own Linux bootloader. Firmware chooses which operating system to start.

Use:

Firmware → Windows Boot Manager → Windows
Firmware → Linux bootloader → Linux

Avoid this as the normal Windows path:

Firmware → Linux bootloader → Windows Boot Manager → Windows

UEFI firmware has a boot manager that uses boot variables such as BootOrder and Boot#### to load EFI applications from bootable devices. That makes firmware-level OS selection available without requiring one operating system’s bootloader to start the other operating system. [1]

⸻

2. Core design

Use separate physical disks whenever possible.

Disk 0: Windows disk
  EFI System Partition
  Microsoft Reserved Partition
  Windows partition
  Windows Recovery partition
Disk 1: Linux disk
  EFI System Partition
  Linux root partition
  optional /home or swap

Each operating system should own its own EFI System Partition.

Do not share the Windows ESP with Linux unless both boot paths can be repaired manually.

Do not make GRUB, systemd-boot, rEFInd, shim, or another Linux-side bootloader responsible for starting Windows.

Windows should remain the default boot target. Linux should be selected when needed through the firmware one-time boot menu, firmware disk selector, or Windows Recovery’s device boot path.

Microsoft’s UEFI/GPT partition guidance describes the EFI System Partition as the system partition used for boot, formatted FAT32, managed by the operating system, and not intended to contain other files. [3]

⸻

3. Why this layout is used

UEFI allows multiple EFI applications, multiple boot entries, and multiple EFI System Partitions. The UEFI specification also states that it does not restrict the number or location of system partitions and that coordination between operating systems sharing one ESP is outside the scope of the specification. [2]

That flexibility does not make every boot arrangement equally robust.

Real systems vary. Firmware may lose, rewrite, hide, or reorder boot entries. Board menus may expose disk labels instead of loader names. Some systems expose only one boot path clearly. Secure Boot and TPM-dependent Windows features rely on early boot state remaining within expected boundaries.

Microsoft documents that Windows uses the TPM for system integrity measurements and that BitLocker can depend on expected startup measurements before releasing the encryption key normally. [5] Microsoft also lists boot-manager changes, firmware changes, BCD changes, Secure Boot changes, and PCR changes among BitLocker recovery triggers. [6]

Operational rule:

Windows boots through the Windows path.
Linux boots through the Linux path.
Neither operating system depends on the other operating system’s bootloader.
EFI variable churn is minimized.
Each disk remains independently recoverable.

The design reduces cross-OS boot coupling.

⸻

4. UEFI boot entries and fallback paths

UEFI systems commonly boot through firmware boot entries stored in NVRAM. Those entries point to EFI executables such as:

\EFI\Microsoft\Boot\bootmgfw.efi
\EFI\systemd\systemd-bootx64.efi
\EFI\GRUB\grubx64.efi

On x86_64 systems, the fallback filename is:

\EFI\BOOT\BOOTX64.EFI

UEFI defines the \EFI\BOOT\BOOT{machine type short name}.EFI pattern. For removable media, UEFI specifies one UEFI-compliant system partition and one executable EFI image per supported processor architecture in the BOOT directory. [2]

Many systems also honor this fallback path when a physical internal disk is selected from a firmware boot menu. Fixed-disk behavior remains firmware-dependent.

Use the Linux disk fallback path as a compatibility measure:

Linux disk ESP:
  \EFI\BOOT\BOOTX64.EFI
  plus the files required by the Linux bootloader

A shared ESP creates a single fallback-loader filename for each architecture. Only one file can occupy this path on a single ESP:

\EFI\BOOT\BOOTX64.EFI

That filename collision is one reason to keep Windows and Linux on separate ESPs.

⸻

5. Board-specific boot behavior

The standard x86_64 fallback filename is:

\EFI\BOOT\BOOTX64.EFI

The normal Windows Boot Manager path is:

\EFI\Microsoft\Boot\bootmgfw.efi

Firmware menus may label entries inconsistently.

Possible labels include:

Windows Boot Manager
UEFI OS
Other OS
Linux Boot Manager
NVMe drive
SATA drive
the physical drive model

Minimum validation:

1. Boot Windows normally.
2. Boot Linux from the firmware one-time boot menu.
3. Boot Linux from Windows Recovery → Use a device, if exposed by the firmware.
4. Power off fully.
5. Cold boot.
6. Confirm Windows still boots first by default.
7. Confirm Linux is still reachable.

Both disks should remain independently bootable. The board determines how clearly those paths are exposed.

⸻

6. Windows boot layout

Windows should have its own ESP.

A normal Windows UEFI installation should manage its own Windows Boot Manager entry and boot files.

The normal Windows boot path is:

\EFI\Microsoft\Boot\bootmgfw.efi

Do not make GRUB, systemd-boot, rEFInd, shim, or Linux efibootmgr responsible for starting Windows.

Do not use this as the normal Windows boot-management path:

bcdboot C:\Windows /s S: /f UEFI

The /s form is useful for recovery, removable media, secondary-disk deployment, and controlled boot-file servicing. Microsoft documents that /s specifies the system partition and should not be used in typical deployment scenarios. Microsoft also states that when /s is used on UEFI systems, BCDBoot does not create the normal NVRAM firmware entry and relies on default firmware behavior instead. [4]

Operational rule:

Windows manages Windows Boot Manager.
Linux manages the Linux bootloader.
The two boot chains stay separate.

⸻

7. Linux boot layout

Linux should have its own ESP on the Linux disk.

The Linux ESP should contain a bootable fallback loader at:

\EFI\BOOT\BOOTX64.EFI

That fallback loader may be GRUB, systemd-boot, shim, PreLoader, or another distribution-supported first-stage loader, depending on Secure Boot state.

Do not install Linux boot files to the Windows ESP.

Do not overwrite the Windows fallback loader.

⸻

8. GRUB fallback install

For a Linux disk that should boot independently, install GRUB only to the Linux ESP.

Generic fallback install:

grub-install \
  --target=x86_64-efi \
  --efi-directory=/efi \
  --boot-directory=/boot \
  --removable

Adjust /efi and /boot for the distribution’s mount layout.

Common ESP mountpoints include:

/efi
/boot
/boot/efi

The --removable option tells grub-install to treat the installation device as removable. On x86_64 UEFI systems, this places the bootloader at the removable-media fallback path:

\EFI\BOOT\BOOTX64.EFI

Some GRUB builds also expose a separate EFI-only option:

--force-extra-removable

Use --force-extra-removable when the distribution supports it and the intent is to install an additional fallback copy while retaining the normal distribution bootloader location.

For a conservative Linux-disk ESP, the relevant GRUB options are:

--removable
--force-extra-removable
--no-nvram

--no-nvram prevents GRUB from updating firmware Boot* NVRAM variables. Use it when the Linux boot path should remain selectable by firmware disk choice or fallback path rather than by a newly written firmware boot entry.

Debian’s grub-install manpage documents --removable, --force-extra-removable, and --no-nvram. [12]

Use these commands only on the Linux ESP.

Do not run them against the Windows ESP.

⸻

9. systemd-boot fallback install

For a simple UEFI Linux setup, systemd-boot is a good option.

Current systemd releases commonly use:

bootctl --esp-path=/efi --variables=no install

Older releases may use:

bootctl --esp-path=/efi --no-variables install

Check the installed bootctl help before running the command. Current bootctl documents --variables=yes|no as the option controlling whether firmware boot-loader variables are touched. [13]

Expected layout:

Linux ESP:
  \EFI\BOOT\BOOTX64.EFI
  \EFI\systemd\systemd-bootx64.efi
  loader\loader.conf
  loader\entries\*.conf

Use systemd-boot when the setup is simple:

UEFI-only boot
simple partitioning
kernel/initramfs available from the ESP or boot partition
no complex GRUB module requirements

Use GRUB when the setup is more complex:

encrypted boot arrangements
Btrfs snapshot booting
LVM or RAID layouts
distribution expects GRUB
GRUB scripting or modules are required

⸻

10. Secure Boot, shim, and PreLoader

If Secure Boot is disabled, the fallback file can often be GRUB or systemd-boot directly.

If Secure Boot is enabled, the fallback file must be the first trusted component in the Linux boot chain.

Microsoft describes Secure Boot as firmware checking boot software signatures before the operating system is allowed to run. [10] Ubuntu documents a Secure Boot chain where firmware validates Microsoft-signed shim, shim validates GRUB and the kernel using distribution trust material, and unsigned or untrusted components are blocked. [11]

A typical shim-based fallback layout is:

Linux ESP:
  \EFI\BOOT\BOOTX64.EFI      ← shimx64.efi copied or installed here
  \EFI\<distro>\grubx64.efi
  \EFI\<distro>\mmx64.efi

Some layouts place GRUB and MokManager beside shim:

Linux ESP:
  \EFI\BOOT\BOOTX64.EFI      ← shim
  \EFI\BOOT\grubx64.efi
  \EFI\BOOT\mmx64.efi

Follow the distribution’s Secure Boot layout.

Do not place unsigned GRUB at:

\EFI\BOOT\BOOTX64.EFI

on a Secure Boot system.

If using PreLoader instead of shim, place PreLoader at:

\EFI\BOOT\BOOTX64.EFI

The fallback path should contain the first EFI binary the firmware is expected to trust and execute.

⸻

11. Secure Boot key state

For a conservative Windows/Linux dual-boot system, prefer the platform’s stock Secure Boot trust structure unless there is a specific production reason to change it.

Recommended target, where supported by the firmware and Linux distribution:

Stock PK.
Stock KEK.
Stock or current DBX.
DB contains the Windows UEFI CA 2023 certificate required for current Windows boot.
DB contains the Microsoft Option ROM UEFI CA 2023 certificate if the hardware requires signed option ROM support.
DB contains only the Linux OS vendor certificate required for the selected Linux boot chain, if that distribution supports direct firmware DB trust.
No broad third-party certificates unless required by the selected boot chain.
No unrelated custom certificates.

This is a recommendation, not a universal requirement.

Some Linux distributions use a Microsoft-signed shim path rather than direct firmware trust of a Linux vendor key. If the selected distribution requires shim, use the distribution-supported shim path. Do not invent a custom key structure to avoid shim unless the machine is being managed under a disciplined Secure Boot key-management process.

Microsoft’s Secure Boot key guidance documents the Secure Boot key hierarchy, including PK, KEK, DB, and DBX, and describes Microsoft’s 2023 Secure Boot certificate transition material. [10]

Avoid generating your own PK, KEK, or DB key hierarchy for ordinary dual boot. Secure Boot key ownership and private-key handling belong to OEM, enterprise, or dedicated platform-security workflows. Mismanaged keys can make firmware updates, option ROMs, operating-system loaders, recovery media, or external boot devices unusable.

⸻

12. Do not chainload Windows from Linux

Do not configure GRUB, systemd-boot, rEFInd, shim, or another Linux-side boot manager to start Windows as the normal path.

Chainloading Windows through a Linux-side bootloader is not recommended because Windows and Linux have different expectations about the boot chain, update path, Secure Boot state, recovery behavior, and boot-file ownership.

Treat GRUB → Windows Boot Manager as outside the normal Windows vendor-documented boot path. Microsoft’s documented Windows boot-file tooling centers on Windows Boot Manager, BCDBoot, BCD, system partitions, and Windows boot repair paths. [4]

Use:

Firmware → Windows Boot Manager → Windows

Use a separate Linux path for Linux:

Firmware → Linux bootloader → Linux

⸻

13. Choosing Linux from Windows

To boot Linux once from Windows:

Settings → System → Recovery → Advanced startup → Restart now

After Windows enters recovery:

Use a device

The exact label depends on the firmware.

It may appear as:

Other OS
UEFI OS
Linux Boot Manager
the Linux disk model
the NVMe/SATA drive name

Windows Recovery Environment documentation describes recovery tools, command prompt access, and device boot behavior for UEFI systems. [19]

Use this for occasional Linux boots instead of permanently making a Linux bootloader responsible for Windows.

⸻

14. BitLocker rule before planned boot changes

Before planned changes to firmware settings, Secure Boot state, DB, DBX, SBAT, bootloaders, EFI partitions, boot order, disk boot layout, Credential Guard, VBS, or Secure Boot servicing state, suspend BitLocker for a finite reboot window.

Use:

manage-bde -protectors -disable C: -RebootCount 3

Do not use -RebootCount 0 as the normal maintenance recommendation.

Microsoft documents -rebootcount values from 0 to 15, with 0 meaning indefinite suspension. [7]

3 is a practical default for boot-maintenance work because Secure Boot, firmware, VBS, Credential Guard, and related servicing changes may require more than one restart. Microsoft’s Secure Boot revocation guidance notes that a restart might be required when Virtual Secure Mode is enabled, including Credential Guard, Device Guard, and Windows Hello. [8]

If the planned procedure requires more restarts, choose a larger finite count.

After the work is complete, boot Windows normally and verify BitLocker protection has resumed:

manage-bde -status C:

Before changing boot state, confirm the BitLocker recovery key is available outside the machine.

⸻

15. Firmware boot-order changes are boot changes

Changing boot order in firmware setup is still a boot-state change.

On UEFI systems, firmware boot selection is represented through variables such as BootOrder and Boot####. Linux tools such as efibootmgr and many firmware setup screens operate on the same class of firmware boot variables. [1]

For BitLocker-protected Windows systems, treat firmware boot-order edits like bootloader or EFI-variable work. Microsoft documents boot-order and boot-manager changes as BitLocker-relevant recovery conditions in supported scenarios. [6]

Operational rule:

Do not repeatedly change firmware boot order.
Do not use efibootmgr casually on a BitLocker system.
Do not assume firmware setup changes are safer than operating-system boot-variable changes.
Suspend BitLocker before planned boot-order edits.
Use one-time boot selection where possible.

⸻

16. Hardware clock

Dual boot can expose disagreement about the hardware clock.

Linux systems commonly prefer the RTC to store UTC. Windows deployments have historically treated the RTC as local time unless explicitly configured otherwise.

Keep this issue separate from the boot-integrity design. If the clock changes after switching operating systems, decide on one RTC policy and apply it consistently. Do not treat clock drift as evidence of bootloader failure.

⸻

17. Secure Boot update order

When Secure Boot is enabled on a Windows/Linux dual-boot system, update every boot path before applying revocations or certificate-transition changes.

Recommended order:

1. Update all Linux installations.
2. Confirm shim, PreLoader, GRUB, systemd-boot, kernels, initramfs files, UKIs, and rescue media are current.
3. Boot each Linux installation once with Secure Boot enabled.
4. Confirm fwupd is not staging UEFI Secure Boot database updates, especially DBX updates.
5. Update all Windows installations.
6. Update Windows recovery and install media.
7. Suspend BitLocker for a finite reboot window before planned Secure Boot, firmware, DB, DBX, SVN, VBS, or boot-manager changes.
8. Apply Windows Secure Boot servicing, certificate-transition, DBX, or SVN changes.
9. Reboot Windows until servicing is complete.
10. Test Windows.
11. Test Linux.

The Linux-first step prevents the Linux side from depending on stale Secure Boot components before Windows applies or stages stricter boot trust.

The Windows update step remains mandatory. Microsoft’s Windows Boot Manager revocation guidance says devices with multiple operating systems should update all Windows operating systems before revocations are applied. Microsoft also warns that unupdated Windows versions may be unable to start after revocations. [8]

Do not let fwupd apply Secure Boot DBX updates in parallel with Windows DB, DBX, SVN, or Windows Boot Manager transition work. The fwupd uefi-dbx plugin exists to update the UEFI revocation database and notes that DBX updates can block vulnerable EFI binaries. It also notes that firmware update delivery of DBX changes has bricking risk if the bootloader is not updated first. [14]

Operational rule:

Update bootloaders first.
Stage revocations second.
Apply one Secure Boot database transition path at a time.
Retest every boot path before proceeding.

⸻

Windows installation safety

18. Protect the Linux ESP before installing Windows

Before installing Windows on a machine that already has Linux, create Linux recovery media and confirm that it boots.

Required recovery item:

current Linux live USB or installer USB
ability to mount the Linux root filesystem
ability to reinstall the Linux bootloader to the Linux ESP
BitLocker recovery key, if Windows already exists on another disk

Windows deployment tooling expects a system partition on UEFI/GPT systems. Microsoft describes the EFI System Partition as the system partition used for boot, formatted FAT32, managed by the operating system, and not intended to contain other files. [3] Microsoft’s BCDBoot tooling copies boot files to the EFI System Partition or to the partition explicitly specified with /s. It can also create or update Windows Boot Manager firmware entries. [4]

That creates a destructive interaction during Windows installation:

If Windows Setup or Windows boot servicing sees an existing ESP,
it may use that ESP for Windows boot files instead of creating a new Windows-only ESP.

If the visible ESP belongs to Linux, Windows installation or repair can overwrite, replace, or disturb Linux boot files. The safest option is to prevent Windows Setup from seeing the Linux ESP.

Preferred order:

1. Disconnect the Linux disk before installing Windows.
2. If the disk cannot be disconnected, disable it in firmware if the board supports that.
3. If it must remain visible, temporarily change the Linux ESP partition type away from EFI System Partition.
4. Restore the Linux ESP partition type after Windows installation.
5. Boot Linux recovery media and reinstall or repair the Linux bootloader if needed.

For a GPT disk, changing the partition type is less destructive than formatting the partition. Formatting the Linux ESP removes bootloader files and requires bootloader repair.

Example Linux sgdisk pattern:

# Before Windows installation:
# Change Linux ESP from EFI System Partition to Linux filesystem.
sgdisk --typecode=<N>:8300 /dev/sdX
# After Windows installation:
# Restore Linux ESP type.
sgdisk --typecode=<N>:EF00 /dev/sdX

Replace <N> with the Linux ESP partition number and /dev/sdX with the correct Linux disk.

Do not run these commands on the Windows disk.

If the Linux ESP was formatted, deleted, or overwritten, boot Linux recovery media and reinstall the Linux bootloader to the Linux disk’s own ESP before relying on the machine.

⸻

19. Windows install media from Linux

Use a two-partition GPT USB layout when the Windows image exceeds FAT32 limits.

Recommended layout:

Partition 1: FAT32
  GPT type: Microsoft Basic Data preferred
  fallback GPT type: EFI System Partition if firmware refuses to boot
  contents: UEFI boot files and Windows setup boot environment
Partition 2: exFAT preferred, NTFS acceptable
  GPT type: Microsoft Basic Data
  contents: full Windows ISO contents, including large install.wim or install.esd

The file system and the GPT partition type are separate requirements.

The boot partition must be firmware-readable. Use FAT32. Start with Microsoft Basic Data for the first partition because it is easier to reuse, format, and manage from Windows after installation. If the target firmware refuses to boot the USB, change the first partition’s GPT type to EFI System Partition and retest.

The image partition must be Windows-readable after WinPE or Windows Setup starts. Use Microsoft Basic Data as the GPT type. Do not leave the image partition marked as Linux filesystem. Windows may refuse to mount, auto-assign, or surface an exFAT partition correctly if the partition type identifies it as Linux data rather than Microsoft Basic Data.

Use exFAT for the second partition when creating the USB from Linux. exFAT supports large files and is usually the lower-friction cross-platform data partition for Linux-authored removable media. Use NTFS when the Linux environment has known-good NTFS write support or when following Microsoft’s WinPE deployment model. Microsoft’s WinPE USB example uses FAT32 for the boot partition and NTFS for the image partition. [15]

Acceptable variants:

FAT32 Microsoft Basic Data + exFAT Microsoft Basic Data
FAT32 Microsoft Basic Data + NTFS Microsoft Basic Data
FAT32 ESP + exFAT Microsoft Basic Data
FAT32 ESP + NTFS Microsoft Basic Data
FAT32-only USB + split-WIM files

Preferred Linux-authored layout:

FAT32 Microsoft Basic Data + exFAT Microsoft Basic Data

Firmware fallback layout:

FAT32 ESP + exFAT Microsoft Basic Data

Example Linux sgdisk type codes:

# Partition 1: Microsoft Basic Data
sgdisk --typecode=1:0700 /dev/sdX
# Partition 2: Microsoft Basic Data
sgdisk --typecode=2:0700 /dev/sdX

If the firmware refuses to boot the USB, change partition 1 to EFI System Partition:

sgdisk --typecode=1:EF00 /dev/sdX

Do not use Linux filesystem type code 8300 for either Windows setup partition.

Example parted intent with Microsoft Basic Data first:

parted /dev/sdX --script mklabel gpt
parted /dev/sdX --script mkpart WINBOOT fat32 1MiB 1025MiB
parted /dev/sdX --script mkpart WINSETUP 1025MiB 100%

Then verify both partitions are Microsoft Basic Data. Some Linux tools default new data partitions to Linux filesystem unless told otherwise.

If firmware refuses to boot the FAT32 Microsoft Basic Data partition, set the ESP flag on partition 1 and retest:

parted /dev/sdX --script set 1 esp on

Microsoft documents split-WIM deployment for FAT32-only media. The two-partition method avoids modifying the Windows image, but the target firmware and Windows recovery environment still need to be tested before erasing the destination disk. [16]

⸻

20. Fixed-drive USB and ESP deadlock

Some USB devices present themselves as fixed drives rather than removable drives.

If Windows Setup or Windows boot servicing treats the installation media as a fixed drive and the media contains an EFI System Partition, Windows may place boot files on the USB media’s ESP instead of creating or using the intended ESP on the internal Windows disk.

Risk condition:

Windows install USB presents as fixed disk.
USB partition 1 is marked EFI System Partition.
Windows Setup sees that ESP during install or repair.
Windows boot files are written to the USB ESP.
Internal Windows disk is left without the intended independent Windows ESP.

Use Microsoft Basic Data for the USB boot partition first. Switch it to ESP only if firmware refuses to boot it as Microsoft Basic Data.

If the drive presents as fixed and the firmware refuses to boot anything except an ESP, use this sequence:

1. Mark USB partition 1 as ESP.
2. Boot the USB.
3. Enter the Windows recovery environment.
4. Open Command Prompt.
5. Delete the USB’s on-drive ESP after WinPE or WinRE has loaded.
6. Launch Windows Setup manually from the second partition.

From the recovery Command Prompt, identify the USB disk and delete the USB ESP:

diskpart
list disk
select disk X
list partition
select partition Y
delete partition override
exit

Replace X with the USB disk number and Y with the USB ESP partition number.

Do not delete an internal disk ESP.

The delete partition override operation is destructive. Use it only after confirming that the selected disk is the installation USB and the selected partition is the USB boot ESP.

⸻

21. Launching Windows Setup from the recovery environment

For this Linux-authored two-partition USB workflow, do not rely on the initial installer path if the goal is to control which ESP Windows Setup can see and use.

Boot the USB, then choose:

Repair your computer
Troubleshoot
Advanced options
Command Prompt

Use Command Prompt to identify drive letters.

First option:

diskpart
list volume
exit

Second option:

notepad

In Notepad, use:

File → Open → This PC

Look at the visible drive letters and labels. Find the partition that contains the full Windows installation source.

If the source partition is E:\, close Notepad and run:

E:\setup.exe

Microsoft’s recovery documentation describes the Windows Recovery Environment and its command-line recovery tools. [19] Microsoft’s BCDBoot documentation also uses diskpart, list volume, and drive-letter identification as part of boot-file repair workflows. [4]

⸻

22. If Windows Setup keeps choosing the wrong ESP

Do not keep repeating the same GUI installation path if Windows Setup keeps selecting an existing Linux ESP or the installation media’s ESP.

Use one of these alternatives:

1. Disconnect or hide the Linux disk and retry Windows Setup.
2. Temporarily change the Linux ESP partition type away from EFI System Partition and retry.
3. If the USB is fixed-drive and ESP-only boot is required, delete the USB ESP from WinRE after boot and before launching setup.exe.
4. Use the VM-based Windows installation path.
5. Install Windows manually from WinPE using DiskPart, DISM, and BCDBoot.

The manual DiskPart/DISM/BCDBoot path is separate deployment work. Use it only with a procedure that explicitly covers image application, ESP creation, BCD creation, and firmware boot validation.

⸻

23. Reusing a USB after it contains an ESP

After booting Windows or WinPE from a USB that contains an EFI System Partition, normal Windows formatting tools may not offer a clean one-click way to return the whole USB to a simple single data partition.

Use DiskPart from Windows, WinPE, or WinRE:

diskpart
list disk
select disk X
clean
convert gpt
create partition primary
select partition 1
format quick fs=exfat
assign
exit

Replace X with the USB disk number.

clean destroys the selected disk’s partition table. Microsoft documents that on GPT disks, clean overwrites GPT partitioning information, including the protective MBR. Confirm the disk number before running it. [18]

If Windows tooling cannot or should not be used, wipe the USB partition table from Linux and recreate it there. The important operation is removing the old GPT layout, including any ESP, before creating the new data partition.

Microsoft’s create partition primary documentation states that if no partition type is specified on GPT, DiskPart creates a basic data partition. [17]

⸻

Troubleshooting and recovery

24. Unexpected Secure Boot failures

Do not disable Secure Boot as a generic first response to a random boot failure on an established trusted system.

If a previously working system suddenly fails Secure Boot, treat the failure as integrity-relevant until the cause is identified.

Use this sequence:

1. Keep Secure Boot enabled.
2. Do not enter BitLocker recovery keys into an unexpected or untrusted boot path.
3. Do not blindly repair BCD.
4. Do not delete ESP files randomly.
5. Boot only known-good, current, Secure-Boot-compatible recovery media.
6. Distinguish update mismatch from boot-chain tampering before changing policy.

Temporary Secure Boot disablement is acceptable during planned installation or controlled recovery. It is not the default response to an unexplained boot-chain failure.

⸻

25. Windows is skipped and Linux boots instead

If Windows is first in firmware boot order but the machine boots Linux, do not immediately assume GRUB, BCD, or the Windows ESP is the primary fault.

On Secure Boot systems, firmware may try Windows first, reject the Windows boot path, and continue to the next available boot option. If Linux is next and its boot chain is trusted, Linux may boot normally.

The symptom may be:

Windows is first in firmware boot order.
Linux starts anyway.
No obvious Windows error appears.

Possible causes include:

Windows Boot Manager is stale relative to current DB, DBX, SVN, or certificate state.
Firmware rejected Windows Boot Manager.
SkuSiPolicy.p7b is missing or mismatched under an existing UEFI lock.
Windows ESP files are damaged.
Firmware boot behavior is unusual.
The boot chain may have been tampered with.

First classify the situation:

planned installation or known recovery
known Secure Boot servicing transition
unexpected failure on an established trusted system

Only the first two cases permit normal controlled-recovery procedures.

⸻

26. Windows Boot Manager revocation recovery

For Secure Boot certificate-transition, DBX, SVN, or Windows Boot Manager revocation recovery, rebuild Windows boot files with /bootex.

Use:

bcdboot C:\Windows /f UEFI /s S: /bootex

This is the required form for this recovery class.

Do not rebuild the Windows ESP for a revocation-transitioned system with ordinary BCDBoot syntax. Without /bootex, BCDBoot can service the ESP with the non-bootex Windows Boot Manager path, which can restore an older or wrong boot path for the system’s current Secure Boot state.

Microsoft documents /bootex as the BCDBoot option that uses bootex binaries for servicing when the necessary conditions are met. Microsoft links this option to the Windows Boot Manager revocations associated with CVE-2023-24932. [4][8]

Use /bootex for:

Windows Boot Manager revocation recovery
Windows UEFI CA 2023 transition work
DB/DBX/SVN Secure Boot transition recovery
CVE-2023-24932 boot media servicing
revocation-aware Windows ESP reconstruction

Do not use /bootex as generic BCD repair for unrelated boot failures.

⸻

27. /offline /bootex recovery

If Secure Boot has been disabled to recover a machine whose old Windows certificate or boot manager is no longer trusted by the current Secure Boot policy, use the offline bootex path when the installed Windows build and recovery source support it:

bcdboot C:\Windows /f UEFI /s S: /offline /bootex

Microsoft documents /offline as forcing boot-file servicing to be handled offline. Microsoft also states that boot-file selection is forced based on the presence of the bootex switch. [4]

Use /offline /bootex when all of the following are true:

The failure is in the Secure Boot revocation or certificate-transition class.
Secure Boot was disabled to allow recovery work.
The Windows installation or recovery source supports /offline.
The source contains the required bootex binaries.
The Windows partition and Windows ESP have been identified correctly.

Microsoft documents /offline support beginning with Windows 11 version 24H2 build 26100.8037 and Windows 11 version 25H2 build 26100.8037. [4]

Do not use an old Windows ISO, stale recovery drive, or unpatched WinPE source for this recovery class. If the source does not contain the required bootex files, the recovery source is wrong for the current Secure Boot state.

⸻

28. SkuSiPolicy.p7b recovery condition

SkuSiPolicy.p7b is a Microsoft-signed revocation policy used for VBS rollback protection. It is not generic BCD repair.

A specific recovery condition exists:

A prior Windows installation applied the SkuSiPolicy.p7b UEFI lock.
The Windows installation was erased, restored, replaced, or rebuilt.
The firmware still carries the policy lock.
The Windows ESP no longer contains the matching policy file.
Windows Boot Manager fails without a visible boot error.
Firmware proceeds to the next available boot option.

Microsoft documents that applying SkuSiPolicy.p7b locks the policy to the device by adding a UEFI firmware variable. Microsoft also states that if the UEFI lock is applied and the policy is removed or replaced with an older version, Windows Boot Manager will not start. [9]

Microsoft further documents that reformatting the disk does not remove the UEFI lock, and that reverting to an earlier Windows state can cause the device not to start, show no error message, and proceed to the next available boot option. [9]

In that condition, restore the current policy file from the installed Windows image to the Windows ESP.

Source:

C:\Windows\System32\SecureBootUpdates\SkuSiPolicy.p7b

Destination:

<Windows ESP>\EFI\Microsoft\Boot\SkuSiPolicy.p7b

PowerShell pattern from a trusted, fully updated Windows environment:

$PolicyBinary = $env:windir+"\System32\SecureBootUpdates\SkuSiPolicy.p7b"
$MountPoint = 's:'
$EFIDestinationFolder = "$MountPoint\EFI\Microsoft\Boot"
mountvol $MountPoint /S
if (-Not (Test-Path $EFIDestinationFolder)) {
    New-Item -Path $EFIDestinationFolder -Type Directory -Force
}
Copy-Item -Path $PolicyBinary -Destination $EFIDestinationFolder -Force
mountvol $MountPoint /D

Use the policy file from the installed Windows image being booted.

Do not copy an old policy file from stale install media, another machine, or an older Windows build.

Do not describe this as a Linux Secure Boot repair. It is a Windows VBS rollback-protection recovery condition.

⸻

Additional installation options

29. Installing Windows in a VM onto a physical disk

This is an advanced installation path. Use it when normal installation is impractical or when Windows Setup repeatedly selects the wrong ESP.

If Windows is installed in a VM and then moved to real hardware, generalize it before the first bare-metal boot.

At the Windows OOBE screen, press:

Ctrl + Shift + F3

Windows will reboot into Audit Mode. Microsoft documents this keyboard shortcut for booting to Audit Mode. [21]

The System Preparation Tool opens automatically.

Select:

System Cleanup Action:
  Enter System Out-of-Box Experience (OOBE)
Generalize:
  checked
Shutdown Options:
  Shutdown

Then shut down the VM and boot the disk on the real machine.

Microsoft documents that Sysprep /generalize removes unique information from a Windows installation so the image can be reused on another computer. [20]

Be prepared to use Microsoft’s boot repair procedures if the real firmware does not discover the disk automatically.

⸻

30. Installing Linux in a VM onto a physical disk

If Linux is installed in a VM and then moved to bare metal, make sure the Linux bootloader is installed to the Linux disk’s own ESP.

Use the fallback path:

\EFI\BOOT\BOOTX64.EFI

Use a fallback or all-drivers initramfs for the first bare-metal boot when the distribution supports that mode.

After Linux successfully boots on the real machine, regenerate the initramfs normally.

A VM install may generate an initramfs optimized for virtual storage and chipset devices. A broader first-boot initramfs reduces the chance that real-hardware storage drivers are missing during the first bare-metal boot.

Follow the selected distribution’s documented recovery and initramfs tooling. The exact commands vary across distributions.

⸻

Daily operation

31. Daily use policy

Default boot should be Windows.

To boot Linux, use one of:

Windows Recovery → Use a device
firmware one-time boot menu
firmware disk selector

Do not make GRUB chainload Windows.

Do not repeatedly edit EFI variables unless necessary.

Do not repeatedly change firmware boot order unless BitLocker has been suspended for a planned boot-maintenance window.

Do not share ESPs unless both systems can be recovered manually.

Keep these recovery items available:

BitLocker recovery key
current Windows install or recovery USB
current Linux live USB
knowledge of controlled Secure Boot disablement for recovery
firmware boot-menu key for the machine

⸻

32. Testing checklist

After installation, verify all of this before relying on the machine:

Windows boots by default.
Windows BitLocker does not ask for recovery during normal boot.
Linux boots from the firmware one-time boot menu.
Linux boots from Windows Recovery → Use a device, if exposed by the firmware.
Secure Boot status is as expected.
Linux still boots after Secure Boot is enabled.
Windows still boots after Linux installation.
Linux still boots after Windows updates.
Both ESPs are on the intended disks.
The Linux bootloader did not overwrite the Windows ESP.
The Windows fallback path and Linux fallback path are on separate ESPs.
The Windows hardware clock stays correct after booting Linux.
The Linux hardware clock stays correct after booting Windows.
fwupd is not staging DBX updates during Windows Secure Boot transition work.
Windows recovery media has been updated before revocations are applied.
Linux recovery media still boots after Secure Boot updates.

⸻

33. Operational summary

Choose the lower-coupling boot design.

Separate disks.
Separate ESPs.
Windows first.
Windows manages Windows Boot Manager.
Linux independently bootable.
Fallback loader on the Linux disk.
Shim or PreLoader in the Linux fallback path when Secure Boot requires it.
No GRUB → Windows chainloading.
Minimal cross-OS EFI-variable manipulation.
Minimal firmware boot-order edits.
Update all Linux boot chains before Windows Secure Boot revocation work.
Hold fwupd DBX updates during Windows Secure Boot transition work.
Update all Windows installations and Windows recovery media before applying revocations.
Use stock Secure Boot platform keys where possible.
Do not generate custom Secure Boot keys for ordinary dual boot.
Suspend BitLocker for a finite reboot window before planned risky boot work.
Use /bootex for Windows Boot Manager revocation or certificate-transition recovery.
Use /offline /bootex when Secure Boot is disabled for supported offline recovery.
Restore SkuSiPolicy.p7b only when a missing or mismatched policy file is the identified recovery condition.
Protect the Linux ESP before Windows installation.
Use Linux recovery media before changing ESP partition types.

This layout minimizes avoidable dependencies between Windows security state, Linux boot tooling, firmware behavior, EFI variables, Secure Boot policy, and recovery paths.

⸻

References

[1] UEFI Specification 2.11, Boot Manager.
https://uefi.org/specs/UEFI/2.11/03_Boot_Manager.html

[2] UEFI Specification 2.11, Protocols: Media Access.
https://uefi.org/specs/UEFI/2.11/13_Protocols_Media_Access.html

[3] Microsoft Learn, UEFI/GPT-based hard drive partitions.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions

[4] Microsoft Learn, BCDBoot Command-Line Options.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdboot-command-line-options-techref-di

[5] Microsoft Learn, How Windows uses the TPM.
https://learn.microsoft.com/en-us/windows/security/hardware-security/tpm/how-windows-uses-the-tpm

[6] Microsoft Learn, BitLocker recovery overview.
https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/bitlocker/recovery-overview

[7] Microsoft Learn, manage-bde protectors.
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde-protectors

[8] Microsoft Support, Windows Boot Manager revocations for Secure Boot changes associated with CVE-2023-24932.
https://support.microsoft.com/en-us/topic/how-to-manage-the-windows-boot-manager-revocations-for-secure-boot-changes-associated-with-cve-2023-24932-41a975df-beb2-40c1-99a3-b3ff139f832d

[9] Microsoft Support, Guidance for blocking rollback of VBS-related security updates.
https://support.microsoft.com/en-us/topic/guidance-for-blocking-rollback-of-virtualization-based-security-vbs-related-security-updates-b2e7ebf4-f64d-4884-a390-38d63171b8d3

[10] Microsoft Learn, Windows Secure Boot key creation and management guidance.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-secure-boot-key-creation-and-management-guidance

[11] Ubuntu documentation, Secure Boot.
https://documentation.ubuntu.com/security/security-features/platform-protections/secure-boot/

[12] Debian manpages, grub-install.
https://manpages.debian.org/unstable/grub2-common/grub-install.8.en.html

[13] systemd bootctl manual.
https://www.freedesktop.org/software/systemd/man/latest/bootctl.html

[14] fwupd, UEFI DBX plugin documentation.
https://github.com/fwupd/fwupd/blob/main/plugins/uefi-dbx/README.md

[15] Microsoft Learn, WinPE: Store or split images to deploy Windows using a single USB drive.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe–use-a-single-usb-key-for-winpe-and-a-wim-file—wim

[16] Microsoft Learn, Split a Windows image file.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/split-a-windows-image–wim–file-to-span-across-multiple-dvds

[17] Microsoft Learn, create partition primary.
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/create-partition-primary

[18] Microsoft Learn, DiskPart clean.
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/clean

[19] Microsoft Learn, Windows Recovery Environment technical reference.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-recovery-environment–windows-re–technical-reference

[20] Microsoft Learn, Sysprep generalize a Windows installation.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep–generalize–a-windows-installation

[21] Microsoft Learn, Boot Windows to Audit Mode or OOBE.
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/boot-windows-to-audit-mode-or-oobe

[22] Microsoft Learn, format command.
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/format

[23] Microsoft Learn, create partition EFI.
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/create-partition-efi
