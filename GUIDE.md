# Conservative Windows + Linux Dual-Boot Guide

## Separate disks, separate ESPs, fallback Linux boot path, no Windows chainloading

This material is provided for informational and educational purposes only. It is not legal, security, compliance, enterprise deployment, or incident-response advice.

No warranty is provided. Use this material at your own risk. The author is not liable for data loss, boot failure, recovery-key lockout, firmware misconfiguration, security weakening, downtime, hardware issues, or any other damage or loss arising from use of this material.

Product names, project names, company names, logos, and trademarks mentioned in this article belong to their respective owners.

This article is independent. It is not affiliated with, sponsored by, approved by, or endorsed by any operating-system vendor, Linux distribution, bootloader project, firmware vendor, hardware vendor, standards body, or software publisher mentioned.

For production, regulated, enterprise, or attested environments, follow the applicable vendor documentation and internal change-control process.

---

## 1. Scope

Modern dual boot is a boot-integrity design problem on systems that use Secure Boot, TPM, BitLocker, device encryption, Credential Guard, VBS, anti-cheat, firmware attestation, or enterprise security baselines.

The conservative layout is:

```text id="y92dp9"
Disk 0: Windows disk
  EFI System Partition
  Microsoft Reserved Partition
  Windows partition
  Windows Recovery partition

Disk 1: Linux disk
  EFI System Partition
  Linux root partition
  optional /home or swap
```

Each operating system owns its own EFI System Partition.

Windows starts through Windows Boot Manager. Linux starts through its own Linux bootloader. Firmware chooses which operating system to start.

Use:

```text id="hcviws"
Firmware → Windows Boot Manager → Windows
Firmware → Linux bootloader → Linux
```

Avoid this as the normal Windows path:

```text id="ez4v20"
Firmware → Linux bootloader → Windows Boot Manager → Windows
```

UEFI firmware has a boot manager that uses boot variables such as `BootOrder` and `Boot####` to load EFI applications from bootable devices. That makes firmware-level OS selection available without requiring one operating system’s bootloader to start the other operating system. [1]

---

## 2. Core design

Use separate physical disks whenever possible.

The conservative layout is:

```text id="e21n2c"
Disk 0: Windows disk
  EFI System Partition
  Microsoft Reserved Partition
  Windows partition
  Windows Recovery partition

Disk 1: Linux disk
  EFI System Partition
  optional whole-device or whole-LVM-device encryption, if confidentiality requires it
  LVM physical volume
    logical volume: root
    optional logical volume: home
    optional logical volume: swap
    optional logical volumes for data, snapshots, or future Linux installations
```

Each operating system owns its own EFI System Partition.

The Linux ESP remains a normal firmware-readable FAT32 EFI System Partition. Do not place the ESP inside LVM.

Use LVM for the Linux system unless the selected distribution has a deliberate alternative layout. LVM makes later partition management easier because root, `/home`, swap, data volumes, and future Linux installations can be resized or separated without repartitioning the whole disk.

If Linux data confidentiality is required, the Linux LVM device may be encrypted as a whole. In that layout, the normal structure is:

```text id="vrs053"
Linux disk:
  EFI System Partition
  encrypted container
    LVM physical volume
      logical volume: root
      optional logical volume: home
      optional logical volume: swap
      optional logical volumes for data, snapshots, or future Linux installations
```

Do not share the Windows ESP with Linux unless both boot paths can be repaired manually.

Do not make GRUB, systemd-boot, rEFInd, shim, or another Linux-side bootloader responsible for starting Windows.

Windows should remain the default boot target. Linux should be selected when needed through the firmware one-time boot menu, firmware disk selector, or Windows Recovery’s device boot path.

Microsoft’s UEFI/GPT partition guidance describes the EFI System Partition as the system partition used for boot, formatted FAT32, managed by the operating system, and not intended to contain other files. [3]

---

## 2A. Linux encryption, boot integrity, and sensitive initramfs contents

Linux disk encryption and Linux boot integrity solve different problems.

Use disk encryption when the requirement is data confidentiality at rest. The Linux LVM device may be encrypted as a whole, with LVM placed inside the encrypted container:

```text id="6hs1g7"
Linux disk:
  EFI System Partition
  encrypted container
    LVM physical volume
      logical volume: root
      optional logical volume: home
      optional logical volume: swap
      optional logical volumes for data, snapshots, or future Linux installations
```

Do not use an encrypted `/boot` partition as the default answer to sensitive boot artifacts on modern UEFI systems.

For boot-artifact integrity, rely on a validated Secure Boot chain. Firmware should validate the first trusted Linux boot component. The Linux boot chain should then validate the next components it loads, according to the selected distribution’s Secure Boot model.

If the initramfs contains sensitive host material, address that directly. Valid approaches include:

```text id="de1f5e"
TPM-bound sealing for secrets required during early boot
hardware-backed encryption where the platform provides it
host-specific credential design that avoids placing reusable system secrets in the initramfs
a dedicated passwd/shadow arrangement where the booted host does not reuse the same sensitive credential material
```

For LUKS2 TPM-bound unlock workflows, systemd-cryptenroll is one available tool for enrolling TPM2 tokens. Use the selected distribution’s documented recovery and key-enrollment procedure before relying on TPM-bound unlocks. [28]

Do not treat ordinary dm-crypt encryption as boot-artifact tamper protection. The Linux kernel documents dm-crypt as transparent block-device encryption. Authenticated disk encryption requires integrity metadata supplied by an integrity layer such as dm-integrity, or an authenticated mode configured with the required integrity support. The kernel dm-integrity documentation states that dm-crypt with dm-integrity can provide authenticated disk encryption where modified encrypted sectors produce an I/O error rather than unauthenticated decrypted data. [26][27]

Operational rule:

```text id="p5x3qw"
Use Secure Boot validation for boot-chain integrity.
Use disk encryption for Linux data confidentiality.
Use TPM-bound or hardware-backed protection for early-boot secrets.
Use dm-integrity or equivalent authenticated storage design only when ciphertext modification detection is required.
Do not rely on encrypted /boot as the default integrity model.
```

---

## 3. Why this layout is used

UEFI allows multiple EFI applications, multiple boot entries, and multiple EFI System Partitions. The UEFI specification also states that it does not restrict the number or location of system partitions and that coordination between operating systems sharing one ESP is outside the scope of the specification. [2]

That flexibility does not make every boot arrangement equally robust.

Real systems vary. Firmware may lose, rewrite, hide, or reorder boot entries. Board menus may expose disk labels instead of loader names. Some systems expose only one boot path clearly. Secure Boot and TPM-dependent Windows features rely on early boot state remaining within expected boundaries.

Microsoft documents that Windows uses the TPM for system integrity measurements and that BitLocker can depend on expected startup measurements before releasing the encryption key normally. [5] Microsoft also lists boot-manager changes, firmware changes, BCD changes, Secure Boot changes, and PCR changes among BitLocker recovery triggers. [6]

Operational rule:

