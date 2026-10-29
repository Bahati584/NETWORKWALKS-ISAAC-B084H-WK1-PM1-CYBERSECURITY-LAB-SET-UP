# Cybersecurity Lab Setup: VirtualBox + Kali Linux (WK1-PM1)

**Student:** Isaac
**Task:** WK1-PM1, Week 1 Project Module 1: Lab Setup (VirtualBox and Kali Linux)
**Program:** Networkwalks Cybersecurity Program

## Objective

Build an isolated cybersecurity testing lab with VirtualBox as the base and Kali Linux as the attacker machine, on a custom NAT Network.

| Requirement | Result |
|---|---|
| VirtualBox as base | Done |
| Kali Linux as attacking machine | Done |
| NAT Network on `10.0.0.0/24` | Done |
| Clipboard and drag-and-drop enabled | Done (Bidirectional) |
| Shared folder (`Downloads` from host) | Done |
| Kali IP address `10.0.0.2/24` | Done |
| Full internet access from Kali | Done |
| Snapshot of the VM | Done |

## Environment

- **Host OS:** Arch Linux (ArchCraft). The course slides use Windows, so some host-side steps differ.
- **Hypervisor:** Oracle VirtualBox 7.x
- **Guest:** Kali Linux 2026.3 (installed from the installer ISO)
- **VM resources:** 4 GB RAM, 2 CPUs, ~30 GB virtual disk

## Steps

### 1. Install VirtualBox and load the kernel driver

```bash
sudo pacman -S virtualbox virtualbox-host-modules-arch
sudo modprobe vboxdrv
lsmod | grep vboxdrv          # confirm the driver is loaded
sudo gpasswd -a $USER vboxusers   # then log out and back in
```

![VirtualBox driver loaded](screenshots/01-vboxdrv.png)

### 2. Create the NAT Network

```bash
VBoxManage natnetwork add --netname NatNetwork --network "10.0.0.0/24" --enable --dhcp on
VBoxManage list natnetworks
```

![NAT Network](screenshots/02-natnetwork.png)

### 3. Create the Kali VM

I used the Kali installer ISO. Settings used in the New VM wizard:

- Type: Linux, Version: Debian (64-bit)
- 4096 MB RAM, 2 CPUs, ~30 GB disk
- Unattended install: not available for this ISO, so I ran the graphical installer manually
- Guided partitioning (whole virtual disk), default Xfce desktop and default tools, GRUB on `/dev/sda`

![New VM summary](screenshots/03-vm-summary.png)

### 4. Configure the VM settings

- **Network > Adapter 1:** Attached to `NAT Network`, Name `NatNetwork`, Promiscuous Mode `Allow All`
- **General > Features:** Shared Clipboard `Bidirectional`, Drag-and-Drop `Bidirectional`
- **Shared Folders:** host `Downloads` folder added with Auto-mount (global), mounted in Kali at `/media/sf_Downloads`

![Network settings](screenshots/04-network-settings.png)
![General settings](screenshots/05-general-settings.png)
![Shared folders](screenshots/06-shared-folders.png)

### 5. Set the static IP in Kali

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method manual \
  ipv4.addresses 10.0.0.2/24 ipv4.gateway 10.0.0.1 ipv4.dns 8.8.8.8
sudo nmcli connection modify "Wired connection 1" ipv4.dad-timeout 0
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

### 6. Verify networking

```bash
ip a                    # eth0 shows 10.0.0.2/24
ping -c 3 10.0.0.1      # gateway
ping -c 3 8.8.8.8       # internet by IP
ping -c 3 google.com    # DNS
```

All three pings returned 0% packet loss.

![IP address](screenshots/07-ip-a.png)
![Ping tests](screenshots/08-pings.png)

### 7. Verify the shared folder and clipboard

```bash
sudo usermod -aG vboxsf $USER   # then reboot the VM
ls /media/sf_Downloads
```

The host's Downloads files are visible inside Kali, and copy/paste works between host and VM.

![Shared folder listing](screenshots/09-shared-folder.png)

### 8. Take a snapshot

Snapshot name: **Fresh Kali - 10.0.0.2 configured**. This is a clean baseline to roll back to after any risky experiment.

![Snapshot list](screenshots/10-snapshot.png)

## Troubleshooting notes

| Problem | Fix |
|---|---|
| "NAT Network" missing from the adapter dropdown | The NAT Network must exist first. Create it with `VBoxManage natnetwork add` (step 2), then reopen Settings. |
| Unattended install checkbox greyed out | The Kali installer ISO doesn't support it. Run the graphical installer manually. |
| No internet in Kali on VirtualBox 7 | Set `ipv4.dad-timeout 0` and restart the connection (step 5). Also check that no other VM is using `10.0.0.2`. |
| `/media/sf_Downloads` not accessible | Add your user to the `vboxsf` group and reboot. If Guest Additions are missing, install `virtualbox-guest-x11` and `virtualbox-guest-utils`. |
| "Kernel driver not installed" on Arch | Install `virtualbox-host-modules-arch` (or `virtualbox-host-dkms` plus kernel headers) and run `sudo modprobe vboxdrv`. |

## What I learned

- How a VirtualBox NAT Network differs from plain NAT: VMs share one subnet and can reach each other and the internet.
- Setting a static IP with `nmcli` and verifying connectivity layer by layer (gateway, internet, DNS).
- Why snapshots matter in a security lab: a known-good baseline to restore.
- Setting up VirtualBox on an Arch-based host, including the kernel driver and group permissions.

## Next steps

Phase 2 (optional): add Windows and Android VMs on the same NAT Network and ping-test between them.

---

*Part of the Networkwalks Cybersecurity Program.*
