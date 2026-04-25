# Netboot Bazzite — Full Setup Guide

## Overview

- **Pi 5** (wired) — proxy DHCP + TFTP, serves iPXE and kernel
- **TrueNAS SCALE** — NFS export of read-only Bazzite golden image
- **PC** — 32GB RAM, boots Bazzite over NFS with tmpfs overlay, reimages every boot

---

## Part 1 — TrueNAS: Create the datasets

SSH into TrueNAS or use the shell in the UI.

```bash
# Dataset for the golden root filesystem
zfs create Lambo/Bugatti/netboot
zfs create Lambo/Bugatti/netboot/bazzite-root

# Dataset for TFTP files (kernel, initramfs, iPXE)
zfs create Lambo/Bugatti/netboot/tftp
```

### Enable NFS on TrueNAS

1. Go to **Shares → Unix Shares (NFS) → Add**
2. Path: `/mnt/Lambo/Bugatti/netboot/bazzite-root`
3. Under **Advanced**:
   - Networks: your LAN subnet e.g. `192.168.1.0/24`
   - Maproot User: `root`
   - Maproot Group: `wheel`
   - Check **No subtree checking**
4. Make sure the NFS service is started: **System → Services → NFS → On**

---

## Part 2 — Pi 5: Install and configure dnsmasq

Wire the Pi into your downstairs switch first.

```bash
sudo apt update
sudo apt install -y dnsmasq ipxe
```

### Copy iPXE bootloaders

```bash
sudo mkdir -p /srv/tftp
sudo cp /usr/lib/ipxe/ipxe.efi /srv/tftp/          # UEFI
sudo cp /usr/lib/ipxe/undionly.kpxe /srv/tftp/      # Legacy BIOS
```

### Create the iPXE boot script

Replace `PI_IP` with your Pi's IP and `NAS_IP` with your TrueNAS IP.

```bash
sudo nano /srv/tftp/boot.ipxe
```

```
#!ipxe
kernel http://PI_IP/bazzite/vmlinuz root=/dev/nfs nfsroot=NAS_IP:/mnt/Lambo/Bugatti/netboot/bazzite-root,vers=4,ro ip=dhcp rd.live.overlay.overlayfs=1 rd.live.overlay=ram rw
initrd http://PI_IP/bazzite/initramfs.img
boot
```

### Configure dnsmasq (proxy mode — router untouched)

```bash
sudo nano /etc/dnsmasq.conf
```

Paste this, replacing `192.168.1.0` with your actual subnet and `PI_IP` with your Pi's IP:

```
# Proxy DHCP — only intercepts PXE clients, ignores everything else
dhcp-range=192.168.1.0,proxy

enable-tftp
tftp-root=/srv/tftp

# UEFI
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-boot=tag:efi-x86_64,ipxe.efi

# Legacy BIOS
dhcp-boot=undionly.kpxe

# Once iPXE is running, hand it the boot script
dhcp-userclass=set:ipxe,iPXE
dhcp-boot=tag:ipxe,http://PI_IP/boot.ipxe

log-dhcp
```

### Serve kernel files over HTTP

```bash
sudo apt install -y nginx
sudo mkdir -p /var/www/html/bazzite
```

We'll copy the kernel and initramfs here in Part 4.

### Start services

```bash
sudo systemctl enable --now dnsmasq
sudo systemctl enable --now nginx
sudo systemctl restart dnsmasq nginx
```

---

## Part 3 — Get Bazzite

Download the Bazzite ISO on any machine (your PC or the NAS via shell).

Go to https://bazzite.gg and grab the ISO for your GPU:
- AMD GPU → `Bazzite-stable-x86_64.iso`
- Nvidia GPU → `Bazzite-stable-nvidia-x86_64.iso`

### Extract the root filesystem onto TrueNAS

On TrueNAS shell:

```bash
# Install required tools
apt install -y squashfs-tools  # or use the TrueNAS app shell

# Mount the ISO
mkdir -p /tmp/iso /tmp/squash
mount -o loop /path/to/bazzite.iso /tmp/iso

# Extract squashfs root
unsquashfs -d /mnt/Lambo/Bugatti/netboot/bazzite-root /tmp/iso/LiveOS/squashfs.img

# Clean up
umount /tmp/iso
```

> If TrueNAS doesn't have `unsquashfs`, do this step on the Pi or any Linux machine and copy the result to the NAS dataset over SSH/rsync.

### Copy kernel and initramfs to Pi

From TrueNAS shell (replace `PI_IP`):

```bash
scp /mnt/Lambo/Bugatti/netboot/bazzite-root/boot/vmlinuz* pi@PI_IP:/var/www/html/bazzite/vmlinuz
scp /mnt/Lambo/Bugatti/netboot/bazzite-root/boot/initramfs* pi@PI_IP:/var/www/html/bazzite/initramfs.img
```