```text id="jgm6fm"
Windows boots through the Windows path.
Linux boots through the Linux path.
Neither operating system depends on the other operating system’s bootloader.
EFI variable churn is minimized.
Each disk remains independently recoverable.
```

The design reduces cross-OS boot coupling.

---

## 4. UEFI boot entries and fallback paths

UEFI systems commonly boot through firmware boot entries stored in NVRAM. Those entries point to EFI executables such as:

```text id="3u8pdo"
\EFI\Microsoft\Boot\bootmgfw.efi
\EFI\systemd\systemd-bootx64.efi
\EFI\GRUB\grubx64.efi
```

On x86_64 systems, the standard fallback filename is:

```text id="lhskxd"
\EFI\BOOT\BOOTX64.EFI
```

UEFI defines the `\EFI\BOOT\BOOT{machine type short name}.EFI` fallback pattern in its boot-manager behavior. For removable media, UEFI also specifies one UEFI-compliant system partition and one executable EFI image per supported processor architecture in the `BOOT` directory. [1][2]

Many systems also honor this fallback path when a physical internal disk is selected from a firmware boot menu. Fixed-disk behavior remains firmware-dependent unless the boot option or firmware path clearly targets the fallback loader.

Use the Linux disk fallback path as a compatibility measure:

```text id="0sca2d"
Linux disk ESP:
  \EFI\BOOT\BOOTX64.EFI
  plus the files required by the Linux bootloader
```

A shared ESP creates a single fallback-loader filename for each architecture. Only one file can occupy this path on a single ESP:

```text id="zblpja"
\EFI\BOOT\BOOTX64.EFI
```

That filename collision is one reason to keep Windows and Linux on separate ESPs.

---

## 5. Board-specific boot behavior

The standard x86_64 fallback filename is:

```text id="r9psbb"
\EFI\BOOT\BOOTX64.EFI
```

The normal Windows Boot Manager path is:

```text id="c4q6ja"
\EFI\Microsoft\Boot\bootmgfw.efi
```

Firmware menus may label entries inconsistently.

Possible labels include:

```text id="12mhzr"
Windows Boot Manager
UEFI OS
Other OS
Linux Boot Manager
NVMe drive
SATA drive
the physical drive model
```

Minimum validation:

```text id="4594jv"
1. Boot Windows normally.
2. Boot Linux from the firmware one-time boot menu.
3. Boot Linux from Windows Recovery → Use a device, if exposed by the firmware.
4. Power off fully.
5. Cold boot.
6. Confirm Windows still boots first by default.
7. Confirm Linux is still reachable.
```

Both disks should remain independently bootable. The board determines how clearly those paths are exposed.

---

## 6. Windows boot layout

Windows should have its own ESP.

A normal Windows UEFI installation should manage its own Windows Boot Manager entry and boot files.

The normal Windows boot path is:

```text id="1049s7"
\EFI\Microsoft\Boot\bootmgfw.efi
```

Do not make GRUB, systemd-boot, rEFInd, shim, or Linux `efibootmgr` responsible for starting Windows.

Do not use this as the normal Windows boot-management path:

```cmd id="5votkr"
bcdboot C:\Windows /s S: /f UEFI
```

The `/s` form is useful for recovery, removable media, secondary-disk deployment, and controlled boot-file servicing. Microsoft documents that `/s` specifies the system partition and should not be used in typical deployment scenarios. Microsoft also states that when `/s` is used on UEFI systems, BCDBoot does not create the normal NVRAM firmware entry and relies on default firmware behavior instead. [4]

Operational rule:

```text id="eo0bwd"
Windows manages Windows Boot Manager.
Linux manages the Linux bootloader.
The two boot chains stay separate.
```

---

## 7. Linux boot layout

Linux should have its own ESP on the Linux disk.

The Linux ESP should contain a bootable fallback loader at:

```text id="j6vfdb"
\EFI\BOOT\BOOTX64.EFI
```

That fallback loader may be GRUB, systemd-boot, shim, PreLoader, or another distribution-supported first-stage loader, depending on Secure Boot state.

Do not install Linux boot files to the Windows ESP.

Do not overwrite the Windows fallback loader.

---

## 8. GRUB fallback install

For a Linux disk that should boot independently, install GRUB only to the Linux ESP.

Generic fallback install:

```bash id="gwon0s"
grub-install \
  --target=x86_64-efi \
  --efi-directory=/efi \
  --boot-directory=/boot \
  --removable
```

Adjust `/efi` and `/boot` for the distribution’s mount layout.

Common ESP mountpoints include:

```text id="5tj1z5"
/efi
/boot
/boot/efi
```

The `--removable` option tells `grub-install` to treat the installation device as removable. On x86_64 UEFI systems, this places the bootloader at the removable-media fallback path:

```text id="e11hxf"
\EFI\BOOT\BOOTX64.EFI
```

Some GRUB builds also expose a separate EFI-only option:

```bash id="vyd3sl"
--force-extra-removable
```

Use `--force-extra-removable` when the distribution supports it and the intent is to install an additional fallback copy while retaining the normal distribution bootloader location.

For a conservative Linux-disk ESP, the relevant GRUB options are:

```text id="0ciold"
--removable
--force-extra-removable
--no-nvram
```

`--no-nvram` prevents GRUB from updating firmware `Boot*` NVRAM variables. Use it when the Linux boot path should remain selectable by firmware disk choice or fallback path rather than by a newly written firmware boot entry.

Debian’s `grub-install` manpage documents `--removable`, `--force-extra-removable`, and `--no-nvram`. [12]

Use these commands only on the Linux ESP.

Do not run them against the Windows ESP.

---

## 9. UKI-first Linux boot layout

For a simple UEFI Linux setup, prefer a distribution-provided and distribution-signed Unified Kernel Image, also called a UKI.

A UKI is a bootable EFI application, commonly using systemd-stub or a similar EFI stub, that contains the kernel, initramfs, kernel command line, and related boot metadata.

Ideal path:

```text id="qkl0iv"
Firmware or shim → signed UKI → Linux
```

In this layout, the signed UKI is the boot artifact. Firmware or shim validates the UKI before Linux starts. Because the initramfs and kernel command line are embedded in the signed UKI, they are covered by the UKI signature instead of being loose files that require separate integrity handling. Red Hat documents UKI as combining the kernel, initramfs, and kernel command line into one executable binary. Debian’s UKI documentation states that UKIs embed boot contents and that those contents, including the initrd, are covered by the PE signature and verified for trust. [31][32]

Direct fallback UKI layout:

```text id="69zc50"
Linux ESP:
  \EFI\BOOT\BOOTX64.EFI      ← signed UKI
```

If Secure Boot is enabled, the fallback path should point to the first trusted component required by the selected distribution:

```text id="dpk0nl"
Secure Boot with shim:
  \EFI\BOOT\BOOTX64.EFI      ← shim
  \EFI\BOOT\<shim-recognized-loader-name>.efi ← signed UKI or UKI launcher

Secure Boot without shim, where supported:
  \EFI\BOOT\BOOTX64.EFI      ← signed UKI
```

