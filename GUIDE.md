# Guide: Building & Booting Immutable Arch Linux (`bootc`) on Bazzite / Container-First Systems

This guide explains how to build the immutable Arch Linux (`bootc`) container image, generate a bootable VM disk, and test it, tailored specifically for **Bazzite** (and other Fedora Silverblue/Kinoite-like immutable systems). 

By design, this guide avoids polluting your host operating system and maximizes the use of **Podman** and isolated container environments.

---

## Prerequisites (Bazzite-Optimized)

Bazzite comes pre-configured with most of what you need:
- **Podman**: Native and fully configured.
- **Just**: Pre-installed on Bazzite out of the box.
- **Flatpak / Distrobox**: Pre-installed for isolated tools.

To run virtual machines without layering packages onto your host with `rpm-ostree`, we will use **GNOME Boxes (Flatpak)**, **virt-manager (Flatpak)**, or a **Distrobox container** for command-line tools.

---

## 1. Configure User Accounts & Passwords (CRUCIAL)

By default, container-native base images are built completely locked down for security—there are no default users or passwords. Before building, you should add your user account and passwords directly to your container image configuration.

Open **`arch/Containerfile`** and add the following block near the bottom (right before the `LABEL containers.bootc 1` line):

```dockerfile
# 1. Install sudo so you can run administrator commands
RUN pacman -Sy --noconfirm sudo && pacman -S --clean --noconfirm

# 2. Create your user account and add it to the admin (wheel) group
# (Note: we use /var/home as the home directory, which bootc manages as writable)
RUN useradd -m -d /var/home/gabriel -G wheel -s /bin/bash gabriel

# 3. Define build arguments with default values for passwords
ARG ROOT_PASSWORD=archlinux
ARG GABRIEL_PASSWORD=archlinux

# 4. Set passwords for root and your new user
RUN echo "root:${ROOT_PASSWORD}" | chpasswd && \
    echo "gabriel:${GABRIEL_PASSWORD}" | chpasswd

# 5. Allow members of the wheel group to use sudo
RUN echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
```
*Change `gabriel` to your preferred username. The passwords default to `archlinux` but can be securely set via a `.env` file in the root directory (which is ignored by git):*

```env
ROOT_PASSWORD=your_secure_root_password
GABRIEL_PASSWORD=your_secure_user_password
```

---

## 2. Build the Immutable Arch Linux Container Image

Since the build is containerized, it runs perfectly inside Podman without needing any local toolchains. Run this command from the **root directory** of the repository (where the `Justfile` is located):

```bash
just build arch
```

*This creates a local container image named `localhost/arch-bootc:latest` in your user-space Podman registry. It does not touch your host filesystem.*

---

## 3. Generate a Bootable VM Disk (Persistent)

The `disk-image` recipe runs `bootc install to-disk` inside a privileged container to lay down the OS and bootloader into a raw disk image file.

From the root directory, run:
```bash
# If you have an old bootable.img, delete it first to ensure changes apply
rm -f bootable.img

# Generate the new disk
just disk-image arch
```

*This produces a **`bootable.img`** (about 20GB dynamically allocated) in your current directory. This is a fully-installed immutable Arch Linux operating system (not a transient live USB).*

---

## 4. Booting the VM Disk on Bazzite (No Host Pollution)

Since `bootc` systems require UEFI, ensure your emulator or hypervisor is configured with UEFI support.

### Option A: Using QEMU (Command Line)
If you have QEMU on your host, you must supply a UEFI firmware file (OVMF). Depending on your host distribution, the path to `OVMF` differs.

#### On Fedora / Bazzite:
If you run QEMU directly on the host, the package `edk2-ovmf` places files under `/usr/share/edk2/ovmf/`. You can boot using:
```bash
# Easiest quick-boot:
qemu-system-x86_64 -m 4096 -enable-kvm -cpu host -smp 2 \
  -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
  -drive file=bootable.img,format=raw
```

Or for a full UEFI boot that lets you save boot/BIOS settings:
```bash
# Copy the variables template locally
cp /usr/share/edk2/ovmf/OVMF_VARS.fd ./OVMF_VARS.fd

# Run QEMU with code + local vars
qemu-system-x86_64 -m 4096 -enable-kvm -cpu host -smp 2 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/ovmf/OVMF_CODE.fd \
  -drive if=pflash,format=raw,file=./OVMF_VARS.fd \
  -drive file=bootable.img,format=raw
```

