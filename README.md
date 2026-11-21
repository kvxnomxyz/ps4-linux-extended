# ps4-linux-extended

Arch-based Linux on the PlayStation 4, set up as a full desktop with KDE Plasma, development tools, and local AI tooling.

Built around a **Baikal (B1) PS4** running the **Strawberry kernel**, `psxitarch`, and the PS4-patched Mesa (`mesa-ps4`).

> **Disclaimer:** This is a hobby project and is not affiliated with Sony. You need a jailbroken console. Use it at your own risk; modifying a console can brick it or void your warranty.

---

## What this is

A documented, repeatable setup for turning a PS4 into a usable Linux desktop:

- Boot from USB using a PS4 Linux loader payload
- `psxitarch` (Arch-based) rootfs on a dedicated ext4 partition
- KDE Plasma on X11
- Build tools, VS Code, OpenCode and Ollama
- Notes on what works, what doesn't, and the traps to avoid

## Tested on

| Item | Value |
|------|-------|
| Console | PS4 with Baikal B1 southbridge |
| Kernel | Strawberry 7.3.3 |
| Distro | psxitarch (Arch-based) |
| Desktop | KDE Plasma (X11 session) |
| Firmware | 14.02 |
| Model | PS4 Slim (Baikal) |

### Firmware compatibility

Using the online exploit via [WebKitty](https://webkitty.arabpixel.net/), firmware **5.05** and **6.72 up to the latest** are supported. Actual support depends on your **console revision**, so check what WebKitty reports for your unit before you start.

## Requirements

- A jailbroken PS4 with GoldHEN (or another payload-capable setup)
- A USB drive (an SSD in an enclosure is much faster)
- A keyboard and mouse
- A USB Ethernet or WiFi adapter (the built-in WiFi is unreliable)
- A PC with `parted`, `tar` and `curl` for preparing the drive

## Disk layout

| Partition | Size | Type | Label | Purpose |
|-----------|------|------|-------|---------|
| `sda1` | 50 MiB | FAT32 | none | Boot files (`bzImage`, `initramfs`, `bootargs.txt`) |
| `sda2` | rest | ext4 | `psxitarch` | Root filesystem |

> **Warning:** the commands below erase the whole drive. Check the device name with `lsblk` before running them.

```bash
sudo umount /dev/sda* 2>/dev/null
sudo wipefs -a /dev/sda
sudo parted -s /dev/sda mklabel msdos mkpart primary fat32 1MiB 51MiB mkpart primary ext4 51MiB 100% set 1 boot on
sudo mkfs.vfat -F32 -s 1 /dev/sda1
sudo mkfs.ext4 -F -L psxitarch /dev/sda2
```

## Install

1. **Extract the rootfs** onto the ext4 partition:
   ```bash
   sudo tar -xpf psxitarch*.tar.xz -C /run/media/$USER/psxitarch
   ```
2. **Copy the boot files** to the FAT32 partition:
   ```bash
   cp bzImage initramfs.cpio.gz bootargs.txt /path/to/fat32/
   sync
   ```
3. **Boot:** plug the drive into the PS4, jailbreak, then send the Linux loader payload (largest VRAM size for a desktop). Use WebKitty if you want to do it online, or the BIN payloads from `arabpixel/ps4-linux-loader` if you prefer offline. The screen may stay black for 1-2 minutes.

If the loader can't find the files on the USB, copy them to `/data/linux/boot/` on the console over GoldHEN FTP (port 2121):

```bash
curl --ftp-create-dirs -T bzImage ftp://PS4_IP:2121/data/linux/boot/
curl -T initramfs.cpio.gz ftp://PS4_IP:2121/data/linux/boot/
```

## First boot

```bash
sudo timedatectl set-ntp true
sudo pacman-key --init && sudo pacman-key --populate archlinux
```

### Lock the PS4 graphics stack (important)

`psxitarch` ships a PS4-patched Mesa (`mesa-ps4`). Replacing it with vanilla Arch Mesa breaks GPU acceleration on Baikal consoles. Pin it **before** any system update:

```bash
sudo sed -i '/^#\?IgnorePkg/d' /etc/pacman.conf
sudo sed -i '/^\[options\]/a IgnorePkg = mesa-ps4 lib32-mesa-ps4 llvm-libs' /etc/pacman.conf
grep IgnorePkg /etc/pacman.conf
```

If pacman ever asks to remove or replace `mesa-ps4`, answer **N**.

### KDE Plasma (X11)

```bash
sudo pacman -Syu --needed plasma-meta konsole dolphin kate sddm networkmanager xorg-server
sudo systemctl enable NetworkManager sddm
sudo reboot
```

Pick **Plasma (X11)** at the login screen. Wayland is likely to hang on this GPU.

### Development tools

```bash
sudo pacman -S --needed base-devel git cmake ninja meson make pkgconf gcc gdb valgrind nasm go rustup nodejs npm yarn python python-pip jdk-openjdk maven gradle
rustup default stable
```

`clang` and `llvm` are intentionally left out because they need a matching `llvm-libs`, which is pinned above.

### Editor, AI tools

```bash
paru -S --needed visual-studio-code-bin opencode
sudo pacman -S --needed ollama
sudo systemctl enable --now ollama
```

Ollama runs on the CPU only (the PS4 GPU isn't supported by ROCm). Stick to small models, for example `ollama run qwen2.5-coder:1.5b`.

If VS Code shows a blank window, launch it with `code --disable-gpu`.

## Known issues

- It is a soft mod: the exploit must be re-run on every boot.
- Built-in WiFi is unreliable; use a USB adapter.
- Wayland sessions can hang; use X11.
- GPU and wireless firmware blobs must come from your own PS4 firmware dump.
- Do not run a blind `pacman -Syu` without the `IgnorePkg` lock in place.

## Troubleshooting

| Problem | Try |
|---------|-----|
| Black screen after login | `Ctrl+Alt+F3`, then `journalctl -b -p err \| tail -30` |
| Pacman signature errors | Fix the clock: `sudo timedatectl set-ntp true` |
| No network | Check `ip a`, then `nmtui`; use a USB adapter |
| Drive is read-only | Run `fsck` on the partition, then replug |

## Credits

- [PS4 Linux tutorial](https://dionkill.github.io/ps4-linux-tutorial/) – main walkthrough
- [arabpixel/ps4-linux-loader](https://github.com/arabpixel/ps4-linux-loader) – offline BIN payloads (the old `ps4-linux/ps4-linux-loader` repo is private)
- [WebKitty](https://webkitty.arabpixel.net/) – online jailbreak and PS4 Linux payloads
- [rmuxnet/linux](https://gitlab.com/rmuxnet/linux) – Strawberry kernel (use 7.3.3 for a Slim on Baikal)
- [hippie68/psxitarch-how-to](https://github.com/hippie68/psxitarch-how-to) – psxitarch guide and releases

## License

_Choose a license (for example MIT) and add a `LICENSE` file._