For the shim path, the second-stage filename and location are distribution-dependent. Some shim builds expect a conventional adjacent filename such as `grubx64.efi`, even when the file being loaded is not GRUB. Use the filename and path that the selected distribution’s shim is built or configured to load.

Second choice: use systemd-boot to select signed UKIs.

```text id="lqo9pa"
Firmware or shim → systemd-boot → signed UKI → Linux
```

In this layout, systemd-boot is a selector. The integrity boundary remains the signed UKI.

Expected components when systemd-boot is used:

```text id="tolab8"
Linux ESP:
  \EFI\systemd\systemd-bootx64.efi           ← systemd-boot
  \EFI\Linux\<distribution-provided-UKI>.efi ← signed UKI
  \EFI\BOOT\BOOTX64.EFI                      ← optional fallback copy or first-stage loader
  loader\loader.conf                         ← optional global loader configuration
  loader\entries\*.conf                      ← optional or required depending on distribution discovery support
```

Do not assume every systemd-boot installation needs every file shown above. Some distributions rely on Boot Loader Specification discovery for UKIs; others use explicit loader entries. Follow the selected distribution’s documented layout.

Use loader entries that launch signed UKIs directly. Avoid entries that reference loose kernel and initramfs files unless the selected distribution documents that layout and the boot-integrity model is understood.

A split-artifact layout can still boot correctly, but it shifts responsibility to the operator. The kernel, initramfs, kernel command line, loader entries, and TPM PCR behavior must be evaluated as separate boot artifacts. If the initramfs can be modified independently, Secure Boot validation of the first EFI binary alone does not prove that the initramfs is the expected one.

Current upstream systemd releases use:

```bash id="8gw9c9"
bootctl --esp-path=/efi --variables=no install
```

Older releases may use:

```bash id="mwqxjl"
bootctl --esp-path=/efi --no-variables install
```

Check the installed `bootctl --help` output before running the command. Current upstream `bootctl` documents `--variables=yes|no` as the option controlling whether firmware boot-loader variables are touched. Older releases document `--no-variables`. [13]

Use GRUB when the boot layout requires GRUB.

```text id="kq6ol7"
Firmware or shim → GRUB → Linux
```

GRUB is appropriate when:

```text id="djslkh"
distribution expects GRUB
Btrfs snapshot booting requires GRUB integration
LVM, RAID, or boot layout requires GRUB support
GRUB scripting or modules are required
distribution does not provide a suitable signed UKI path
systemd-boot cannot launch the desired boot artifacts cleanly
```

Operational preference order:

```text id="zcue6n"
1. Firmware or shim loads a signed UKI directly.
2. systemd-boot selects and launches signed UKIs.
3. GRUB is used when the distribution or storage layout requires it.
```

---

## 10. Secure Boot, shim, and PreLoader

If Secure Boot is disabled, the fallback file can often be GRUB or systemd-boot directly.

If Secure Boot is enabled, the fallback file must be the first trusted component in the Linux boot chain.

Microsoft describes Secure Boot as firmware checking boot software signatures before the operating system is allowed to run. [10] Ubuntu documents a Secure Boot chain where firmware validates Microsoft-signed shim, shim validates GRUB and the kernel using distribution trust material, and unsigned or untrusted components are blocked. [11]

A typical shim-based fallback layout is:

```text id="awvxlb"
Linux ESP:
  \EFI\BOOT\BOOTX64.EFI      ← shimx64.efi copied or installed here
  \EFI\<distro>\grubx64.efi
  \EFI\<distro>\mmx64.efi
```

Some layouts place GRUB and MokManager beside shim:

```text id="fve73j"
Linux ESP:
  \EFI\BOOT\BOOTX64.EFI      ← shim
  \EFI\BOOT\grubx64.efi
  \EFI\BOOT\mmx64.efi
```

Follow the distribution’s Secure Boot layout.

Do not place unsigned GRUB at:

```text id="stoxkg"
\EFI\BOOT\BOOTX64.EFI
```

on a Secure Boot system.

If using PreLoader instead of shim, place PreLoader at:

```text id="k1ds6x"
\EFI\BOOT\BOOTX64.EFI
```

The fallback path should contain the first EFI binary the firmware is expected to trust and execute.

---

## 11. Secure Boot key state

For a conservative Windows/Linux dual-boot system, prefer the platform’s stock Secure Boot trust structure unless there is a specific production reason to change it.

Recommended target, where supported by the firmware and Linux distribution:

```text id="bwyl25"
Stock PK.
Stock KEK.
Stock or current DBX.
DB contains the Windows UEFI CA 2023 certificate required for current Windows boot.
DB contains the Microsoft Option ROM UEFI CA 2023 certificate if the hardware requires signed option ROM support.
DB contains only the Linux OS vendor certificate required for the selected Linux boot chain, if that distribution supports direct firmware DB trust.
No broad third-party certificates unless required by the selected boot chain.
No unrelated custom certificates.
```

This is a recommendation, not a universal requirement.

Some Linux distributions use a Microsoft-signed shim path rather than direct firmware trust of a Linux vendor key. If the selected distribution requires shim, use the distribution-supported shim path. Do not invent a custom key structure to avoid shim unless the machine is being managed under a disciplined Secure Boot key-management process.

Microsoft’s Secure Boot key guidance documents the Secure Boot key hierarchy, including PK, KEK, DB, and DBX, and describes Microsoft’s 2023 Secure Boot certificate transition material. [10]

Avoid generating your own PK, KEK, or DB key hierarchy for ordinary dual boot. Secure Boot key ownership and private-key handling belong to OEM, enterprise, or dedicated platform-security workflows. Mismanaged keys can make firmware updates, option ROMs, operating-system loaders, recovery media, or external boot devices unusable.

---

## 12. Do not chainload Windows from Linux

Do not configure GRUB, systemd-boot, rEFInd, shim, or another Linux-side boot manager to start Windows as the normal path.

Chainloading Windows through a Linux-side bootloader is not recommended because Windows and Linux have different expectations about the boot chain, update path, Secure Boot state, recovery behavior, and boot-file ownership.

Treat GRUB → Windows Boot Manager as outside the normal Windows vendor-documented boot path. Microsoft’s documented Windows boot-file tooling centers on Windows Boot Manager, BCDBoot, BCD, system partitions, and Windows boot repair paths. [4]

Use:

```text id="qngmho"
Firmware → Windows Boot Manager → Windows
```

Use a separate Linux path for Linux:

```text id="757c4k"
Firmware → Linux bootloader → Linux
```

---

## 13. Choosing Linux from Windows

To boot Linux once from Windows:

```text id="ghz28i"
Settings → System → Recovery → Advanced startup → Restart now
```

After Windows enters recovery:

```text id="4kfrg6"
Use a device
```

The exact label depends on the firmware.

It may appear as:

```text id="eerv8n"
Other OS
UEFI OS
Linux Boot Manager
the Linux disk model
the NVMe/SATA drive name
```