#### On Ubuntu / Debian:
```bash
qemu-system-x86_64 -m 4096 -enable-kvm -cpu host -smp 2 \
  -bios /usr/share/ovmf/OVMF.fd \
  -drive file=bootable.img,format=raw
```

> **Troubleshooting: `qemu: could not load PC BIOS ...`**
> If you get this error, it means the specified `.fd` file doesn't exist on your host. If Bazzite's base image does not bundle UEFI firmware, use **Option B** (GNOME Boxes) or **Option D** (Distrobox) to avoid installing packages on your host system.

---

### Option B: Using GNOME Boxes (Flatpak) — Easiest GUI (No Host Setup)
1. Install GNOME Boxes via the Software Center or terminal:
   ```bash
   flatpak install flathub org.gnome.Boxes
   ```
2. Open GNOME Boxes, click **+** (New) -> **Create from file...** and select your `bootable.img`.
3. Select **Arch Linux** as the template and run it. GNOME Boxes Flatpak manages and bundles UEFI firmware automatically!

---

### Option C: Using Virt-Manager (Flatpak)
1. Install Virt-Manager:
   ```bash
   flatpak install flathub org.virt_manager.VirtManager
   ```
2. Open Virt-Manager. Set up a connection to **QEMU/KVM User Session** (this runs entirely within your user account without requiring host system privileges).
3. Create a new VM:
   * Select **Import existing disk image** -> Browse to `bootable.img`.
   * Check **Customize configuration before install**.
   * Under **Overview** -> **Firmware**, select **UEFI (OVMF)**.
   * Click **Begin Installation**.

---

### Option D: Running QEMU via a Distrobox (Keep Host 100% Clean)
If you prefer the command line but don't want to layer QEMU on your host, you can run QEMU and download the required bios inside an isolated Distrobox:

1. Create and enter a clean Fedora distrobox:
   ```bash
   distrobox create -n qemu-box -i registry.fedoraproject.org/fedora:latest
   distrobox enter qemu-box
   ```
2. Install QEMU and the UEFI firmware inside the distrobox:
   ```bash
   sudo dnf install -y qemu-kvm edk2-ovmf
   ```
3. Run the VM from inside the distrobox (this correctly resolves the path to `/usr/share/edk2/ovmf/OVMF_CODE.fd`):
   ```bash
   qemu-system-x86_64 -m 4096 -enable-kvm -cpu host -smp 2 \
     -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
     -drive file=bootable.img,format=raw
   ```

   Alternatively, with VGA / GPU:
   ```
   qemu-system-x86_64   -m 4096   -smp 2   -enable-kvm   -cpu host   -vga virtio   -display default,show-cursor=on   -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/ovmf/OVMF_CODE.fd   -drive file=bootable.img,format=raw
   ```
---

## 5. Ephemeral Boot Testing via `bcvk`

The repository has built-in integration with `bcvk` (Bootc Virtualization Kit) to spin up quick test VMs. Since we want to keep your host clean, you can run `bcvk` inside a temporary container or a distrobox.

If you have a testing distrobox (like the `qemu-box` created above), you can enter it, install `just` and `bcvk` (or cargo), and boot the ephemeral test:

```bash
distrobox enter qemu-box

# Inside distrobox: Install just & bcvk (or cargo) and test
sudo dnf install -y rust cargo just
cargo install bcvk

# Run ephemeral testing
just test arch
```

---

## 6. Build an Installation ISO

To package your Arch system into a standard installation ISO to flash onto a USB, use the official `bootc-image-builder` container. This process is 100% container-native and runs safely on Bazzite.

Since Bazzite uses SELinux, you must append `--security-opt label=type:unconfined_t` so Podman has permission to build the image:

```bash
# Create target output directory
mkdir -p output

# Run the image builder container
sudo podman run \
  --rm \
  -it \
  --privileged \
  --pull=always \
  --security-opt label=type:unconfined_t \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  -v ./output:/output \
  quay.io/centos-bootc/bootc-image-builder:latest \
  --type anaconda-iso \
  localhost/arch-bootc:latest
```

Once the run completes, your bootable installer ISO will be located at `./output/bootiso/install.iso`. You can write this ISO to a USB flash drive or mount it directly in a VM.
