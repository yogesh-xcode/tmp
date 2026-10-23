# PHASE 1 — Instructions: Bare Arch, hardened, hardware-supported

**Machine:** ASUS TUF Gaming F15 **FX506HF**
**Goal:** one encrypted, self-snapshotting, cool-running Arch install. No desktop, no rice, no AUR.
**Rule:** one gate at a time. Do not skip a check. If a check fails, stop — do not continue "to see if it works".

---

## 0. Already decided (do not re-litigate)

| Decision | Value |
|---|---|
| Distro | Arch, manual install (`pacstrap` + `arch-chroot`), **no `archinstall`** |
| Disk | **Wipe everything** — Windows + old Debian. Backed up. |
| Layout | 1 GiB ESP + LUKS2 (argon2id) → btrfs with `@ @home @snapshots @var_log @var_cache` |
| `/boot` | unencrypted on the ESP (GRUB cannot read LUKS2/argon2id) |
| Swap | zram only, **no hibernation** |
| Bootloader | GRUB + `grub-btrfs` (snapshot entries in the menu) |
| Kernels | `linux` + `linux-lts` |
| GPU | Intel iGPU drives the session; RTX 2050 = offload only (`prime-run`) |
| Packages | official repos only — `pacman -Qm` must stay empty |
| Hostname | `tuf` |
| User | `yo`, in `wheel` |

**Disk to destroy:** `/dev/nvme0n1` — 476.9 GB — serial `S6F5NX0W273529`
**Do not touch:** `/dev/sda` (the install USB)

---

## 1. Materials

- Arch USB, verified: two partitions on it, `ARCH_202610` (iso9660) + `ARCHISO_EFI` (vfat)
- Ethernet cable (strongly preferred for the install)
- Your LUKS passphrase written on paper, **two copies**
- Backup of everything you care about (done)

---

## 2. Boot the medium

1. Reboot → tap **F2** → **F7** (Advanced Mode)
2. **Security → Secure Boot → Secure Boot Control → Disabled**
3. **Boot → Fast Boot → Disabled** (CSM/Legacy off if present)
4. **F10** → save & exit
5. Power on → tap **F8** → pick the entry beginning **`UEFI:`**
6. Arch menu → **"Arch Linux install medium (x86_64, UEFI)"**

You land at `root@archiso ~ #`.

### Gate 1
```bash
cat /sys/firmware/efi/fw_platform_size       # must print: 64
timedatectl                                   # "System clock synchronized: yes"
ip a                                          # note interface name
ping -c3 archlinux.org                        # replies
```
If no network and you're on wifi:
```bash
iwctl
 device list
 station wlan0 scan
 station wlan0 get-networks
 station wlan0 connect "YOUR_SSID"
 exit
ping -c3 archlinux.org
```
Then:
```bash
pacman -Sy --needed archlinux-keyring
```

---

## 3. Confirm the disk (mandatory)

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL
```
Expected:
```
nvme0n1   476.9G  SAMSUNG MZVLQ512HBLU-00B00  S6F5NX0W273529
```
**If the serial differs, stop.** Report what you see.

---

## 4. Partition (destroys the disk)

```bash
sgdisk --zap-all /dev/nvme0n1
sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"ESP"       /dev/nvme0n1
sgdisk -n 2:0:0   -t 2:8309 -c 2:"CRYPTARCH" /dev/nvme0n1
partprobe /dev/nvme0n1
lsblk /dev/nvme0n1
```
Expected: `nvme0n1p1` 1G, `nvme0n1p2` ~475G.

---

## 5. Encrypt

```bash
mkfs.fat -F32 -n ESP /dev/nvme0n1p1

cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 \
  --key-size 512 --pbkdf argon2id --label CRYPTARCH /dev/nvme0n1p2
```
- Type `YES` in capitals.
- Type the passphrase twice. **No echo. No undo. No recovery.**
- Caps Lock off. Then prove it:

```bash
cryptsetup open /dev/nvme0n1p2 cryptroot
cryptsetup close cryptroot
cryptsetup open /dev/nvme0n1p2 cryptroot     # must succeed again
```
If the second open fails → reformat now (`luksFormat` again); nothing is lost yet.

### Gate 2
`ls /dev/mapper/cryptroot` exists.

---

## 6. btrfs and subvolumes

```bash
mkfs.btrfs -L ARCH /dev/mapper/cryptroot

OPTS="noatime,compress=zstd:3,ssd,discard=async"

mount -o $OPTS,subvolid=5 /dev/mapper/cryptroot /mnt
for s in @ @home @snapshots @var_log @var_cache; do btrfs subvolume create /mnt/$s; done
umount /mnt

