# Proxmox Homelab – Working Notes

## Environment
- Host: HP EliteDesk 800 G3 Mini (i5, 16 GB RAM, 256 GB SSD), shipped with preinstalled Windows
- Client: ThinkPad (Windows, web UI access)
- Proxmox VE: 9.2 (Debian 13 based)
- Local repo path: C:\Users\Krobe\Projects\homelab\proxmox-homelab

## Log
| Date | Phase | Step | Notes |
|------|-------|------|-------|
| 2026-09-24 | 0 | Created GitHub repo | Public, README + MIT license, no .gitignore template |
| 2026-09-24 | 0 | Cloned repo | Created shared parent folder Projects\homelab for all upcoming homelab repos |
| 2026-09-24 | 0 | Created folder structure | docs/, configs/, screenshots/ with .gitkeep placeholders |
| 2026-09-24 | 0 | Added .gitignore | Excludes OS files, secrets (keys, env, passwords) and large binaries (ISO/IMG) |

## Decisions
| Decision | Reason |
|----------|--------|
| Parent folder Projects\homelab | Keeps all future homelab repos (VMs, AD, DNS, ...) in one place |
| .gitkeep in empty folders | Git does not track empty directories |
| .gitignore saved as ASCII | Windows PowerShell 5.1 adds a BOM with -Encoding utf8, which can break config files |
| Secrets excluded via .gitignore | Public repo – no passwords, tokens or keys may ever be committed |

## Problems & Solutions
| Problem | Cause | Solution |
|---------|-------|----------|
| PowerShell opened in C:\WINDOWS\system32 | Session was started as Administrator | Closed it and used a normal user session (principle of least privilege) |

## Commands Used
```powershell
New-Item -ItemType Directory -Path "$HOME\Projects\homelab" -Force
git clone https://github.com/rkraus-tech/proxmox-homelab.git
New-Item -ItemType Directory -Path docs, configs, screenshots
New-Item -ItemType File -Path screenshots\.gitkeep, configs\.gitkeep
```

## Additional Problems & Solutions
| Problem | Cause | Solution |
|---------|-------|----------|
| Inline code backticks missing in notes.md | In a double-quoted here-string (`@"..."@`) the backtick is PowerShell's escape character | Use single-quoted here-strings (`@'...'@`) for Markdown content – no escaping, no variable expansion |
| Git warning "LF will be replaced by CRLF" | Git for Windows default `core.autocrlf=true` | Added `.gitattributes` with `* text=auto eol=lf` so Linux configs keep LF endings regardless of client settings |

## Network Plan
| Item | Value |
|------|-------|
| Router | Telekom Speedport Smart 3 (192.168.2.1) |
| Subnet | 192.168.2.0/24 |
| Router DHCP range | 192.168.2.100 – 192.168.2.230 |
| Proxmox host | pve01.home.arpa – 192.168.2.10/24 (static, outside DHCP range) |
| Reserved: infrastructure | 192.168.2.11 – 192.168.2.19 |
| Reserved: homelab VMs | 192.168.2.20 – 192.168.2.49 |

| Problem | Cause | Solution |
|---------|-------|----------|
| DHCP settings not visible in Speedport UI | Hidden in "Standardmodus" | Settings → Ansicht → switch to "Expertenmodus", then Netzwerk → Heimnetzwerk (LAN) → DHCP |
| `Test-Connection` threw cryptic "PingException" | Windows PowerShell 5.1 reports an unanswered ping as an exception | Verified with classic `ping` instead |
| `ping 192.168.2.10` answered by own client (.180) with "Destination host unreachable" | Same subnet → client sends ARP request, nobody answers | Confirms the IP is free – chosen as static Proxmox IP |

## Bootable USB
| Item | Value |
|------|-------|
| Tool | Rufus 4.15 Portable |
| Stick | 512 GB USB stick (fully wiped) |
| Write mode | DD image mode (forced automatically by Rufus) |

| Problem | Cause | Solution |
|---------|-------|----------|
| Partition scheme / target system greyed out in Rufus | Proxmox ISO is ISOHybrid, not compatible with ISO copy mode – Rufus forces DD mode and uses the partition table embedded in the image | No action needed – the image already supports UEFI and legacy boot |

## BIOS Configuration (HP Computer Setup, F10)
| Setting | Value | Reason |
|---------|-------|--------|
| Virtualization Technology (VTx) | Enabled | Required for KVM virtual machines |
| Virtualization Technology for Directed I/O (VTd) | Enabled | Enables PCI passthrough to VMs later |
| USB Storage Boot | Enabled | Boot the Proxmox installer from USB |
| Boot mode | UEFI only (Legacy disabled) | Modern boot path, required for Secure Boot |
| Secure Boot | Enabled, MS UEFI CA key enabled | Proxmox boots via Microsoft-signed shim |
| After Power Loss | Power On | Server comes back automatically after a power outage |
| Startup Delay | 5 sec | Fast Boot with 0 sec made F9/F10 almost impossible to hit |

| Problem | Cause | Solution |
|---------|-------|----------|
| Hard to enter BIOS / boot menu | Fast Boot with 0 sec startup delay | Set Startup Delay to 5 sec |
| Booted into preinstalled Windows setup instead of USB | Windows Boot Manager is first in the UEFI boot order, F9 was pressed too late | Powered off, tapped F9 repeatedly right after power-on, selected "USB Disk 3.0" in the boot menu |

## Decision: Filesystem
| Option | Verdict | Reason |
|--------|---------|--------|
| ext4 + LVM-Thin | ✅ Chosen | Single SSD, low RAM overhead, Proxmox default, LVM-Thin still supports VM snapshots |
| ZFS (RAID0, single disk) | ❌ Rejected | No redundancy/self-healing with one disk, ARC cache consumes RAM needed for VMs, higher SSD wear |

## Installation (Proxmox VE 9.2)
| Setting | Value |
|---------|-------|
| Target disk | /dev/nvme0n1 – Samsung 256 GB NVMe (238.47 GiB) |
| Filesystem | ext4, default partitioning (root / swap / LVM-Thin data) |
| Locale | Germany, Europe/Berlin, keymap de |
| Management interface | nic0 (Intel I219-LM, e1000e), interface name pinned |
| Hostname | pve01.home.arpa |
| IP / Gateway / DNS | 192.168.2.10/24 / 192.168.2.1 / 192.168.2.1 |
| Result | First boot OK – web UI announced at https://192.168.2.10:8006 |

| Problem | Cause | Solution |
|---------|-------|----------|
| Risk: network interface names can change after kernel/firmware updates | Linux predictable interface naming depends on hardware/driver details | Kept "Pin network interface names" enabled → NIC stays `nic0` |
