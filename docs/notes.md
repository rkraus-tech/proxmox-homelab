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

## Decision: Filesystem
| Option | Verdict | Reason |
|--------|---------|--------|
| ext4 + LVM-Thin | ✅ Chosen | Single SSD, low RAM overhead, Proxmox default, LVM-Thin still supports VM snapshots |
| ZFS (RAID0, single disk) | ❌ Rejected | No redundancy/self-healing with one disk, ARC cache consumes RAM needed for VMs, higher SSD wear |

## First Login & Repositories
| Item | Value |
|------|-------|
| Web UI | https://192.168.2.10:8006 (self-signed certificate warning accepted) |
| Login | root, realm Linux PAM |
| Detected | Proxmox VE 9.2.2, 4x i5-6500, 15.5 GiB RAM, EFI Secure Boot |
| Repositories | Disabled pve-enterprise + ceph enterprise, added pve-no-subscription (deb822 `.sources` format) |

| Problem | Cause | Solution |
|---------|-------|----------|
| "Update package database" task failed (`apt-get update` exit code 100) | Only the enterprise repositories are enabled after install; they require a paid subscription key (HTTP 401) | Disabled both enterprise repos (PVE + Ceph), added the free `pve-no-subscription` repo → refresh returns TASK OK |
| Browser certificate warning on port 8006 | Proxmox uses a self-signed certificate by default | Accepted for the homelab; replacing it with a trusted certificate is a possible later improvement |

## System Update
| Item | Value |
|------|-------|
| Method | Updates → Refresh → Upgrade (`apt dist-upgrade` in web shell) |
| Result | System up to date, pve-manager 9.2.20 |
| Kernel | 7.0.2-6-pve (ISO) → 7.0.14-19-pve, reboot required to activate |

| Problem | Cause | Solution |
|---------|-------|----------|
| Several warnings during upgrade (ceph chown, watchdog-mux, "skipping ESP sync") | Packages for unused features (Ceph, HA cluster, ZFS boot sync) | Reviewed and classified as harmless for a single ext4 node |
| Git command accidentally pasted into the Proxmox web shell | Wrong window focused | Discarded with Ctrl+C; lesson: always check the prompt (`PS C:\>` vs `root@pve01:~#`) before pasting |
| Upgrade popup console (Upgrade button) unusable after abort | The upgrade console session ended; the window only showed the old prompt | Closed it and used the node shell (pve01 → Shell) instead |

## Second Update Round (after 16 days uptime)
| Item | Value |
|------|-------|
| Pending updates | 21 packages (OpenSSL security fixes, pve-manager 9.2.21, new kernel 7.0.14-23) |
| Method | `apt full-upgrade` in node shell, cached packages reused (0 B download) |
| Kernels installed | 7.0.14-23 (default), 7.0.14-19, 7.0.2-6 (fallbacks in GRUB) |
| Lesson | Check for new updates before a planned reboot → one reboot instead of two |

## Additional Problems & Solutions (Update Round 2)
| Problem | Cause | Solution |
|---------|-------|----------|
| New kernel not active after upgrade (still 7.0.2-6-pve after 15 days uptime) | Reboot after the kernel update was never performed – a new kernel only loads on boot | Ran pending updates again, then rebooted once to activate the latest kernel |
| Upgrade aborted after download (`^C` in console) | Ctrl+C sends SIGINT and cancels apt before packages are unpacked | Ran `dpkg --configure -a` (consistency check), then `apt full-upgrade` again – downloaded packages were reused from the apt cache |

## Network Bridge (vmbr0)
| Item | Value |
|------|-------|
| Physical NIC | nic0 (alt. name enp0s31f6), no IP, uplink port of the bridge |
| Bridge | vmbr0 – Linux bridge, 192.168.2.10/24, gateway 192.168.2.1, autostart |
| STP / forward delay | off / 0 – single uplink, no loop possible |
| VLAN aware | No – Speedport router has no VLAN support (possible future improvement with a managed switch) |
| Config backup | `configs/network-interfaces` (copy of `/etc/network/interfaces`) |

## Storage
| Storage | Type | Content | Location | Purpose |
|---------|------|---------|----------|---------|
| local | Directory | Backup, Import, ISO image, Container template | /var/lib/vz (root partition, ~72 GB) | Files: ISOs, templates, backups |
| local-lvm | LVM-Thin | Disk image, Container | Thin pool on the same SSD | VM disks as block devices – thin provisioning + snapshots |

| Item | Value |
|------|-------|
| Decision | Kept the default layout – fits a single-disk node, no changes needed |
| Shared | No – only relevant for network storage in a cluster |
| Test | Downloaded debian-13.7.0-amd64-netinst.iso via "Download from URL" with SHA-256 verification → "checksum verified", TASK OK |
| ISO path | /var/lib/vz/template/iso/ |
