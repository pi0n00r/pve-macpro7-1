# GPU Passthrough on Mac Pro 7,1 with Proxmox VE

This fork tracks
[AdityaGarg8/pve-edge-kernel-t2](https://github.com/AdityaGarg8/pve-edge-kernel-t2)
and retains a field-tested GPU-passthrough guide for the 2019 Mac Pro 7,1.

The build source follows upstream. Installable T2 kernel and header packages are
published on the
[upstream releases page](https://github.com/AdityaGarg8/pve-edge-kernel-t2/releases).

## Validated baseline

The passthrough procedure was originally validated with:

- Hardware: Mac Pro 7,1 (2019)
- GPU: AMD RX 580 / Pro 580X
- Hypervisor: Proxmox VE 9.1.2, upgraded from the 9.1.1 ISO
- Kernel: `6.17.2-2-pve-t2`
- Validation: piKVM video output and USB-controller passthrough

This records a known-good baseline; it does not claim that every newer kernel
has been qualified on this hardware.

### Known limitations

Wi-Fi and Bluetooth did not work in the validated installation. This guide does
not claim or qualify support for either interface.

## Install a T2 kernel

Download the matching kernel and header packages from the upstream release you
intend to run, then install the downloaded files with `apt`. Current releases
use `proxmox-kernel-*` and `proxmox-headers-*` package names.

```bash
sudo apt install ./proxmox-kernel-VERSION_amd64.deb \
  ./proxmox-headers-VERSION_amd64.deb
```

Keep a previously bootable kernel installed and retain remote-console access
until the new kernel and passthrough configuration have been verified.

## Identify devices

```bash
lspci -nn | grep -i amd
lspci | grep USB
lsusb -t
```

Confirm the actual GPU and HDMI-audio PCI IDs before editing `vfio.conf`. The
validated RX 580 example used `1002:67df` and `1002:aaf0`; do not assume those
IDs match another GPU.

## Configure VFIO

Create or update `/etc/modprobe.d/vfio.conf`:

```text
options vfio-pci ids=1002:67df,1002:aaf0 disable_vga=1
```

The validated host prevented its graphics drivers from claiming the passthrough
GPU using `/etc/modprobe.d/blacklist.conf`:

```text
blacklist radeon
blacklist amdgpu
blacklist drm
```

These blacklist entries affect the host globally. Apply them only when the host
must not use the affected GPU and an independent console such as piKVM is
available.

Create or update `/etc/modprobe.d/kvm.conf`:

```text
options kvm ignore_msrs=1 report_ignored_msrs=0
```

Ensure `/etc/modules` contains:

```text
vfio
vfio_iommu_type1
vfio_pci
```

Rebuild the initramfs and reboot:

```bash
sudo update-initramfs -u
sudo reboot
```

## Verify binding

```bash
lspci -k -nn -d 1002:67df
```

The selected GPU should report `vfio-pci` as its in-use kernel driver.

## Configure and validate the VM

1. Use OVMF/UEFI firmware for the guest.
2. Assign the GPU and its required companion functions to the VM.
3. Assign the intended USB controller when direct keyboard and mouse input are
   required.
4. Verify physical video output through piKVM.
5. In a Linux guest, verify hardware rendering:

```bash
glxinfo | grep renderer
```

The validated Ubuntu guest reported the AMD Radeon RX 580 renderer. An RDP
session can report `llvmpipe` even while the physical guest display uses GPU
acceleration. RDP also forwards audio independently, so blacklisting
`snd_hda_intel` was not required by the validated setup.

## Remove a kernel

Remove only the specific superseded kernel and header packages after another
kernel has been verified bootable:

```bash
sudo apt remove proxmox-kernel-VERSION proxmox-headers-VERSION
```

## Credits

- [AdityaGarg8/pve-edge-kernel-t2](https://github.com/AdityaGarg8/pve-edge-kernel-t2)
- [fabianishere/pve-edge-kernel](https://github.com/fabianishere/pve-edge-kernel)
- [t2linux](https://wiki.t2linux.org/)