Windows Recovery Environment documentation describes recovery tools, command prompt access, and device boot behavior for UEFI systems. [19]

Use this for occasional Linux boots instead of permanently making a Linux bootloader responsible for Windows.

---

## 14. BitLocker rule before planned boot changes

Before planned changes to firmware settings, Secure Boot state, DB, DBX, SBAT, bootloaders, EFI partitions, boot order, disk boot layout, Credential Guard, VBS, or Secure Boot servicing state, suspend BitLocker for a finite reboot window.

Use:

```cmd id="acoxds"
manage-bde -protectors -disable C: -RebootCount 3
```

Do not use `-RebootCount 0` as the normal maintenance recommendation.

Microsoft documents `-rebootcount` values from `0` to `15`, with `0` meaning indefinite suspension. [7]

`3` is a practical default for boot-maintenance work because Secure Boot, firmware, VBS, Credential Guard, and related servicing changes may require more than one restart. Microsoft’s Secure Boot revocation guidance notes that a restart might be required when Virtual Secure Mode is enabled, including Credential Guard, Device Guard, and Windows Hello. [8]

If the planned procedure requires more restarts, choose a larger finite count.

After the work is complete, boot Windows normally and verify BitLocker protection has resumed:

```cmd id="qewx11"
manage-bde -status C:
```

Before changing boot state, confirm the BitLocker recovery key is available outside the machine.

---

## 15. Firmware boot-order changes are boot changes

Changing boot order in firmware setup is still a boot-state change.

On UEFI systems, firmware boot selection is represented through variables such as `BootOrder` and `Boot####`. Linux tools such as `efibootmgr` and many firmware setup screens operate on the same class of firmware boot variables. [1]

For BitLocker-protected Windows systems, treat firmware boot-order edits like bootloader or EFI-variable work. Microsoft documents boot-order and boot-manager changes as BitLocker-relevant recovery conditions in supported scenarios. [6]

Operational rule:

```text id="uxyler"
Do not repeatedly change firmware boot order.
Do not use efibootmgr casually on a BitLocker system.
Do not assume firmware setup changes are safer than operating-system boot-variable changes.
Suspend BitLocker before planned boot-order edits.
Use one-time boot selection where possible.
```

---

## 16. Hardware clock: use UTC

Windows and Linux often disagree about how the motherboard hardware clock should be interpreted.

Linux normally expects the hardware clock, also called the RTC, to store UTC. Windows traditionally treats the hardware clock as local time. In a dual-boot system, this can make the clock jump forward or backward after switching operating systems.

Use UTC for the hardware clock.

On Linux, verify that the RTC is not in local-time mode:

```bash id="5jusl3"
timedatectl
```

Desired state:

```text id="r7nger"
RTC in local TZ: no
```

If needed, set Linux to use UTC for the RTC:

```bash id="ydd5mb"
sudo timedatectl set-local-rtc 0
```

On Windows, configure Windows to treat the hardware clock as UTC:

```cmd id="cy3y8t"
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" ^
  /v RealTimeIsUniversal ^
  /t REG_DWORD ^
  /d 1 ^
  /f
```

Then reboot Windows.

If the clock still drifts after switching operating systems, resync time in Windows:

```cmd id="higz8u"
w32tm /resync
```

If Windows should rediscover its configured time source first:

```cmd id="eblkdj"
w32tm /resync /rediscover
```

---

## 17. Secure Boot update order

When Secure Boot is enabled on a Windows/Linux dual-boot system, update every boot path before applying revocations or certificate-transition changes.

Recommended order:

```text id="i1l4du"
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
```

The Linux-first step prevents the Linux side from depending on stale Secure Boot components before Windows applies or stages stricter boot trust.

The Windows update step remains mandatory. Microsoft’s Windows Boot Manager revocation guidance says devices with multiple operating systems should update all Windows operating systems before revocations are applied. Microsoft also warns that unupdated Windows versions may be unable to start after revocations. [8]

Do not let fwupd apply Secure Boot DBX updates in parallel with Windows DB, DBX, SVN, or Windows Boot Manager transition work. The fwupd `uefi-dbx` plugin exists to update the UEFI revocation database and notes that DBX updates can block vulnerable EFI binaries. It also notes that firmware update delivery of DBX changes has bricking risk if the bootloader is not updated first. [14]

Operational rule:

```text id="5nwczl"
Update bootloaders first.
Stage revocations second.
Apply one Secure Boot database transition path at a time.
Retest every boot path before proceeding.
```

---

# Windows installation safety

## 18. Protect the Linux ESP before installing Windows

Before installing Windows on a machine that already has Linux, create Linux recovery media and confirm that it boots.

Required recovery item:

```text id="ic6k4j"
current Linux live USB or installer USB
ability to mount the Linux root filesystem
ability to reinstall the Linux bootloader to the Linux ESP
BitLocker recovery key, if Windows already exists on another disk
```

Windows deployment tooling expects a system partition on UEFI/GPT systems. Microsoft describes the EFI System Partition as the system partition used for boot, formatted FAT32, managed by the operating system, and not intended to contain other files. [3] Microsoft’s BCDBoot tooling copies boot files to the EFI System Partition or to the partition explicitly specified with `/s`. It can also create or update Windows Boot Manager firmware entries. [4]

That creates a destructive interaction during Windows installation:

```text id="zx566m"
If Windows Setup or Windows boot servicing sees an existing ESP,
it may use that ESP for Windows boot files instead of creating a new Windows-only ESP.
```

If the visible ESP belongs to Linux, Windows installation or repair can overwrite, replace, or disturb Linux boot files. The safest option is to prevent Windows Setup from seeing the Linux ESP.

Preferred order:

```text id="7fwza1"
1. Disconnect the Linux disk before installing Windows.
2. If the disk cannot be disconnected, disable it in firmware if the board supports that.
3. If it must remain visible, temporarily change the Linux ESP partition type away from EFI System Partition.
4. Restore the Linux ESP partition type after Windows installation.
5. Boot Linux recovery media and reinstall or repair the Linux bootloader if needed.
```

For a GPT disk, changing the partition type is less destructive than formatting the partition. Formatting the Linux ESP removes bootloader files and requires bootloader repair.

Example Linux `sgdisk` pattern:

```bash id="8jkvi2"
# Before Windows installation:
# Change Linux ESP from EFI System Partition to Linux filesystem.
sgdisk --typecode=<N>:8300 /dev/sdX

# After Windows installation:
# Restore Linux ESP type.
sgdisk --typecode=<N>:EF00 /dev/sdX
```

Replace `<N>` with the Linux ESP partition number and `/dev/sdX` with the correct Linux disk.

Do not run these commands on the Windows disk.

If the Linux ESP was formatted, deleted, or overwritten, boot Linux recovery media and reinstall the Linux bootloader to the Linux disk’s own ESP before relying on the machine.

---

## 19. Windows install media from Linux

If a working Windows machine is available, use Microsoft’s Media Creation Tool first. Microsoft documents the Media Creation Tool as the normal path for creating Windows installation media on a USB flash drive. [33]

