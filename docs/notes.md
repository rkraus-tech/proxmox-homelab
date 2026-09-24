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
