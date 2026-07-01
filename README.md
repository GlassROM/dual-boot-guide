# Conservative Dual Boot Guide

A conservative Windows + Linux dual-boot guide focused on separate disks, separate EFI System Partitions, independent boot paths, Secure Boot, BitLocker, TPM-related recovery risks, and bootloader isolation.

The goal is to reduce avoidable cross-OS boot coupling. Windows should boot through Windows Boot Manager. Linux should boot through its own bootloader. Firmware should choose between them.

## Scope

This guide is intended for technically capable users who understand that firmware, EFI partitions, Secure Boot databases, BitLocker, and bootloader changes can make a system temporarily or permanently unbootable if handled incorrectly.

It is not a general beginner dual-boot tutorial.

## Notice and disclaimer

This material is provided for informational and educational purposes only.

No warranty is provided. Use this material at your own risk. The author is not liable for data loss, boot failure, recovery-key lockout, firmware misconfiguration, security weakening, downtime, hardware issues, or any other damage or loss arising from use of this material.

This is not legal, security, compliance, enterprise deployment, or incident-response advice. For production, regulated, enterprise, or attested environments, follow the applicable vendor documentation and internal change-control process.

Product names, project names, company names, logos, and trademarks mentioned in this project belong to their respective owners.

This project is independent. It is not affiliated with, sponsored by, approved by, or endorsed by any operating-system vendor, Linux distribution, bootloader project, firmware vendor, hardware vendor, standards body, or software publisher mentioned.

## License

Conservative Dual Boot Guide © 2026 by Aryan Duggal is licensed under Creative Commons Attribution 4.0 International. To view a copy of this license, visit https://creativecommons.org/licenses/by/4.0/
