# Quick start - Prerequisites
- UEFI boot only (systemd-boot)
- SSD storage (NVMe drives strongly recommended)
- At least 16gb RAM
# ZFS Encryption setup guide
Currently, the Proxmox installer does not support setting up encryption with ZFS. Thus, we have to set it up manually. This guide will go over how to use the native ZFS encryption with Proxmox.

The guide also assumes that the Proxmox installation is new and does not have any virtual machines or containers yet.
<img width="2532" height="1396" alt="image" src="https://github.com/user-attachments/assets/6cd346a5-6463-49cb-b68c-1a0d711936bb" />
# Encrypting the rpool/ROOT dataset
Proxmox installs its system inside the rpool/ROOT dataset. This is what we will encrypt first.

First, boot into the initramfs. On the startup menu, press e to edit the boot argument. Remove root=ZFS=rpool/ROOT/pve-1 boot=zfs from the argument and press enter.

<img width="2050" height="1396" alt="image" src="https://github.com/user-attachments/assets/921aea83-de80-4945-83a2-96e71be60be6" />

Load in the zfs kernel module:
```
modprobe zfs
```
Next, follow this gist to encrypt the dataset. You do not need to use any sort of live USB or rescue mode, as the initramfs has all that we need. In case it gets moved or deleted, I will copy and paste it here (we will make a few changes to better suit our purposes as well):
```
# Import the old
zpool import -f rpool

# Make a snapshot of the current one
zfs snapshot -r rpool/ROOT@copy

# Send the snapshot to a temporary root
zfs send -R rpool/ROOT@copy | zfs receive rpool/copyroot

# Destroy the old unencrypted root
zfs destroy -r rpool/ROOT

# Set better ZFS properties
zpool set autoexpand=on rpool
zpool set autotrim=on rpool
zpool set failmode=wait rpool

# Create a new zfs root, with encryption turned on
# OR -o encryption=aes-256-gcm - aes-256-ccm vs aes-256-gcm
zfs create -o acltype=posix -o atime=off -o compression=zstd-3 -o checksum=blake3 -o dnodesize=auto -o encryption=on -o keyformat=passphrase -o overlay=off -o xattr=sa rpool/ROOT

# Copy the files from the copy to the new encrypted zfs root
zfs send -R rpool/copyroot/pve-1@copy | zfs receive -o encryption=on rpool/ROOT/pve-1

# Deviate from the original gist and delete copyroot
zfs destroy -r rpool/copyroot

# Set the Mountpoint
zfs set mountpoint=/ rpool/ROOT/pve-1

# Export the pool again, so you can boot from it
zpool export rpool
```
Reboot into the system. You should now be prompted for an encryption password.
```
reboot -f
```
# Encrypting the rpool/data dataset
Next, we need to encrypt the rpool/data dataset. This is where Proxmox stores virtual machine disks.
```
# Destroy the original dataset
zfs destroy -r rpool/data
```
Create a diceware passphrase, and save it to /.data.key. Then, continue with:
```# Remove all but ASCII characters 
perl -i -pe 's/[^ -~]//g' /.data.key

# Set the appropriate permission
chmod 400 /.data.key

# Make the key immutable
chattr +i /.data.key

# Create a new dataset with encryption enabled
zfs create -o acltype=posix -o atime=off -o compression=zstd-3 -o checksum=blake3 -o dnodesize=auto -o encryption=on -o keyformat=passphrase -o keylocation=file:///.data.key -o overlay=off -o xattr=sa rpool/data
```
Next, we need to set up a systemd service for automatic unlocking. Put the following inside `/etc/systemd/system/zfs-load-key.service`:
```
[Unit]
Description=Load encryption keys
DefaultDependencies=no
After=zfs-import.target
Before=zfs-mount.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/zfs load-key -a

[Install]
WantedBy=zfs-mount.service
```
Finally, enable the service:
```
systemctl enable zfs-load-key
```
# Run the script
1.
```
chmod +x Proxmox-9.1.sh
```
2.
```
./Promox.9.1.sh
```
The script will:

- Disable debug shell and mask unwanted services.
- Apply apt upgrade behavior to avoid phased updates.
- Configure NTP (chrony) and restart chronyd.
- Harden SSH (custom sshd/ssh configs and systemd override).
- Switch APT sources to Debian trixie + Proxmox non-subscription repo.
- Update packages and install microcode, proxmox-kernel, unattended-upgrades, systemd-zram-generator, tuned.
- Append a hardened kernel cmdline (expects systemd-boot) and refresh the bootloader.
- Blacklist framebuffer and other modules; apply sysctl hardening.
- Rebuild initramfs.
- Disable core dumps.
- Configure systemd ZRAM generator.
- Apply UI patch to suppress nags and restart pveproxy.
- Deploy unattended-upgrades config, enable tuned profile, and enable fstrim.timer.