This section is primarily for Linux-authored Windows installation media, especially when the Windows ISO contains an `install.wim` or `install.esd` that cannot be placed on a single FAT32 partition.

Use a two-partition GPT USB layout when the Windows image exceeds FAT32 limits.

Recommended layout:

```text id="r7s441"
Partition 1: FAT32
  GPT type: Microsoft Basic Data preferred
  fallback GPT type: EFI System Partition if firmware refuses to boot
  size: small; large enough for the Windows installation files except install.wim or install.esd
  contents: Windows installation media copied without \sources\install.wim or \sources\install.esd

Partition 2: exFAT preferred, NTFS acceptable
  GPT type: Microsoft Basic Data
  contents: full Windows installation media, including \sources\install.wim or \sources\install.esd
```

The file system and the GPT partition type are separate requirements.

The boot partition must be firmware-readable. Use FAT32. Start with Microsoft Basic Data for the first partition because it is easier to reuse, format, and manage from Windows after installation. If the target firmware refuses to boot the USB, change the first partition’s GPT type to EFI System Partition and retest.

Do not put the full Windows image file on the FAT32 partition when it exceeds FAT32 limits.

The FAT32 partition should contain the Windows setup boot environment and installer files, but not the main Windows image payload:

```text id="qjrl01"
Copy to Partition 1:
  all files and directories from the Windows installation media
  except:
    \sources\install.wim
    \sources\install.esd
```

Preserve the original directory structure.

The second partition should contain the complete Windows installation media:

```text id="6wtwgz"
Copy to Partition 2:
  all files and directories from the Windows installation media
  including:
    \sources\install.wim
    or
    \sources\install.esd
```

Size the FAT32 partition from the actual source media. Measure the Windows installation media after excluding the main image file:

```text id="9prb0e"
Windows installation media
minus \sources\install.wim
minus \sources\install.esd
plus free space for alignment and small servicing variation
```

Use a small FAT32 partition. Do not allocate a large 32 GB FAT32 partition by default. A practical rule is:

```text id="fkwdxh"
FAT32 partition size =
  size of the installer files excluding install.wim/install.esd
  + 512 MiB to 1 GiB slack
```

Keep the FAT32 partition below 32 GB. If the remaining installer files do not fit comfortably below that limit, the source media is unusual and should be rebuilt or handled with a different deployment procedure.

The image partition must be Windows-readable after WinPE or Windows Setup starts. Use Microsoft Basic Data as the GPT type. Do not leave the image partition marked as Linux filesystem. Windows may refuse to mount, auto-assign, or surface an exFAT partition correctly if the partition type identifies it as Linux data rather than Microsoft Basic Data.

Use exFAT for the second partition when creating the USB from Linux. exFAT supports large files and is usually the lower-friction cross-platform data partition for Linux-authored removable media. Use NTFS only when the Linux environment has known-good NTFS write support or when following Microsoft’s WinPE deployment model. Microsoft’s WinPE USB example uses FAT32 for the boot partition and NTFS for the image partition. [15]

Acceptable variants:

```text id="0jf4oe"
FAT32 Microsoft Basic Data + exFAT Microsoft Basic Data
FAT32 Microsoft Basic Data + NTFS Microsoft Basic Data
FAT32 ESP + exFAT Microsoft Basic Data
FAT32 ESP + NTFS Microsoft Basic Data
FAT32-only USB + split-WIM files, documented by Microsoft but outside this guide
```

Preferred Linux-authored layout:

```text id="p8854u"
FAT32 Microsoft Basic Data + exFAT Microsoft Basic Data
```

Firmware fallback layout:

```text id="vfwvkw"
FAT32 ESP + exFAT Microsoft Basic Data
```

Example Linux `sgdisk` type codes:

```bash id="z5qyao"
# Partition 1: Microsoft Basic Data
sgdisk --typecode=1:0700 /dev/sdX

# Partition 2: Microsoft Basic Data
sgdisk --typecode=2:0700 /dev/sdX
```

If the firmware refuses to boot the USB, change partition 1 to EFI System Partition:

```bash id="fdcu1o"
sgdisk --typecode=1:EF00 /dev/sdX
```

Do not use Linux filesystem type code `8300` for either Windows setup partition.

Example `parted` intent with Microsoft Basic Data first:

```bash id="pbrf1t"
parted /dev/sdX --script mklabel gpt
parted /dev/sdX --script mkpart WINBOOT fat32 1MiB 2049MiB
parted /dev/sdX --script mkpart WINSETUP 2049MiB 100%
```

The `2049MiB` boundary is an example only. Adjust it to the measured size of the Windows installation files excluding `install.wim` or `install.esd`, plus slack.

Then verify both partitions are Microsoft Basic Data. Some Linux tools default new data partitions to Linux filesystem unless told otherwise.

If firmware refuses to boot the FAT32 Microsoft Basic Data partition, set the ESP flag on partition 1 and retest:

```bash id="efeeco"
parted /dev/sdX --script set 1 esp on
```

Microsoft documents split-WIM deployment for FAT32-only media. That method is intentionally out of scope for this guide. Splitting, exporting, rebuilding, or modifying `install.wim` or `install.esd` adds deployment tooling and can create Windows Setup failures if the image is handled incorrectly.

This guide uses the two-partition method so the Windows image remains intact. Test the target firmware and Windows recovery environment before erasing the destination disk. [16]

---

## 20. Fixed-drive USB and ESP deadlock

Some USB devices present themselves as fixed drives rather than removable drives.

If Windows Setup or Windows boot servicing treats the installation media as a fixed drive and the media contains an EFI System Partition, a conservative procedure should assume that Windows may select that visible ESP during installation or repair instead of creating or using the intended ESP on the internal Windows disk.

Risk condition:

```text id="bhu2u6"
Windows install USB presents as fixed disk.
USB partition 1 is marked EFI System Partition.
Windows Setup sees that ESP during install or repair.
Windows boot files may be written to the USB ESP.
Internal Windows disk may be left without the intended independent Windows ESP.
```

Use Microsoft Basic Data for the USB boot partition first. Switch it to ESP only if firmware refuses to boot it as Microsoft Basic Data.

If the drive presents as fixed and the firmware refuses to boot anything except an ESP, use this sequence:

```text id="aeqgxk"
1. Mark USB partition 1 as ESP.
2. Boot the USB.
3. Enter the Windows recovery environment.
4. Open Command Prompt.
5. Delete the USB’s on-drive ESP after WinPE or WinRE has loaded.
6. Launch Windows Setup manually from the second partition.
```

From the recovery Command Prompt, identify the USB disk and delete the USB ESP:

```cmd id="ns3dsg"
diskpart
list disk
select disk X
list partition
select partition Y
delete partition override
exit
```

Replace `X` with the USB disk number and `Y` with the USB ESP partition number.

Do not delete an internal disk ESP.

The `delete partition override` operation is destructive. Use it only after confirming that the selected disk is the installation USB and the selected partition is the USB boot ESP.