mount -o $OPTS,subvol=@ /dev/mapper/cryptroot /mnt
mkdir -p /mnt/{home,.snapshots,var/log,var/cache,boot}

mount -o $OPTS,subvol=@home      /dev/mapper/cryptroot /mnt/home
mount -o $OPTS,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount -o $OPTS,subvol=@var_log   /dev/mapper/cryptroot /mnt/var/log
mount -o $OPTS,subvol=@var_cache /dev/mapper/cryptroot /mnt/var/cache
mount /dev/nvme0n1p1 /mnt/boot

findmnt -R /mnt
```

### Gate 3
Six btrfs mounts + `vfat` on `/mnt/boot`. No mount is missing. `@var_cache` is **not** inside `@`.

---

## 7. Install the base

```bash
pacstrap -K /mnt base base-devel linux linux-headers linux-lts linux-lts-headers \
  linux-firmware intel-ucode btrfs-progs cryptsetup grub efibootmgr \
  networkmanager sudo git vim nano man-db bash-completion \
  snapper snap-pac grub-btrfs pacman-contrib reflector alsa-utils

genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```
Check: every entry is `UUID=…`, the five btrfs lines carry `subvol=`, and all use `noatime,compress=zstd:3,ssd,discard=async`.

```bash
arch-chroot /mnt
```

---

## 8. Configure the system

```bash
ln -sf /usr/share/zoneinfo/<Region/City> /etc/localtime
hwclock --systohc

sed -i 's/^#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us"        > /etc/vconsole.conf
echo "tuf"              > /etc/hostname

passwd                                     # root password
useradd -m -G wheel -s /bin/bash yo
passwd yo

echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/10-wheel
chmod 440 /etc/sudoers.d/10-wheel

systemctl enable NetworkManager
```

### 8.1 pacman and journald
```bash
sed -i 's/^#Color/Color/; s/^#ParallelDownloads = 5/ParallelDownloads = 5/' /etc/pacman.conf
grep -q ILoveCandy /etc/pacman.conf || sed -i '/^Color/a ILoveCandy' /etc/pacman.conf
grep -q VerbosePkgLists /etc/pacman.conf || sed -i '/^Color/a VerbosePkgLists' /etc/pacman.conf

mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\nSystemMaxUse=500M\n' \
  > /etc/systemd/journald.conf.d/persistent.conf
```

### 8.2 Initramfs
Edit `/etc/mkinitcpio.conf` so these two lines are exactly:
```
MODULES=(i915)
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt filesystems fsck)
```
```bash
mkinitcpio -P
```

### 8.3 Kernel command line
```bash
blkid -s UUID -o value /dev/nvme0n1p2
```
Copy that UUID. Edit `/etc/default/grub`:
```
GRUB_TIMEOUT=3
GRUB_DISABLE_SUBMENU=y
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
GRUB_CMDLINE_LINUX="rd.luks.name=<UUID>=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ rw"
```
`GRUB_CMDLINE_LINUX` must be one line, UUID pasted exactly.

### 8.4 Bootloader
```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH
grub-mkconfig -o /boot/grub/grub.cfg
grep -c "Arch Linux" /boot/grub/grub.cfg     # should be > 0
```

### 8.5 Reboot
```bash
exit
umount -R /mnt
cryptsetup close cryptroot
reboot
```

### Gate 4
- GRUB menu appears with **Arch Linux** and **Arch Linux (LTS)**
- You're asked for the LUKS passphrase
- You log in as `yo`

**If the menu doesn't appear or the passphrase loops, stop and go to §13.**

---

## 9. Turn on the safety net

```bash
sudo snapper -c root create-config /
sudo snapper -c root set-config "TIMELINE_CREATE=no" "NUMBER_LIMIT=10" "NUMBER_LIMIT_IMPORTANT=5"
sudo systemctl enable --now grub-btrfsd paccache.timer reflector.timer systemd-timesyncd
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo snapper -c root create -d "clean-base"
sudo snapper -c root list
```

If `create-config` refuses because `/.snapshots` exists: it already is our subvolume — create `/etc/snapper/configs/root` manually with `SUBVOLUME="/"`, `ALLOW_GROUPS="wheel"`, `TIMELINE_CREATE="no"`, `NUMBER_LIMIT="10"`, `NUMBER_LIMIT_IMPORTANT="5"`, then `snapper -c root list`.

### Gate 5
- `snapper -c root list` shows `clean-base`
- reboot → GRUB now lists **snapshots** as extra entries
- `pacman -Qm` prints nothing

---

## 10. Hardware support

```bash
sudo pacman -S mesa mesa-utils vulkan-intel intel-media-driver libva-utils \
  nvidia-open-dkms nvidia-utils libva-nvidia-driver \
  pipewire pipewire-audio wireplumber pavucontrol sof-firmware alsa-ucm-conf \
  tlp thermald zram-generator lm_sensors asusctl fwupd smartmontools nvme-cli \
  bluez bluez-utils powertop
