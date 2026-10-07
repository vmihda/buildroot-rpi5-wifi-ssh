# buildroot-rpi5-wifi-ssh

A Buildroot external tree that builds a small Linux image for the Raspberry Pi 5. The board joins your Wi-Fi on boot and you can log in over SSH with a key. You don't need a monitor or Ethernet.

The image is about 150 MB and the root filesystem uses roughly 34 MB. It contains the Raspberry Pi 6.6 kernel, BusyBox, Dropbear, wpa_supplicant and the Broadcom Wi-Fi firmware.

## What's inside

```
configs/rpi5_ssh_defconfig         Buildroot config, based on raspberrypi5_defconfig
board/rpi5/post-build.sh           injects secrets, fixes file permissions
board/rpi5/rootfs_overlay/         files copied on top of the root filesystem
  etc/network/interfaces           brings up wlan0: wpa_supplicant, DHCP, power save off
  etc/modprobe.d/brcmfmac.conf     driver workaround, see "Known issues"
  etc/sysctl.d/                    keeps driver errors off the HDMI console
  etc/init.d/S99netdiag            writes a Wi-Fi report to the boot partition
  root/.ssh/authorized_keys        public key allowed to log in as root
secrets.env.example                variables you need to set before building
external.desc, Config.in, external.mk   BR2_EXTERNAL boilerplate
```

## Requirements

