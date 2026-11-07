# Step 1: Ubuntu Server installation

Part of the [Linux Administration](../../tree/main) project. This branch covers setting up Ubuntu Server 26.04.1 LTS on a VirtualBox virtual machine, checking it, reaching it from Windows, and protecting it with a snapshot and a clone.

## Files

| File | Description |
|---|---|
| `Linux_administration.pdf` | Full step-by-step report with screenshots |

## What was done

1. Checked the host resources and that hardware virtualization is enabled.
2. Created the VM in VirtualBox.
3. Installed Ubuntu Server from the ISO.
4. Verified the system and the network.
5. Reached the server from Windows over SSH (Bridged Adapter, or port forwarding in NAT).
6. Took a snapshot of the clean install.
7. Cloned the VM to prepare for the next labs.

## VM configuration

| Setting | Value |
|---|---|
| VM name | Ubuntu Server |
| OS | Ubuntu Server 26.04.1 LTS, 64-bit |
| RAM | 2 GB |
| vCPU | 2 |
| Disk | 60 GB, VDI, dynamically allocated |
| Network | NAT |

## Installer choices

| Option | Choice |
|---|---|
| Installer language | English |
| Keyboard | French (AZERTY) |
| Network | DHCP (NAT) |
| Proxy | None |
| Mirror | Default |
| Storage | Entire 60 GB virtual disk |
| Server name | `ubuntu-server` |
| Ubuntu Pro | Skipped |
| OpenSSH server | Installed |
| Extra snaps | None |

## Commands used

Checking the system:

```bash
hostname
cat /etc/os-release | grep VERSION
free -h
lscpu | grep "CPU(s):"
lsblk
df -h
```

Checking the network and SSH:

```bash
ip addr
ip route
ping -c 4 8.8.8.8
ping -c 4 google.com
systemctl status ssh
```

Reaching the server from Windows:

```bash
# Bridged Adapter (use the IP shown by "ip a" in the VM)
ssh <user>@<vm-ip>

# NAT with a port forwarding rule (host port 2222 -> guest port 22)
ssh <user>@127.0.0.1 -p 2222
```

Snapshot and VirtualBox command line (the snapshot is taken after a clean shutdown):

```bash
sudo systemctl poweroff
```

```powershell
VBoxManage list vms
VBoxManage showvminfo "Ubuntu Server"
```

## Snapshot and clone

- Snapshot `Clean-Install`: a restore point of the fresh installation. It is not a backup, because it depends on the original disk file.
- Clone `Ubuntu Client #1`: a full clone with new MAC addresses, reused in the next labs.

## Reference

A. Ahmadi, TP 1: Mise en place de l'environnement de virtualisation, Cloud Computing et Virtualisation, ENSAM Meknès.

## License

Copyright (C) 2026 Zaayd

This project is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

See the `LICENSE` file for the full text.
