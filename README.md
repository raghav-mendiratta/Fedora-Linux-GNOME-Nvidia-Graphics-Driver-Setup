# Asus TUF Gaming F15 (FX507ZV4) Fedora Linux Nvidia Setup & Troubleshooting Guide

This comprehensive guide details the exact step-by-step procedure to install, configure, and stabilize the proprietary Nvidia display drivers (version 610.57.04+) on an **Asus TUF Gaming F15 (FX507ZV4)** running **Fedora Linux** on GNOME/Wayland. It covers dynamic power management via supergfxctl, fixing early Kernel Mode Setting (KMS) display freezes/black screens during pure dGPU MUX switching, and integrating hardware telemetry.

## 1. Hardware & Software Specifications

---

| Component | Specification / Detail |
| :--- | :--- |
| **Laptop Model** | ASUS TUF Gaming F15 (FX507ZV4) |
| **CPU** | Intel Core i7-12700H |
| **GPU** | NVIDIA GeForce RTX 4060 Laptop GPU (115W) |
| **Display Panel** | BOE NE156FHM-NX6 Panel (eDP Interface) |
| **Operating System** | Fedora Linux (Kernel 7.2.x, GNOME / Wayland) |
| **Nvidia Driver** | Proprietary Driver 610.57.04 (via RPM Fusion / akmods) |

## 2. Prerequisites & Firmware Requirements

---

### 2.1 Disable Secure Boot

Nvidia kernel modules compiled via akmods will fail to load if Secure Boot is active, resulting in nvidia-smi communication errors.

1. Reboot the laptop and press F2 to enter the ASUS UEFI/BIOS setup screen.
2. Press F7 to switch to **Advanced Mode**.
3. Navigate to the **Security** tab > **Secure Boot**.
4. Set **Secure Boot Control** to **Disabled**.
5. Press F10 to save changes and exit.

## 3. Nvidia Driver Installation & Kernel Module Compilation

---

### 3.1 Enable RPM Fusion Repositories

Execute the following commands in the terminal to enable the non-free repositories required for proprietary Nvidia drivers:

```bash
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
sudo dnf config-manager --set-enabled rpmfusion-nonfree-nvidia-driver
```

### 3.2 Install the Driver and Build Dependencies

```bash
sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda kmod-nvidia
```

### 3.3 Force Rebuild Kernel Modules

If the kernel update outpaces the pre-compiled driver package, force a rebuild using akmods:

```bash
sudo akmods --force --rebuild
```

Verify that the kernel module was successfully generated and matched to your active kernel:

```bash
rpm -qa | grep kmod-nvidia
```

## 4. Fixing the Pure dGPU Black Screen / MUX Switch Freeze

---

### 4.1 Root Cause Analysis

When switching to pure dGPU mode (AsusMuxDgpu), the physical hardware MUX switch routes the BOE display panel directly to the RTX 4060 (or whatever graphic card you have), bypassing the Intel/AMD iGPU. During boot, Plymouth (Fedora's graphical boot logo) attempts to initialize early display drivers via simple DRM/KMS before the Nvidia proprietary driver takes control of the bus. This handoff fails, causing a permanent blank screen freeze before GDM loads.

### 4.2 Permanent Fix: Disabling Early KMS via GRUB (Do this AFTER setting up Nvidia Drivers properly)

Adding nomodeset to the kernel boot parameters bypasses early kernel display initialization, allowing GDM and the proprietary Nvidia driver to take control of the display directly upon boot.

1. Open the default GRUB configuration file:

   ```bash
   sudo nano /etc/default/grub
   ```

2. Locate the `GRUB_CMDLINE_LINUX` line and append nomodeset inside the quotes:

   ```bash
   GRUB_CMDLINE_LINUX="rhgb quiet nomodeset"
   ```

3. Save and exit (Press Ctrl + X -> y -> Enter respectively).
4. Regenerate the GRUB bootloader configuration:

   ```bash
   sudo grub2-mkconfig -o /boot/grub2/grub.cfg
   ```

## 5. TTY Rescue & Recovery Workflow

---

**Emergency Recovery:** If the system ever encounters a black screen during boot due to GPU mode changes or cache corruption, use this exact sequence to regain access without reinstalling.

### Step-by-Step TTY Rescue Process:

1. When stuck at a blank screen or boot menu, restart and highlight Fedora in GRUB.
2. Press e to edit the boot parameters.
3. Scroll down to the line beginning with linux or linux16.
4. Move to the word rhgb quiet. Remove that word and replace it with this word:

   ```text
   3 nomodeset
   ```

5. Press Ctrl + X or F10 to boot directly into text mode (TTY).
6. Log in with your username and password.
7. Force supergfxctl back to Hybrid mode:

   ```bash
   supergfxctl -m Hybrid
   sudo systemctl restart supergfxd
   ```

8. Clear GNOME and Mutter session caches:

   ```bash
   rm -rf ~/.cache/mutter ~/.cache/gnome-shell
   ```

9. Reboot cleanly:

   ```bash
   sudo reboot
   ```

## 6. GPU Mode Switching & Daily Operation

---

### 6.1 GPU Modes Comparison

| Mode | Command | Power Consumption | Best Used For |
| :--- | :--- | :--- | :--- |
| **Hybrid** | `supergfxctl -m Hybrid` | ~2W (Idle) | General daily usage, maximum power saving, safe performance offloading via prime-run. |
| **AsusMuxDgpu** | `supergfxctl -m Dedicated` | 15W–115W | Direct MUX routing, maximum frame rate consistency without desktop composition overhead. |
| **Integrated** | `supergfxctl -m Integrated` | 0W (dGPU Powered Down) | Maximum battery life; completely disables the RTX 4060 on the PCI bus. |

## 7. Verification Commands Reference

---

| Verification Target | Command | Expected Successful Output |
| :--- | :--- | :--- |
| **Driver & VRAM Status** | `nvidia-smi` | Displays driver version 610.57.04, RTX 4060 info, VRAM allocation, and active processes. |
| **Kernel Modules** | `lsmod \| grep nvidia` | Lists active modules: nvidia, nvidia_modeset, nvidia_uvm, nvidia_wmi_ec_backlight. |
| **Active OpenGL Renderer** | `glxinfo \| grep "OpenGL renderer"` | Outputs NVIDIA GeForce RTX 4060 Laptop GPU/PCIe/SSE2. |
| **PRIME Offloading Test** | `__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia glxinfo \| grep "OpenGL renderer"` | Explicitly confirms OpenGL rendering offloaded to the Nvidia GPU in Hybrid mode. |
| **supergfxctl Status** | `supergfxctl -g` | Reports active mode (e.g., Hybrid or AsusMuxDgpu). |