You need a Linux build host with the [usual Buildroot dependencies](https://buildroot.org/downloads/manual/manual.html#requirement-mandatory). I built it on a MacBook (Apple Silicon) inside an `ubuntu:24.04` container. Keep the Buildroot tree and the output directory in a Docker volume, not in a folder shared from macOS: the macOS filesystem is case-insensitive and the kernel sources will not build on it.

Use the Buildroot 2025.02.x LTS branch. Newer releases build the Pi 5 config with a Bootlin toolchain that only runs on x86_64 hosts. The 2025.02 config builds its own toolchain, so it also works on arm64.

Build as a regular user, not root.

## Build

The commands below assume the repo is cloned to `~/buildroot-rpi5-wifi-ssh` and you start in your home directory.

```sh
# Buildroot itself
wget https://buildroot.org/downloads/buildroot-2025.02.18.tar.xz
tar xf buildroot-2025.02.18.tar.xz

# Secrets live outside the repository
cp buildroot-rpi5-wifi-ssh/secrets.env.example ~/rpi5-secrets.env
chmod 600 ~/rpi5-secrets.env
$EDITOR ~/rpi5-secrets.env

# Configure (out-of-tree build) and build
cd buildroot-2025.02.18
make O=$HOME/rpi5-output BR2_EXTERNAL=$HOME/buildroot-rpi5-wifi-ssh rpi5_ssh_defconfig
cd $HOME/rpi5-output
set -a; . ~/rpi5-secrets.env; set +a
make
```

The first build took about 45 minutes on an M2 Pro, mostly spent on the toolchain and the kernel. The result is `images/sdcard.img`.

Put your own public key in `board/rpi5/rootfs_overlay/root/.ssh/authorized_keys` before building. The one in the repo is mine.

## Secrets

The repository contains no passwords and no hashes. `post-build.sh` reads them from the environment of the `make` call:

| Variable | Meaning |
|---|---|
| `ROOT_PASSWORD_HASH` | crypt hash for root, `$6$...` (SHA-512). Generate it with `mkpasswd -m sha-512` or `openssl passwd -6` |
| `WIFI_SSID` | network name |
| `WIFI_PASSWORD` | Wi-Fi passphrase. The script derives the WPA2 PSK from it |
| `WIFI_PSK` | a precomputed 64-hex-digit PSK, instead of `WIFI_PASSWORD` |
| `WIFI_COUNTRY` | optional regulatory domain, `UA` by default |

Keep the values in single quotes in the env file, because the hash contains `$`.

If a variable is missing, the build fails with a message such as `ROOT_PASSWORD_HASH: is not set`. Without that check you could end up with an image where root has no password.

The secrets still end up inside `sdcard.img`, since the board needs them. That is why `*.img` is in `.gitignore`.

## Flash and connect

On macOS:

```sh
diskutil list external
diskutil unmountDisk /dev/diskN
sudo dd if=sdcard.img of=/dev/rdiskN bs=4m status=progress
diskutil eject /dev/diskN
```

On Linux, use `dd` with `of=/dev/sdX`. Double-check the device name: `dd` overwrites the whole disk.

Boot the Pi and give it a minute. The board does not send a hostname over DHCP, so look for its MAC in your router's client list (Pi 5 boards usually start with `2c:cf:67` or `d8:3a:dd`), or scan for an open port 22 (macOS `nc`; on Linux replace `-G 1` with `-w 1`):

```sh
for i in $(seq 1 254); do (nc -z -G 1 192.168.1.$i 22 2>/dev/null && echo "192.168.1.$i") & done; wait
```

Then:

```sh
ssh -i ~/.ssh/your_key root@<board-ip>
```

Every reflash generates a new Dropbear host key, so SSH will warn that the host identification changed. Remove the old entry with `ssh-keygen -R <board-ip>`.

## If the board does not show up

You don't need a monitor for this. `S99netdiag` runs for five minutes after boot and writes `netdiag.txt` to the FAT boot partition, rewriting it every minute. Power the board off, put the card into any computer and open that file. It shows the `wpa_supplicant` state over time, the connected frequency, the networks seen in the last scan, the `wpa_supplicant` messages from syslog and the Wi-Fi lines from `dmesg`.

With a monitor and keyboard, log in as root on HDMI0 (the port next to USB-C power).

## Known issues and workarounds

### Kernel panic at boot on newer boards

Pi 5 boards with the BCM2712 D0 stepping (they report `Rev 1.1`) panic with `Asynchronous SError Interrupt` in `bcm2712_pull_config_set`. The firmware has to apply `overlays/bcm2712d0.dtbo`, and the stock `raspberrypi5_defconfig` in 2025.02 does not install overlays. This config enables `BR2_PACKAGE_RPI_FIRMWARE_INSTALL_DTB_OVERLAYS`.

### Wi-Fi drops or never authenticates

Since 2.11, wpa_supplicant waits for a "port authorized" event after a firmware-offloaded handshake, and `brcmfmac` does not send it, so the link times out after 10 seconds. Buildroot used to carry a revert patch; it was dropped with the 2.12 security bump, which is also in 2025.02.x. `options brcmfmac feature_disable=0x82000` turns off the firmware handshake offload (FWSUP and SAE) and leaves the handshake to wpa_supplicant. Encryption is unaffected. With only `0x2000` the connection was still unstable for me.

### 300 ms ping and lost packets

`brcmfmac` enables Wi-Fi power save by default. The interface config turns it off with `iw dev wlan0 set power_save off`.

### `set chanspec 0x... fail, reason -52` in dmesg

The firmware's regulatory table does not include `UA`, so the driver logs `Firmware rejected country setting`. After each scan wpa_supplicant asks for per-channel noise statistics, and the firmware refuses channels 12, 13 and 34 to 46. It doesn't affect the connection. The sysctl file keeps these messages off the console.

### Logs

`/var/log` is a tmpfs, so nothing is written to the SD card. wpa_supplicant logs to syslog (`-s`), and BusyBox syslogd rotates `/var/log/messages` at 200 KB with one old copy, so log size stays bounded however long the board runs.

## References

- [Buildroot manual](https://buildroot.org/downloads/manual/manual.html)
- [Buildroot commit e7b3286c79](https://github.com/buildroot/buildroot/commit/e7b3286c79), the wpa_supplicant/brcmfmac problem and the `feature_disable=0x82000` workaround
- [Raspberry Pi forums: Buildroot Pi 5 kernel fails to boot](https://forums.raspberrypi.com/viewtopic.php?t=378197), the D0 overlay
- [boompi PR #10](https://github.com/TooTallNate/boompi/pull/10), the same wpa_supplicant 2.12 issue in another Buildroot project