---

## 21. Launching Windows Setup from the recovery environment

For this Linux-authored two-partition USB workflow, do not continue through the initial installer path.

The FAT32 partition is used only to boot the Windows setup environment. It intentionally does not contain the main Windows image file:

```text id="t71bt8"
\sources\install.wim
or
\sources\install.esd
```

If the initial installer continues from that partition, Windows Setup may fail because the required installation image is not present there.

Do not proceed until setup is being launched from the second partition, which contains the complete Windows installation media.

This matters because Windows Setup may modify the target disk before the failure becomes visible. It may create or alter the partition table, create setup-related partitions, or place temporary setup files on the target disk. After that state exists, do not continue trying to repair the same failed install attempt.

If the initial installer path was used and setup failed after touching the target disk, restart the machine, boot the Windows setup media again, open Command Prompt, clean the intended Windows target disk with DiskPart, and restart the installation from the correct `setup.exe`.

Boot the USB, then choose:

```text id="58afl3"
Repair your computer
Troubleshoot
Advanced options
Command Prompt
```

Use Command Prompt to identify the second partition, which contains the complete Windows installation media including `install.wim` or `install.esd`.

First option:

```cmd id="x9rmks"
diskpart
list volume
exit
```

Second option:

```cmd id="xme0f6"
notepad
```

In Notepad, use:

```text id="b1c3aw"
File → Open → This PC
```

Look at the visible drive letters and labels. Find the partition that contains the full Windows installation source.

If the source partition is `E:\`, close Notepad and run:

```cmd id="rlwqb5"
E:\setup.exe
```

If the earlier failed setup attempt already modified the target Windows disk, clean only the intended Windows target disk before launching setup again:

```cmd id="jkq381"
diskpart
list disk
select disk X
clean
exit
```

Replace `X` with the intended internal Windows target disk.

Do not clean the Linux disk.

Do not clean the Windows setup USB.

Do not clean any disk unless the disk number has been confirmed by size, model, and installation intent.

Microsoft’s recovery documentation describes the Windows Recovery Environment and its command-line recovery tools. Microsoft’s BCDBoot documentation also uses `diskpart`, `list volume`, and drive-letter identification as part of boot-file repair workflows. [19][4]

---

## 21A. Encrypted hard drives and Block SID

For Windows encrypted hard drives, also called eDrive or eHDD devices, use Windows Setup or a Windows deployment procedure that explicitly supports encrypted hard drive provisioning.

Do not install Windows to an encrypted hard drive through a Linux partitioning workflow and then expect Windows hardware-encryption provisioning to be correct afterward.

Before booting the Windows setup media, check firmware setup for a Block SID option.

The firmware option name varies by vendor. Common labels include:

```text id="a5y7gd"
Disable Block SID
Block SID
SED Block SID Authentication
PPI Bypass for SED Block SID Command
Disable issuing a Block SID Authentication command
```

For this installation path, configure firmware so that the BIOS does not issue a Block SID command that prevents Windows or the deployment environment from provisioning the self-encrypting drive. NVIDIA documents firmware behavior where BIOS sends a Block SID request by default and instructs users to enable a `Disable Block Sid` feature before SED initialization. Dell documents a related firmware setting named `SED Block SID Authentication`. [36][37]

On some systems this means:

```text id="gpfj1b"
Disable Block SID: enabled
```

On other systems it means:

```text id="qksac0"
SED Block SID Authentication: disabled
```

Use the motherboard or system vendor’s wording. The operational target is the same: prevent firmware from blocking authorized encrypted-drive provisioning during Windows installation.

Encrypted hard drive startup-disk requirements are stricter than ordinary storage installation. The target drive must be uninitialized and security-inactive. The system must boot natively through UEFI, CSM must be disabled, and the firmware must support the EFI storage security protocol required for security commands to the drive. [29]

If setup fails or the wrong installer path was used after the target disk was selected, assume the target disk may already have been modified. Reboot, return to the Windows setup recovery command prompt, clean only the intended Windows target disk, and restart the installation from the correct setup source.

Do not use this encrypted-hard-drive path unless the recovery key, firmware settings, disk identity, and target disk selection have been confirmed before installation.

After Windows encrypted-hard-drive provisioning is complete, return the firmware Block SID behavior to its protective state.

The temporary installation state is:

```text id="rx2rx3"
Do not let firmware block the SID command needed for encrypted-drive provisioning.
```

The normal post-installation state is:

```text id="6nu66l"
Firmware should block unintended SID commands again.
```

Vendor wording varies, so verify the meaning of the setting before changing it back.

Examples:

```text id="5jriw1"
If the firmware option is "Disable Block SID":
  enable it only for provisioning
  disable it again after provisioning

If the firmware option is "SED Block SID Authentication":
  disable it only for provisioning
  enable it again after provisioning
