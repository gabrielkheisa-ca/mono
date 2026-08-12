# Arch Linux `bootc` Disk Image Builder

This document describes how to build a bootable disk image (`disk.raw`) from the `arch-bootc` container image using `bootc install to-filesystem`.

---

## Workspace Structure

All commands in this guide are executed from the repository root or the `arch/` directory:

```text
mono/
├── arch/
│   ├── Containerfile
│   ├── config.json
│   └── output/
│       ├── arch-bootc.tar
│       └── disk.raw
└── shared/
```

---

## 1. Build the Squashed Container Image

When building `bootc` images, multiple `RUN` statements in a `Containerfile` can generate redundant OSTree commit metadata objects across layers, leading to `error: Multiple commit objects found`. 

Always build with `--squash` to collapse all layers into a single clean image state:

```bash
cd ~/cloud-ace/mono/arch

# Build with --squash to prevent multi-commit OSTree conflicts
podman build --squash -t localhost/arch-bootc:latest .
```

---

## 2. Prepare the Disk & Partitions

Create a 10GB raw disk file, attach it as a loop device with partition scanning enabled (`-P`), and format it with GPT partitions (512MB EFI + Btrfs Root).

```bash
# 1. Clean up any existing loop mounts
sudo umount -R /tmp/target 2>/dev/null || true
sudo losetup -D

# 2. Create raw disk file
mkdir -p output
truncate -s 10G output/disk.raw

# 3. Attach loop device
LOOP_DEV=$(sudo losetup -P --find --show output/disk.raw)
echo "Attached loop device: $LOOP_DEV"

# 4. Partition disk (512M EFI, Remaining Btrfs)
sudo sfdisk $LOOP_DEV <<EOF # ## $LOOP_DEV ${LOOP_DEV}p1${LOOP_DEV}p2 (avoiding * , - --- -F -L -f -n 3. 32 5. 512M, EFI-SYSTEM EOF Filesystems Format L, Mount Target U, `/mnt `/tmp/target` ``` an directory gpt host isolated, label: like mkfs.btrfs mkfs.fat non-symlinked partitions partprobe root sudo the to> /var/mnt` collisions):

```bash
# Create dedicated mountpoint
sudo mkdir -p /tmp/target

# Mount Btrfs root
sudo mount -o compress=zstd ${LOOP_DEV}p2 /tmp/target

# Mount EFI System Partition
sudo mkdir -p /tmp/target/boot/efi
sudo mount ${LOOP_DEV}p1 /tmp/target/boot/efi
```

---

## 4. Install `bootc` to Filesystem

Run `bootc install to-filesystem` directly on the host using your local Podman storage (`containers-storage:`). 

> **Note:** Running `bootc` natively on host targets ONLY the explicitly provided path (`/tmp/target`) and will **not** modify or affect your host OS (e.g., Bazzite/Atomic Desktop).

```bash
# Run bootc installation from local host Podman storage
sudo bootc install to-filesystem \
  --type uninitialized \
  --source-imgref containers-storage:localhost/arch-bootc:latest \
  /tmp/target
```

---

## 5. Clean Up Mounts

Unmount the target filesystem and detach the virtual loop device:

```bash
sudo umount -R /tmp/target
sudo losetup -d $LOOP_DEV
sudo rmdir /tmp/target
```

Your bootable raw image is saved at `./output/disk.raw`.

---

## 6. Test in QEMU

Verify your newly minted bootable Arch Linux OS:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 \
  -smp 4 \
  -drive file=./output/disk.raw,format=raw \
  -bios /usr/share/ovmf/x86/OVMF.fd \
  -net nic,model=virtio -net user
```

---

## Troubleshooting Guide

### Issue 1: `image not known` or `podman save` fails
* **Symptom:** `Error: localhost/arch-bootc:latest: image not known`
* **Cause:** The image was built under `root` (`sudo podman build`) or in a different user storage scope, so standard `podman` cannot locate it.
* **Fix:** Check image availability with `sudo podman images` vs `podman images`. Pass `containers-storage:` directly to host `sudo bootc install`.

### Issue 2: `error: Multiple commit objects found`
* **Symptom:** `bootc` aborts when reading an OCI archive or local storage reference.
* **Cause:** Container intermediate build layers contain multiple OSTree commit references.
* **Fix:**
  1. Rebuild the container with `podman build --squash`.
  2. Pass `--type uninitialized` to `bootc install to-filesystem`.

### Issue 3: `Initializing images: No such file or directory (os error 2)`
* **Symptom:** Containerized `podman run ... bootc install` fails when inspecting internal image stores.
* **Cause:** `bootc` running inside an isolated container namespace fails to resolve host `/var/lib/containers/storage` paths.
* **Fix:** Run `bootc install to-filesystem` **natively on the host machine** rather than inside a Podman container.

### Issue 4: `Found empty directory: cache`
* **Symptom:** `Requiring directory contains only mount points: Found empty directory: cache`
* **Cause:** Host symlink `/mnt -> var/mnt` exposes host `/var/cache` folders, or Btrfs created an initial subfolder during mount.
* **Fix:** Mount exclusively to an isolated path like `/tmp/target` and clear its contents prior to calling `bootc`.