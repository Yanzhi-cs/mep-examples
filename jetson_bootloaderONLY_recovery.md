# Jetson Orin NX Bootloader Recovery Guide
## Target: JetPack 6.0 / L4T R36.3.0 on Ubuntu 24.04 (Noble) host

---

## 0. Prerequisites
- USB-C to USB-A cable
- [Host PC] with Ubuntu 24.04 (Noble) OS
- [MEP] with Jetson Orin NX (The OS SSD does *NOT* need to be plugged in at all)

---

## 1. Physical Setup
1. Plug USB-C into MEP Jetson (while powered off) -> USB-A into any x86_64 Ubuntu host. Ubuntu 24.04 (Noble) is known to work.
2. Put Jetson into recovery mode, by shorting out the leads on the 'recovery' 2-pin port, then power on the Jetson

On the host, confirm Jetson is visible in recovery mode:
```bash
lsusb | grep -i nvidia
# Should show: NVIDIA Corp. APX
```
---

## 2. (On the Host PC) Install dependencies

```bash
sudo apt update
sudo apt install -y qemu-user-static binfmt-support nfs-kernel-server sshpass abootimg
```

### 2.1 Ubuntu 26.04 compatibility

On Ubuntu 26.04, install the renamed QEMU packages and `xmllint` dependency:

```bash
sudo apt install -y qemu-user qemu-user-binfmt binfmt-support libxml2-utils \
  nfs-kernel-server sshpass abootimg
```

If `/usr/bin/qemu-aarch64-static` is missing, provide the name expected by the NVIDIA scripts:

```bash
if [ ! -e /usr/bin/qemu-aarch64-static ]; then
  sudo ln -s /usr/bin/qemu-aarch64 /usr/bin/qemu-aarch64-static
fi
```

---

## 3. (On the Host PC) Increase USB buffer size

```bash
sudo sh -c 'echo 1000 > /sys/module/usbcore/parameters/usbfs_memory_mb'
```

---

## 4. (On the Host PC) Fix USB network interface naming

Ubuntu 24.04 renames the Jetson's USB network interface away from `usb0`. This rule forces it to stay as `usb0` for NVIDIA devices only:

```bash
echo 'SUBSYSTEM=="net", ACTION=="add", ATTRS{idVendor}=="0955", NAME="usb0"' | sudo tee /etc/udev/rules.d/72-nvidia-usb.rules
sudo udevadm control --reload-rules
```

---
## 5. (On the Host PC) If UFW is active, allow inbound initrd-flash traffic on usb0

This rule allows inbound IPv6 traffic on `usb0` from the Jetson initrd flashing subnet. It does not open all `usb0` traffic.

```bash
sudo ufw allow in on usb0 from fc00:1:1::/48 \
  comment 'NVIDIA Jetson initrd flashing'
```

---

## 6. (On the Host PC) Download and extract the BSP and root filesystem

```bash
cd ~/Downloads

curl -L -o Jetson_Linux_R36.3.0_aarch64.tbz2 \
  https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/release/Jetson_Linux_R36.3.0_aarch64.tbz2

tar xf Jetson_Linux_R36.3.0_aarch64.tbz2

curl -L -o Tegra_Linux_Sample-Root-Filesystem_R36.3.0_aarch64.tbz2 \
  https://developer.nvidia.com/downloads/embedded/l4t/r36_release_v3.0/release/Tegra_Linux_Sample-Root-Filesystem_R36.3.0_aarch64.tbz2

sudo tar xpf Tegra_Linux_Sample-Root-Filesystem_R36.3.0_aarch64.tbz2 -C Linux_for_Tegra/rootfs/
```

Note: the root filesystem tarball is ~1.7GB. It is needed to build the ramdisk even though the OS itself will not be flashed.

---

## 7. (On the Host PC) Apply binaries

Remove conflicting device files first to prevent the script from failing silently:

```bash
cd ~/Downloads/Linux_for_Tegra
sudo rm -f rootfs/dev/random rootfs/dev/urandom rootfs/dev/null rootfs/dev/zero
sudo ./apply_binaries.sh
```

The last line of output should say `L4T BSP package installation completed!`

### 7.1 Ubuntu 26.04 compatibility

OpenSSH 10 no longer generates DSA keys. Replace the obsolete key type used by the NVIDIA recovery-image script:

```bash
cd ~/Downloads/Linux_for_Tegra
sudo sed -i 's/-t dsa/-t rsa -b 2048/' \
  tools/ota_tools/version_upgrade/ota_make_recovery_img_dtb.sh
```

---

## 8. (From the Host PC) Flash Jetson bootloader only using l4t_initrd_flash

Make sure Jetson is in recovery mode and plugged in, then:

```bash
cd ~/Downloads/Linux_for_Tegra
sudo ./tools/kernel_flash/l4t_initrd_flash.sh \
  -p "-c bootloader/generic/cfg/flash_t234_qspi.xml" \
  --showlogs \
  --network usb0 \
  jetson-orin-nano-devkit internal
```

The script will:
1. Build the flash images
2. Boot the Jetson into a minimal initrd environment over USB
3. SSH into it and flash the QSPI bootloader partitions
4. Reboot the Jetson

You will see repeated `Waiting for target to boot-up...` and `Waiting for device to expose ssh...` messages for several minutes — this is normal. Do not interrupt it.

Once SSH connects, it will flash eMMC then QSPI. It may appear stuck at `Starting to flash to qspi` for several minutes — this is also normal. Do not interrupt it.

When complete you will see:
```text
Flash is successful
Reboot device
```

### 8.1 Ubuntu 26.04: Fix false `NFS server is not running` errors

If `rpc.mountd` reports `Unable to watch /proc/fs/nfsd/clients: No space left on device`, increase the inotify limits and restart NFS:

```bash
sudo sysctl -w fs.inotify.max_user_watches=1048576
sudo sysctl -w fs.inotify.max_user_instances=8192

printf '%s\n' \
  'fs.inotify.max_user_watches=1048576' \
  'fs.inotify.max_user_instances=8192' | \
  sudo tee /etc/sysctl.d/99-inotify.conf

sudo systemctl restart nfs-kernel-server
sudo systemctl start nfs-mountd.service
```

Confirm that `mountd` and NFS are available before rerunning the flash command:

```bash
sudo systemctl status nfs-mountd.service --no-pager -l
showmount -e localhost
sudo rpcinfo -p localhost | grep -E 'mountd|nfs'
```

`showmount` should return `Export list for localhost:` instead of `Connection refused`.

---

## 9. Optional: (On the Host PC) Remove the temporary UFW rule

If you added the `usb0` allow rule for flashing and do not want to keep it, list numbered rules and delete the matching entry:

```bash
sudo ufw status numbered
sudo ufw delete <rule-number>
```

---

## Notes
- The board config `jetson-orin-nano-devkit` is correct for both Orin NX and Orin Nano
- The Jetson does **not** need to boot for this to work — recovery mode bypasses the bootloader entirely
- `l4t_initrd_flash.sh` is the official NVIDIA method for Orin NX — `flash.sh` does not work reliably for this board
- The BSP downloads may require an NVIDIA developer account — if curl returns an HTML page instead of the tarball, log in at developer.nvidia.com and download manually