```

The operational target is what matters: allow the SID command only for the installation or provisioning window, then restore the firmware setting that prevents unintended SID commands.

Do not leave the provisioning state enabled longer than necessary. Firmware behavior varies. Some systems allow the change for only one boot. Some require physical confirmation on every boot. Some keep the setting persistently. A persistent provisioning state is unnecessary exposure after encryption setup is complete.

On Linux, do not use `cryptsetup --hw-opal-only` as the default OPAL path for this guide.

This is not a claim that hardware-only OPAL configurations are generally broken or insecure. The narrower issue is operational fit. `cryptsetup --hw-opal-only` uses OPAL without a dm-crypt software-encryption layer. Cryptsetup has documented OPAL locking-range geometry failures where a block-size discrepancy could make the configured locking range smaller than intended, leaving remaining space outside the protected range. In a hardware-only OPAL layout, there is no dm-crypt layer above it to preserve confidentiality if that class of range error occurs. [34]

Cryptsetup also documents OPAL support as distinct from ordinary dm-crypt, with `--hw-opal` using OPAL plus dm-crypt and `--hw-opal-only` using OPAL without dm-crypt. It also notes that OPAL resizing is not supported. [30]

If OPAL is required on Linux, use a dedicated OPAL provisioning workflow with sedutil, full-disk SED locking, and pre-boot authentication. The OPAL path should be tested before relying on the machine. Sedutil documents OPAL boot-drive use through pre-boot authentication material and setup flows. [35]

```text id="6jl4px"
confirm the drive is OPAL-capable
test the PBA
enable OPAL locking for the intended full-disk range
load and test the PBA
power off fully
confirm the drive locks on cold boot
confirm the PBA unlocks the drive
confirm the operating system boots only after unlock
document recovery and PSID reset procedure
```

Do not treat cryptsetup OPAL mode as a substitute for a sedutil PBA-based SED deployment. For ordinary Linux confidentiality, use the normal software-encryption path:

```text id="lp5b2p"
LUKS2 / dm-crypt
optional dm-integrity when authenticated storage is required
LVM inside the encrypted container when flexible volume management is required
```

---

## 22. If Windows Setup keeps choosing the wrong ESP

Do not keep repeating the same GUI installation path if Windows Setup keeps selecting an existing Linux ESP or the installation media’s ESP.

Use one of these alternatives:

```text id="mxw73w"
1. Disconnect or hide the Linux disk and retry Windows Setup.
2. Temporarily change the Linux ESP partition type away from EFI System Partition and retry.
3. If the USB is fixed-drive and ESP-only boot is required, delete the USB ESP from WinRE after boot and before launching setup.exe.
4. Use the VM-based Windows installation path.
5. Install Windows manually from WinPE using DiskPart, DISM, and BCDBoot.
```

The manual DiskPart/DISM/BCDBoot path is separate deployment work. Use it only with a procedure that explicitly covers image application, ESP creation, BCD creation, and firmware boot validation.

---

## 23. Reusing a USB after it contains an ESP

After booting Windows or WinPE from a USB that contains an EFI System Partition, normal Windows formatting tools may not offer a clean one-click way to return the whole USB to a simple single data partition.

Use DiskPart from Windows, WinPE, or WinRE:

```cmd id="fk8gvd"
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
```

Replace `X` with the USB disk number.

`clean` destroys the selected disk’s partition table. Microsoft documents that on GPT disks, `clean` overwrites GPT partitioning information, including the protective MBR. Confirm the disk number before running it. [18]

If Windows tooling cannot or should not be used, wipe the USB partition table from Linux and recreate it there. The important operation is removing the old GPT layout, including any ESP, before creating the new data partition.

Microsoft’s `create partition primary` documentation states that if no partition type is specified on GPT, DiskPart creates a basic data partition. [17]

---

# Troubleshooting and recovery

## 24. Unexpected Secure Boot failures

Do not disable Secure Boot as a generic first response to a random boot failure on an established trusted system.

If a previously working system suddenly fails Secure Boot, treat the failure as integrity-relevant until the cause is identified.

Use this sequence:

```text id="6t0op9"
1. Keep Secure Boot enabled.
2. Do not enter BitLocker recovery keys into an unexpected or untrusted boot path.
3. Do not blindly repair BCD.
4. Do not delete ESP files randomly.
5. Boot only known-good, current, Secure-Boot-compatible recovery media.
6. Distinguish update mismatch from boot-chain tampering before changing policy.
```

Temporary Secure Boot disablement is acceptable during planned installation or controlled recovery. It is not the default response to an unexplained boot-chain failure.

---

## 25. Windows is skipped and Linux boots instead

If Windows is first in firmware boot order but the machine boots Linux, do not immediately assume GRUB, BCD, or the Windows ESP is the primary fault.

On Secure Boot systems, firmware may try Windows first, reject the Windows boot path, and continue to the next available boot option. If Linux is next and its boot chain is trusted, Linux may boot normally.

The symptom may be:

```text id="5ggxw0"
Windows is first in firmware boot order.
Linux starts anyway.
No obvious Windows error appears.
```

Possible causes include:

```text id="9yewpp"
Windows Boot Manager is stale relative to current DB, DBX, SVN, or certificate state.
Firmware rejected Windows Boot Manager.
SkuSiPolicy.p7b is missing or mismatched under an existing UEFI lock.
Windows ESP files are damaged.
Firmware boot behavior is unusual.
The boot chain may have been tampered with.
```

First classify the situation:

```text id="da8cdj"
planned installation or known recovery
known Secure Boot servicing transition
unexpected failure on an established trusted system
```

Only the first two cases permit normal controlled-recovery procedures.

---

## 26. Windows Boot Manager revocation recovery

For Secure Boot certificate-transition, DBX, SVN, or Windows Boot Manager revocation recovery, rebuild Windows boot files with `/bootex`.

Use:

```cmd id="qv999z"
bcdboot C:\Windows /f UEFI /s S: /bootex
```

This is the required form for this recovery class.

Do not rebuild the Windows ESP for a revocation-transitioned system with ordinary BCDBoot syntax. Without `/bootex`, BCDBoot can service the ESP with the non-bootex Windows Boot Manager path, which can restore an older or wrong boot path for the system’s current Secure Boot state.

Microsoft documents `/bootex` as the BCDBoot option that uses bootex binaries for servicing when the necessary conditions are met. Microsoft links this option to the Windows Boot Manager revocations associated with CVE-2023-24932. [4][8]

Use `/bootex` for:

```text id="dps20w"
Windows Boot Manager revocation recovery
Windows UEFI CA 2023 transition work
DB/DBX/SVN Secure Boot transition recovery
CVE-2023-24932 boot media servicing
revocation-aware Windows ESP reconstruction
```

Do not use `/bootex` as generic BCD repair for unrelated boot failures.

---

## 27. `/offline /bootex` recovery

If Secure Boot has been disabled to recover a machine whose old Windows certificate or boot manager is no longer trusted by the current Secure Boot policy, use the offline bootex path when the installed Windows build and recovery source support it:

```cmd id="m91l46"
bcdboot C:\Windows /f UEFI /s S: /offline /bootex
```

Microsoft documents `/offline` as forcing boot-file servicing to be handled offline. Microsoft also states that boot-file selection is forced based on the presence of the `bootex` switch. [4]

Use `/offline /bootex` when all of the following are true:

```text id="tjmkqd"
The failure is in the Secure Boot revocation or certificate-transition class.
Secure Boot was disabled to allow recovery work.
The Windows installation or recovery source supports /offline.
The source contains the required bootex binaries.
The Windows partition and Windows ESP have been identified correctly.
```

Microsoft documents `/offline` support beginning with Windows 11 version 24H2 build 26100.8037 and Windows 11 version 25H2 build 26100.8037. [4]

Do not use an old Windows ISO, stale recovery drive, or unpatched WinPE source for this recovery class. If the source does not contain the required bootex files, the recovery source is wrong for the current Secure Boot state.

---

## 28. SkuSiPolicy.p7b recovery condition

`SkuSiPolicy.p7b` is a Microsoft-signed revocation policy used for VBS rollback protection. It is not generic BCD repair.

A specific recovery condition exists:

```text id="qefcio"
A prior Windows installation applied the SkuSiPolicy.p7b UEFI lock.
The Windows installation was erased, restored, replaced, or rebuilt.
The firmware still carries the policy lock.
The Windows ESP no longer contains the matching policy file.
Windows Boot Manager fails without a visible boot error.
Firmware proceeds to the next available boot option.
```

Microsoft documents that applying `SkuSiPolicy.p7b` locks the policy to the device by adding a UEFI firmware variable. Microsoft also states that if the UEFI lock is applied and the policy is removed or replaced with an older version, Windows Boot Manager will not start. [9]

Microsoft further documents that reformatting the disk does not remove the UEFI lock, and that reverting to an earlier Windows state can cause the device not to start, show no error message, and proceed to the next available boot option. [9]

In that condition, restore the current policy file from the installed Windows image to the Windows ESP.

Source:

```text id="rrr3hy"
C:\Windows\System32\SecureBootUpdates\SkuSiPolicy.p7b
```

Destination:

```text id="0yuhnq"
<Windows ESP>\EFI\Microsoft\Boot\SkuSiPolicy.p7b
```

PowerShell pattern from a trusted, fully updated Windows environment:

```powershell id="gmsgvp"
$PolicyBinary = $env:windir+"\System32\SecureBootUpdates\SkuSiPolicy.p7b"
$MountPoint = 's:'
$EFIDestinationFolder = "$MountPoint\EFI\Microsoft\Boot"