```

### 10.1 ASUS control, fans, battery
```bash
sudo systemctl enable --now asusd
asusctl profile -P Balanced
asusctl battery limit 80          # if the subcommand differs: asusctl battery --help
asusctl profile -p                # confirm
```
With `power-profiles-daemon` **not** installed. `tlp`:
```bash
sudo systemctl enable --now tlp
sudo tlp-stat -b | head -30
```

### 10.2 Memory (8 GB — this matters)
```bash
sudo tee /etc/systemd/zram-generator.conf >/dev/null <<'EOF'
[zram0]
zram-size = min(ram / 2, 8192)
compression-algorithm = zstd
EOF
echo 'vm.swappiness=100' | sudo tee /etc/sysctl.d/99-zram.conf
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service
zramctl
```

### 10.3 Wifi without drops
```bash
sudo tee /etc/NetworkManager/conf.d/wifi-powersave-off.conf >/dev/null <<'EOF'
[connection]
wifi.powersave = 2
EOF
sudo systemctl restart NetworkManager
```

### 10.4 Storage health
```bash
sudo systemctl enable --now btrfs-scrub@-.timer btrfs-scrub@home.timer smartd
sudo sensors-detect --auto
sudo nvme smart-log /dev/nvme0n1 | head -20
```

### 10.5 GPU offload wrapper
```bash
sudo tee /usr/local/bin/prime-run >/dev/null <<'EOF'
#!/bin/bash
export __NV_PRIME_RENDER_OFFLOAD=1
export __GLX_VENDOR_LIBRARY_NAME=nvidia
export __VK_LAYER_NV_optimus=NVIDIA_only
exec "$@"
EOF
sudo chmod +x /usr/local/bin/prime-run
prime-run glxinfo -B | head -20
nvidia-smi
vainfo | head -15
```

### 10.6 Firmware
```bash
sudo fwupdmgr refresh --force
sudo fwupdmgr get-devices
```

### Gate 6 — hardware
| Check | Command | Pass |
|---|---|---|
| iGPU session works | `glxinfo -B` | Intel renderer |
| dGPU offload | `prime-run glxinfo -B` | NVIDIA renderer |
| driver loaded | `nvidia-smi` | card listed |
| video decode | `vainfo` | Intel profile present |
| audio out/in | `wpctl status`, `pavucontrol` | speakers + mic |
| zram | `zramctl` | device, ~4G |
| fans/battery | `asusctl profile -p`, `asusctl battery` | Balanced, limit 80 |
| power | `tlp-stat -b` | battery section populated |
| suspend | `systemctl suspend`, wake | resumes clean |
| errors | `dmesg --level=err,crit`, `journalctl -b -p 3` | no recurring i915/mt7921/ACPI errors |

---

## 11. Baseline (record these — Phase 2 needs them)

```bash
sensors                                              # idle temps
sudo powertop --time=60                              # power
upower -d | grep -E "percentage|energy-rate"         # battery
systemd-analyze                                      # boot time
sudo nvme smart-log /dev/nvme0n1 | grep -E "temperature|percentage_used"
```
Write them down / keep them in this folder. Also record:
- `cat /sys/class/drm/card*-eDP-1/modes | head`  (panel resolution/refresh options)
- `hyprctl` does not exist yet — that's Phase 2.

---

## 12. Rehearse recovery (do not skip)

```bash
sudo snapper -c root create -d "known-good"
# make a harmless change (e.g. install a package)
sudo pacman -S --needed cowsay
# roll back that package change is not enough; test the boot path:
# reboot → GRUB → "Arch Linux (snapshot: known-good)" → log in → sudo snapper rollback <N>
```
Procedure to write down (paper/phone):
1. Reboot → GRUB → pick the snapshot entry
2. Log in
3. `sudo snapper -c root list` → find the target number
4. `sudo snapper rollback <N>` → reboot
5. `sudo grub-mkconfig -o /boot/grub/grub.cfg` if the menu looks stale

---

## 13. Troubleshooting

**No USB in the F8 menu** → Secure Boot still on, CSM on, or try another port/USB.

**GRUB prompt (`grub>`) after install**
```
grub> ls
grub> set root=(hd0,gpt1)
grub> configfile /grub/grub.cfg
```
If that boots, fix permanently from inside the system: `sudo grub-mkconfig -o /boot/grub/grub.cfg` and `grub-install …` again.

**Passphrase accepted, then kernel panic / "unknown filesystem"** → `rootflags=subvol=@` missing or UUID wrong in `GRUB_CMDLINE_LINUX`. Fix from the live USB (§13.1).

**LUKS prompt loops** → wrong passphrase or a typo in `rd.luks.name=<UUID>`. Compare with `blkid -s UUID -o value /dev/nvme0n1p2`.

**No network after boot** → `ip a`; if the interface is down: `sudo systemctl restart NetworkManager`, `nmcli device wifi list`. Ethernet is the fallback.

**`pacman` signature errors** → `sudo pacman -Sy archlinux-keyring` then `sudo pacman -Syu`.

**No sound** → `wpctl status`; check `sof-firmware` + `alsa-ucm-conf` installed; `sudo dmesg | grep -i sof`.

**Suspend wakes instantly** → `cat /proc/acpi/wakeup` → disable the offender: `echo XHC > /proc/acpi/wakeup` (test first, then make it permanent with a systemd unit).

**Disk filling up** → `df -h /`, `sudo paccache -r`, `sudo snapper -c root list` and delete old snapshots.

### 13.1 Live-USB recovery (chroot)
```bash
cryptsetup open /dev/nvme0n1p2 cryptroot
OPTS="noatime,compress=zstd:3,ssd,discard=async"
mount -o $OPTS,subvol=@ /dev/mapper/cryptroot /mnt
mount -o $OPTS,subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o $OPTS,subvol=@var_log /dev/mapper/cryptroot /mnt/var/log
mount /dev/nvme0n1p1 /mnt/boot
arch-chroot /mnt
# fix grub / mkinitcpio / fstab here
grub-mkconfig -o /boot/grub/grub.cfg
exit; umount -R /mnt; cryptsetup close cryptroot; reboot
```

### 13.2 Snapper rollback from the live USB
```bash
# same mounts as 13.1, then:
sudo snapper -c root rollback <N>     # or: mv @ @old; btrfs subvolume snapshot @snapshots/N @
```

---

## 14. Hard rules from here on

1. `pacman -Syu` only. Never `pacman -Sy <pkg>`.
2. **No AUR, no third-party repos.** `pacman -Qm` empty is the invariant.
3. No `--overwrite` / `--nodeps` / `--force`. No `make install` into `/usr`.
4. Reboot after kernel/systemd/glibc updates; then run the smoke test below.
5. Never upgrade under ~5 GB free.
6. `pacdiff` monthly.
7. Read archlinux.org/news before big upgrades.

**Smoke test** (save as `~/smoke-test.sh`):
```bash
#!/bin/bash
echo "== failed units =="; systemctl --failed --no-legend
echo "== errors this boot =="; journalctl -b -p 3 --no-pager | tail -20
echo "== network =="; ping -c1 -W2 archlinux.org && echo ok
echo "== audio =="; wpctl status | head -12
echo "== disk =="; df -h / | tail -1; zramctl
echo "== gpu =="; nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
echo "== sensors =="; sensors | head -12
```

---

## 15. Phase 1 done = all of these true

- [ ] boots twice; LUKS prompt both times
- [ ] GRUB shows `linux`, `linux-lts` **and** snapshot entries
- [ ] `pacman -Qm` → empty
- [ ] `snapper -c root list` → `clean-base`, `known-good`
- [ ] `prime-run glxinfo -B` → NVIDIA; plain `glxinfo -B` → Intel
- [ ] audio in/out, zram active, charge limit 80, suspend clean
- [ ] `systemctl --failed` empty; no recurring dmesg errors
- [ ] baselines recorded (§11)
- [ ] one rollback rehearsed (§12)
- [ ] smoke test passes

---

## 16. Handover to Phase 2

On the fresh Arch system:

```bash
sudo pacman -S --needed opencode git
```

(`opencode 2.1.2` is in `extra` — no AUR.)

Then:
```bash
mkdir -p ~/dotfiles && cd ~/dotfiles && git init
```
Copy `PHASE2-CONTEXT.md` and `design-sheet.md` (from `~/.opencode/plan/` on the old machine) somewhere you can reach — e.g. into `~/dotfiles/design/`.

Save these for the Phase 2 session:
- input devices: `ls /dev/input/by-path/` (keyboard names for Hyprland binds)
- panel modes: `cat /sys/class/drm/card*-eDP-1/modes | head`
- baseline temps/power from §11
- `asusctl profile -p`, `asusctl battery` output