---

## Part 4 — Rebuild initramfs with NFS support

The stock Bazzite initramfs may not include NFS modules. Rebuild it on the Pi or a Linux machine:

```bash
# On the Pi (needs dracut)
sudo apt install -y dracut

sudo dracut \
  --add "nfs network base" \
  --kver $(ls /mnt/Lambo/Bugatti/netboot/bazzite-root/lib/modules/) \
  --force \
  /var/www/html/bazzite/initramfs.img
```

> The `--kver` value should match the kernel version in the bazzite-root — check with `ls /mnt/.../bazzite-root/lib/modules/`

---

## Part 5 — PC BIOS settings

1. Enter BIOS (usually Del or F2 on boot)
2. **Enable** Network Stack / PXE Boot
3. **Disable** Secure Boot
4. Set **boot order**: Network first, then Windows drive
5. Save and exit

> When you want to boot Windows just skip the network boot with F12 (or whatever your one-time boot menu key is) and pick the SSD.

---

## Part 6 — First boot: Create your golden image

Boot the PC — it should PXE boot into a live Bazzite session. Now set everything up:

```bash
# Set your password
passwd

# Install Discord
flatpak install flathub com.discordapp.Discord

# Install Sober (Roblox)
flatpak install flathub org.vinegarhq.Sober

# Any other config — themes, settings, etc.
```

Once you're happy with everything, **do not reboot yet**. Go to TrueNAS and snapshot the dataset:

```bash
# On TrueNAS shell
zfs snapshot Lambo/Bugatti/netboot/bazzite-root@golden
```

This is now your golden image. Every boot will start from this state.

---

## Part 7 — Auto-reimage on every boot

On TrueNAS, create a script that rolls back to the golden snapshot before each boot:

```bash
nano /mnt/Lambo/Bugatti/netboot/reimage.sh
```

```bash
#!/bin/bash
zfs rollback -r Lambo/Bugatti/netboot/bazzite-root@golden
echo "Reimaged to golden snapshot"
```

```bash
chmod +x /mnt/Lambo/Bugatti/netboot/reimage.sh
```

### Run it automatically before the PC boots

Add a cron job on TrueNAS (or a systemd service) that runs the rollback whenever the PC requests a PXE boot. The simplest approach is to run it on a schedule — e.g. every time the NFS share is remounted:

On TrueNAS, go to **System → Advanced → Cron Jobs → Add**:
- Command: `/mnt/Lambo/Bugatti/netboot/reimage.sh`
- Schedule: `@reboot` (runs when TrueNAS starts)

For true on-demand reimaging (rollback every time the PC boots), you can hook into the dnsmasq lease log on the Pi and trigger it via SSH — but the simple cron approach works fine for a single machine.

---

## Part 8 — tmpfs overlay (RAM disk for session writes)

Add this to your boot kernel cmdline in `/srv/tftp/boot.ipxe` — already included in Part 2 but explained here:

```
rd.live.overlay.overlayfs=1   # use overlayfs (modern, efficient)
rd.live.overlay=ram            # writes go to RAM, not NFS
```

This means:
- NFS root is **read-only** — your golden image is never touched
- All writes during your session go to RAM (tmpfs)
- On reboot everything in RAM is gone, NFS rolls back to golden

With 32GB RAM you have ~26GB of overlay space after Bazzite's base usage. More than enough for Sober, Discord, and gaming sessions.

---

## Troubleshooting

**PC doesn't get a PXE boot prompt**
- Check dnsmasq is running: `sudo systemctl status dnsmasq`
- Check logs: `sudo journalctl -u dnsmasq -f`
- Make sure both switches are uplinked and on the same VLAN

**Boots to iPXE shell but kernel fails to load**
- Check nginx is serving files: `curl http://PI_IP/bazzite/vmlinuz`
- Check the filename matches exactly in boot.ipxe

**Mounts NFS but kernel panics**
- initramfs doesn't have NFS modules — redo Part 4
- Double check NAS_IP and NFS path in boot.ipxe

**Boots but very slow**
- Expected on first boot (shader compilation)
- Subsequent boots from golden snapshot will be faster as tmpfs caches warm up

---

## Adding more machines later

For each new machine:
1. Create a new dataset: `zfs create Lambo/Bugatti/netboot/machine2-root`
2. Copy or clone the golden image: `zfs clone Lambo/Bugatti/netboot/bazzite-root@golden Lambo/Bugatti/netboot/machine2-root`
3. Add NFS export for the new dataset
4. Add a new entry in boot.ipxe matching that machine's MAC address to its own NFS root

Each machine gets its own isolated root, all served from the NAS.