mountvol $MountPoint /S

if (-Not (Test-Path $EFIDestinationFolder)) {
    New-Item -Path $EFIDestinationFolder -Type Directory -Force
}

Copy-Item -Path $PolicyBinary -Destination $EFIDestinationFolder -Force

mountvol $MountPoint /D
```

Use the policy file from the installed Windows image being booted.

Do not copy an old policy file from stale install media, another machine, or an older Windows build.

This recovery condition applies to Windows VBS rollback protection. It does not repair the Linux Secure Boot chain, Linux shim, GRUB, systemd-boot, or Linux kernel trust state.

Do not use `mokutil --set-ssp-policy` for this recovery path. This section is limited to restoring the Windows policy file expected by Windows Boot Manager on the Windows ESP. Shim-mediated SkuSiPolicy variable handling is separate and should not be used as a substitute for rebuilding or restoring the Windows boot path.

---

# Additional installation options

## 29. Installing Windows in a VM onto a physical disk

This is an advanced installation path. Use it when normal installation is impractical or when Windows Setup repeatedly selects the wrong ESP.

If Windows is installed in a VM and then moved to real hardware, generalize it before the first bare-metal boot.

At the Windows OOBE screen, press:

```text id="xab836"
Ctrl + Shift + F3
```

Windows will reboot into Audit Mode. Microsoft documents this keyboard shortcut for booting to Audit Mode. [21]

The System Preparation Tool opens automatically.

Select:

```text id="xjgotk"
System Cleanup Action:
  Enter System Out-of-Box Experience (OOBE)

Generalize:
  checked

Shutdown Options:
  Shutdown
```

Then shut down the VM and boot the disk on the real machine.

Microsoft documents that Sysprep `/generalize` removes unique information from a Windows installation so the image can be reused on another computer. [20]

Be prepared to use Microsoft’s boot repair procedures if the real firmware does not discover the disk automatically.

---

## 30. Installing Linux in a VM onto a physical disk

If Linux is installed in a VM and then moved to bare metal, make sure the Linux bootloader is installed to the Linux disk’s own ESP.

Use the fallback path:

```text id="hmj1in"
\EFI\BOOT\BOOTX64.EFI
```

Use a fallback or all-drivers initramfs for the first bare-metal boot when the distribution supports that mode.

After Linux successfully boots on the real machine, regenerate the initramfs normally.

A VM install may generate an initramfs optimized for virtual storage and chipset devices. A broader first-boot initramfs reduces the chance that real-hardware storage drivers are missing during the first bare-metal boot.

Follow the selected distribution’s documented recovery and initramfs tooling. The exact commands vary across distributions.

---

# Daily operation

## 31. Daily use policy

Default boot should be Windows.

To boot Linux, use one of:

```text id="npgh0j"
Windows Recovery → Use a device
firmware one-time boot menu
firmware disk selector
```

Do not make GRUB chainload Windows.

Do not repeatedly edit EFI variables unless necessary.

Do not repeatedly change firmware boot order unless BitLocker has been suspended for a planned boot-maintenance window.

Do not share ESPs unless both systems can be recovered manually.

Keep these recovery items available:

```text id="oyijp2"
BitLocker recovery key
current Windows install or recovery USB
current Linux live USB
knowledge of controlled Secure Boot disablement for recovery
firmware boot-menu key for the machine
```

---

## 32. Testing checklist

After installation, verify all of this before relying on the machine:

```text id="7we0a2"
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
```

---

## 33. Operational summary

Choose the lower-coupling boot design.

```text id="a6i6mj"
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
```

This layout minimizes avoidable dependencies between Windows security state, Linux boot tooling, firmware behavior, EFI variables, Secure Boot policy, and recovery paths.

---

# References

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
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe--use-a-single-usb-key-for-winpe-and-a-wim-file---wim

[16] Microsoft Learn, Split a Windows image file.  
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/split-a-windows-image--wim--file-to-span-across-multiple-dvds

[17] Microsoft Learn, create partition primary.  
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/create-partition-primary

[18] Microsoft Learn, DiskPart clean.  
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/clean

[19] Microsoft Learn, Windows Recovery Environment technical reference.  
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-recovery-environment--windows-re--technical-reference

[20] Microsoft Learn, Sysprep generalize a Windows installation.  
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/sysprep--generalize--a-windows-installation

[21] Microsoft Learn, Boot Windows to Audit Mode or OOBE.  
https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/boot-windows-to-audit-mode-or-oobe

[22] Microsoft Learn, format command.  
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/format

[23] Microsoft Learn, create partition EFI.  
https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/create-partition-efi

[24] systemd timedatectl manual.  
https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html

[25] Microsoft Learn, w32tm.  
https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings

[26] Linux kernel documentation, dm-crypt.  
https://docs.kernel.org/admin-guide/device-mapper/dm-crypt.html

[27] Linux kernel documentation, dm-integrity.  
https://docs.kernel.org/admin-guide/device-mapper/dm-integrity.html

[28] systemd manual, systemd-cryptenroll.  
https://man.archlinux.org/man/systemd-cryptenroll.1.en

[29] Microsoft Learn, Encrypted hard drives.  
https://learn.microsoft.com/en-us/windows/security/operating-system-security/data-protection/encrypted-hard-drive

[30] cryptsetup manual, SED OPAL extension.  
https://manpages.debian.org/trixie/cryptsetup-bin/cryptsetup.8.en.html

[31] Red Hat Documentation, Managing kernel command-line parameters with UKI.  
https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_monitoring_and_updating_the_kernel/managing-kernel-command-line-parameters-with-uki

[32] Debian Wiki, UKI.  
https://wiki.debian.org/UKI

[33] Microsoft Support, Create installation media for Windows.  
https://support.microsoft.com/windows/create-installation-media-for-windows-99a58364-8c02-206f-aa6f-40c3b507420d

[34] cryptsetup 2.7.3 release notes.  
https://kernel.googlesource.com/pub/scm/utils/cryptsetup/cryptsetup/+/refs/heads/v2.8.x/docs/v2.7.3-ReleaseNotes

[35] sedutil project documentation.  
https://sedutil.com/

[36] NVIDIA DGX Station A100 User Guide, Managing Self-Encrypting Drives.  
https://docs.nvidia.com/dgx/dgx-station-a100-user-guide/manage-seds.html

[37] Dell Support, Pre-Boot Authentication Will Not Activate Due to SED Block SID Authentication Enabled.  
https://www.dell.com/support/kbdoc/en-in/000126083/pre-boot-authentication-will-not-activate-due-to-sed-block-sid-authentication-enabled
