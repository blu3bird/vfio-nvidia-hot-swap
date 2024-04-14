# Overview

This guide describes how to set up GPU hot-swapping in a VFIO gaming setup.

Goal: To be able to play games in your Windows VM and - while the VM is powered down - be able to play games in your Linux host utilizing that jucy dGPU of yours. Seamlessly. Without rebooting your PC or restarting the GUI.

## Bullet points
- Linux will primarly use the iGPU
- Specific apps (including but not limited to games) can be rendered on the dGPU
  - you can use switcheroo-control or a wrapper script to define which apps should use the dGPU
  - 3d acceleration, cuda/optix and opencl work
- dGPU can be assigned to the VM once all apps that use it have been stopped

## TOC

- [Requirements & limitations](#requirements--limitations)
  - [Two GPUs](#two-gpus)
  - [All monitors connected to your iGPU](#all-monitors-connected-to-your-igpu)
  - [Working VFIO gaming setup](#working-vfio-gaming-setup)
  - [NVIDIA dGPU and non-NVIDIA iGPU](#nvidia-dgpu-and-non-nvidia-igpu)
  - [Xorg or Wayland](#xorg-or-wayland)
  - [Systemd](#systemd)
  - [Reduced Linux gaming performance](#reduced-linux-gaming-performance)
- [Steps](#steps)
  - [Remove dGPU isolation](#steps)
  - [Install & configure NVIDIA drivers](#install--configure-nvidia-drivers)
    - [Enable modesetting](#enable-modesetting)
    - [Reboot](#reboot)
  - [Restrict applications from using the dGPU](#reboot)
    - [Wayland](#wayland)
    - [Xorg](#xorg)
    - [Regular applications](#regular-applications)
    - [Reboot](#reboot)
  - [Install switcheroo-control](#install-switcheroo-control)
  - [Setup wrapper script](#setup-wrapper-script)
    - [Workaround for unpatched switcheroo-control](#workaround-for-unpatched-switcheroo-control)
  - [Configure libvirt hooks](#configure-libvirt-hooks)
  - [Verification](#verification)
- [Final words](#final-words)

# Requirements & limitations

## Two GPUs

One GPU will be used exclusively by the host. Since quite a few people use an integrated GPU for this I will henceforth refer to this GPU as iGPU even if it is a PCIe card.
The other GPU will be shared between host and guest. Since most (all?) people use a discrede GPU for this I will henceforth refer to this as dGPU.

## All monitors connected to your iGPU

This is the main limitation of this approach: You can not use **any** of the outputs on your dGPU in Linux. Therefore you must connect all of your monitors (at least those that you want to use in Linux) to the iGPU.

Monitors connected to the dGPU (if any) will only be usable in Windows.

## Working VFIO gaming setup

You need a working VFIO gaming setup. Because setting one up is not part of this guide. See [PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) if you do no have one.

It does not matter if you use a physical monitor for graphics output or a frame relay software such as [Looking Glass](https://looking-glass.io/) or [Sunshine](https://github.com/LizardByte/Sunshine)/[Moonlight](https://github.com/moonlight-stream/moonlight-qt).

Also, what we are doing could be described as an advanced setup. You should be familiar with the basics first.

## NVIDIA dGPU and non-NVIDIA iGPU

I know from personal experience that this also works with an Intel dGPU and I have heard reports of it working with an AMD dGPU as well. But this guide is specifically tailored for a NVIDIA dGPU.

However if your iGPU (integraded or PCIe) is also a NVIDIA card you are out of luck. Because the [sysfs unbind mechanism](https://lwn.net/Articles/143397/) is unrelible for NVIDIA cards the only way to reliably "unbind" the card is by unloading the NVIDIA modules altogether. And that will not be possible if there is a 2nd NVIDIA card that is still in-use.

The only way to get this to work would be to run your iGPU using the [nouveau](https://nouveau.freedesktop.org) driver which doesn't support any cards made in the last 7 years...

## Xorg or Wayland

Both Xorg and Wayland are fine. Just don't use [MIR](https://mir-server.io/).

If you are still on Xorg you may want to consider moving to Wayland. It has all the good stuff like:
- [High dynamic range](https://en.wikipedia.org/wiki/High_dynamic_range)*
- [Variable refresh rate](https://en.wikipedia.org/wiki/Variable_refresh_rate)*
- [Fractional scaling](https://en.wikipedia.org/wiki/Resolution_independence)*
- Lots of [glitches](https://github.com/ValveSoftware/steam-for-linux/issues/10313) when using an NVIDIA card as primary GPU. Which we are not affected by because we will be using the NVIDIA card as a _secondary_ GPU.

\* Feature availabillity may vary depending on implementation.

## Systemd

In theory everything described in this guide should be possible with other service managers but most distributions use systemd nowadays. So that's what this guide assumes you have.

## Reduced Linux gaming performance

When playing in Linux frames will be rendered on the dGPU and then transfered to the iGPU for displaying (known as offloading or PRIME). This produces overhead and decreases performance but there is no way around it as the dGPU has to be used as a secondary card. We can't use the dGPU as primary GPU because then we won't be unable to swap it betwen VM and host...there simply is no support for hot-unplugging a primary GPU yet.

You should expect a performance reduction of up to 20%* compared to using your dGPU as a primary GPU without offloading.

Performance in Windows is not affected by this and is exactly the same as it is in a traditional VFIO gaming setup.

\* Worst case when using an integrated GPU as iGPU. Using a PCIe GPU as IGPU or a [MUX](https://en.wikipedia.org/wiki/Multiplexer) (only available in laptops) greatly reduces this.

# Steps

## Remove dGPU isolation

In a traditional VFIO gaming setup the dGPU is used exclusively by the VM and is isolated from the host usually by configuring `vfio_pci.ids`.

Remove the isolation. Refer to the guide you used for setting up VFIO for details.

## Install & configure NVIDIA drivers

Install the [official NVIDIA drivers](https://www.nvidia.com/download/index.aspx) using the recommended method for your distribution.

You do not need a specific version. Anything recent should do.

You can use the proprietary or the [open kernel](https://github.com/NVIDIA/open-gpu-kernel-modules) version of the driver. For all intents and purposes both versions are virtually identical.

Hint: If you are using archlinux do not disable the kms HOOK. You still want the iGPU to be your primary GPU.

### Enable modesetting

For Wayland support you must enable modesetting. This can be done by setting `nvidia_drm.modeset=1` in _/etc/modprobe.d/nvidia.conf_:
```
# Kernel Mode Setting (notably needed for fbdev and wayland).
# Enabling may possibly cause issues with SLI and Reverse PRIME.
options nvidia-drm modeset=1
```

### Enable persistenced/powerd (optional)

If you want to use [persistenced](https://download.nvidia.com/XFree86/Linux-x86_64/550.67/README/nvidia-persistenced.html) and/or [powerd](https://download.nvidia.com/XFree86/Linux-x86_64/550.67/README/dynamicboost.html) enable them:
```
systemctl enable nvidia-persistenced.service
systemctl enable nvidia-powerd.service
```

### Reboot

Reboot your system and use `nvidia-smi` afterwards to verify your dGPU is usable in Linux.

Hint: If you are running a graphical session Linux may use your dGPU as output. You can avoid this by temporaritly unplugging any monitor or dummy plug from the dGPU.

## Restrict applications from using the dGPU

We want to restrict all applications from using the dGPU without beeing explicitly told to use it. Because to be able to assign the dGPU to the VM it must not be in-use. And we don't want to stop/kill a bunch of applications everytime we start the VM.

### Wayland

The [Wayland specification](https://wayland.freedesktop.org/docs/html/apa.html) includes support for [multi-seat](https://www.freedesktop.org/wiki/Software/systemd/multiseat/). Every implementation (Gnome, KDE/Plasma or Sway) should therefore support multi-seat. We can use that to tell the login manager and the compositor to completly ignore the dGPU.

Create _/etc/udev/rules.d/99-hidden-seat.rules_ to configure multi-seat:
```
# assign nvidia card to seat1 but remove master-of-seat tag to force CanGraphical=false
# this causes wayland login managers and compositors to completely ignore it
TAG=="seat", ENV{ID_FOR_SEAT}=="drm-pci-0000_01_00_0", ENV{ID_SEAT}="seat1", TAG-="master-of-seat"
```

You may need to update _ID_FOR_SEAT_. It's basicly the PCI bus id of your dGPU. You can use `udevadm info /dev/dri/card* | grep ID_FOR_SEAT` to list possible values and after comparing it with `lspci | grep VGA` you should be able to identify the correct one.

### Xorg

Even if your desktop environment uses Wayland you might still be using an Xorg-based login manager. Therefore it is recommended to always create a reasonable xorg config.

If you are using an AMD iGPU, create _/etc/X11/xorg.conf.d/20-amdgpu.conf_:
```
Section "OutputClass"
        Identifier "AMD GPU"
        MatchDriver "amdgpu"
        Driver "amdgpu"
EndSection
```

If you are using an Intel iGPU, create _/etc/X11/xorg.conf.d/20-modesetting.conf_:
```
Section "Device"
        Identifier "KMS GPU"
        Driver "modesetting"
EndSection
```

And finally, configure Xorg to not add any GPUs you did not explicitly configure by creating _/etc/X11/xorg.conf.d/99-no-autoaddgpu.conf_:
```
Section "ServerFlags"
        Option "AutoAddGPU" "false"
EndSection
```

### Regular applications

By default all regular applications should be using the iGPU instead of the dGPU. But *should* is not good enought.

Vulkan has built-in support for multiple GPUs. Any vulkan app could simply use the dGPU if it so desires. And they will definitly do that. A good example is [Google Chrome](https://www.google.com/intl/en_US/chrome/). Simply navigate to [chrome://gpu](chrome://gpu) and it will list (and therefore lock) all GPUs.

Create _/etc/profile.d/force-igpu_:
```
# force vulkan apps to ignore nvidia card
export VK_LOADER_DRIVERS_DISABLE=nvidia_icd.json
```

Another candidate for accessing the dGPU without being told to is Xwayland. And since there is no Xwayland equivalent to _xorg.conf.d_ we need to force it to mesa using environment variables.

Update _/etc/profile.d/force-igpu_ and append:
```
# force opengl apps (specifically Xwayland) to only use mesa
export __EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/50_mesa.json
export __GLX_VENDOR_LIBRARY_NAME=mesa
```

And last but not least we have powerdevil (KDE only). If you have a [DDC](https://de.wikipedia.org/wiki/Display_Data_Channel) capable monitor attached to your dGPU powerdevil will try to control it's brightness through the dGPU and thus preventing us from assigning it to the VM.

Update _/etc/udev/rules.d/99-hidden-seat.rules_ you created earlier and append:
```
# also assign i2c device nodes to this seat to hide them from powerdevil
TAG=="seat", ENV{ID_FOR_SEAT}=="i2c-dev-pci-0000_01_00_0", ENV{ID_SEAT}="seat1"
```

Note: This will not only prevent powerdevil from managing the monitor but restrict access for everyone except the root account. You can still control it with tools such as [ddcutil](https://www.ddcutil.com/) run as root/sudo. Again you may need to update _ID_FOR_SEAT_. Refer to the previous step for details.

### Reboot

Reboot your system.

Hint: If you temporaritly unplugged your dGPU outputs you can plug them back in. Linux won't use your dGPU as output anymore.

## Install switcheroo-control

Switcheroo-control provides a convinient way to select which applications should use the dGPU. However there is a small obsticale. In the previous step we configured a bunch of environment variables to restrict applications from using the dGPU. We need to tell switcheroo-control to unset these environment variables if an application is launched on the dGPU. And sadly the only way to do that is by patching it's source code.

This is te most convinient way but if you are unwilling or unable to patch switcheroo-control there's a workaround available. In that case skip this step.

The easiest way to patch switcheroo-control is to install it like you would install any regular package and then just replace the main binary with one you compiled from (patched) source:
```
git clone https://gitlab.freedesktop.org/hadess/switcheroo-control.git
cd switcheroo-control
patch -p0 << EOF
--- src/switcheroo-control.c
+++ src/switcheroo-control.c
@@ -254,6 +254,12 @@ get_card_env (GUdevClient *client,
 		/* Make sure Vulkan apps always select Nvidia GPUs */
 		g_ptr_array_add (array, g_strdup ("__VK_LAYER_NV_optimus"));
 		g_ptr_array_add (array, g_strdup ("NVIDIA_only"));
+
+		/* Undo global gpu restriction */
+		g_ptr_array_add (array, g_strdup ("VK_LOADER_DRIVERS_DISABLE"));
+		g_ptr_array_add (array, g_strdup (""));
+		g_ptr_array_add (array, g_strdup ("__EGL_VENDOR_LIBRARY_FILENAMES"));
+		g_ptr_array_add (array, g_strdup (""));
 	} else {
 		char *id;
EOF
meson _build
ninja -v -C _build

# some distributions wrongfully put the binary in /usr/lib/, check before copying
sudo cp ./_build/src/switcheroo-control /usr/libexec/
```

Note: Switcheroo-control is rarely updated, but you will still need to redo this everytime you update it.

Enable the switcheroo-control service: `systemctl enable --now switcheroo-control.service`

You can now configure applications to run on the dGPU by adding `PrefersNonDefaultGPU=true` and/or `X-KDE-RunOnDiscreteGpu=true` to their _.desktop_ files. Most games (or game-like applications) like [Steam](https://steampowered.com/) will already have this as a default.

## Setup wrapper script

A wrapper script is the simplest way to run applications on the dGPU. Even if you opted to patch switcheroo-control, you should still setup a script like this. It's quite handy.

Create _/usr/local/bin/nvidia-run_:
```
#!/bin/sh

if [ -d /sys/module/nvidia ]; then
        unset VK_LOADER_DRIVERS_DISABLE
        unset __EGL_VENDOR_LIBRARY_FILENAMES

        export __GLX_VENDOR_LIBRARY_NAME=nvidia
        export __NV_PRIME_RENDER_OFFLOAD=1
        export __VK_LAYER_NV_optimus=NVIDIA_only
else
        echo "Warning: dGPU not available. Application will run on iGPU." 1>&2
fi

exec "${@}"
```

And mark it executable: `chmod a+x /usr/local/bin/nvidia-run`

You can now use the wrapper script to start applications on the dGPU using a terminal.

### Workaround for unpatched switcheroo-control

If you opted not to patch switcheroo-control you will have to overwrite all _.desktop_ files of the applications you want to run on the dGPU and prepend _nvidia-run_ to _Exec_.
```
--- /usr/share/applications/supertuxkart.desktop
+++ /usr/share/applications/supertuxkart.desktop
@@ -1,2 +1,2 @@
 [Desktop Entry]
-Exec=supertuxkart
+Exec=nvidia-run supertuxkart
```

## Configure libvirt hooks

By default libvirt will try to unbind your dGPU from the NVIDIA driver using the [sysfs unbind mechanism](https://lwn.net/Articles/143397/). This will not work because the NVIDIA card will always be in-use by the _nvidia-drm_ module (at least with `nvidia_drm.modeset=1`).

The only way to "unbind" the card is by unloading all NVIDIA modules. Therefore we need to create a libvirt hook to unload the NVIDIA modules before the VM is started and to load them again after it is stopped.

Create hook directory: `mkdir -p /etc/libvirt/hooks/qemu.d`

Create _/etc/libvirt/hooks/qemu_:
```
#!/bin/sh
set -e

export VM_NAME="${1}"
export OPERATION="${2}"
export SUB_OPERATION="${3}"
export EXTRA_ARGUMENT="${4}"

if [ -r "/etc/libvirt/hooks/qemu.d/functions.sh" ]; then
        . "/etc/libvirt/hooks/qemu.d/functions.sh"
fi

if [ -r "/etc/libvirt/hooks/qemu.d/${VM_NAME}.sh" ]; then
        . "/etc/libvirt/hooks/qemu.d/${VM_NAME}.sh"
fi
```

And mark it executable: `chmod a+x /etc/libvirt/hooks/qemu`

Create _/etc/libvirt/hooks/qemu.d/functions.sh_:
```
#!/bin/false

load_nvidia_modules() {
        # load modules
        modprobe nvidia_uvm
        modprobe nvidia_drm

        # start services
        for service in persistenced powerd; do
            if systemctl is-enabled "nvidia-${service}.service" > /dev/null; then
                systemctl start "nvidia-${service}.service"
            fi
        done
}

unload_nvidia_modules() {
        # stop services
        for service in powerd persistenced; do
            systemctl stop "nvidia-${service}.service"
        do

        # check if the card is in use
        # note: your dGPU might have a different card and renderer index
        for dev in /dev/dri/card1 /dev/dri/renderD129 /dev/nvidia0 /dev/nvidiactl /dev/nvidia-modeset /dev/nvidia-uvm /dev/nvidia-uvm-tools; do
                if [ -c "${dev}" ]; then
                        for cmd in $(lsof -Fc "${dev}" 2> /dev/null | grep "^c" | sed -e 's/^c//'); do
                                echo "Card is in use by '${cmd}'." 1>&2
                                exit 1
                        done
                fi
        done

        # unload all modules in the correct order
        for module in nvidia_drm nvidia_modeset nvidia_uvm nvidia; do
                if [ -d "/sys/module/${module}" ]; then
                        rmmod "${module}"
                fi
        fi
}
```

Create _/etc/libvirt/hooks/qemu.d/${VM_NAME}.sh_:
```
#!/bin/false

case "${OPERATION}" in
        prepare)
                unload_nvidia_modules
                ;;
        release)
                load_nvidia_modules
                ;;
esac
```

Restart libvirtd: `systemctl restart libvirtd.service`

Hint: Even with the hook you still want `<hostdev managed='yes'>` in your xml to do the vfio binding/unbinding.

## Verification

Reboot your system and start your graphical environment (if you haven't already).

Temporarily stop nvidia-persistenced and nvidia-powerd if you have them enabled:
```
systemctl stop nvidia-powerd.service
systemctl stop nvidia-persistenced}.service
```

Verify the NVIDIA modules are loaded:
```
lsmod | grep -E "^(Module|nvidia)"
```

And verify that they are not in-use. The `lsmod` output should look like this:
<blockquote>
Module                  Size  Used by<br/>
nvidia_uvm           4653056  0<br/>
nvidia_drm             98304  0<br/>
nvidia_modeset       1413120  1 nvidia_drm<br/>
nvidia               8024064  2 nvidia_uvm,nvidia_modeset<br/>
</blockquote>

Note: The usage count is shown in the 3rd column. It should be exactly 2 for "nvidia", 1 for "nvidia_modeset" and 0 for "nvidia_drm" and "nvidia_uvm". If the numbers differ something is using the dGPU.

You can also use `lsof` to see which application is accessing the card:
```
# note: your dGPU might have a different card and renderer index
lsof /dev/dri/card1 /dev/dri/renderD129 /dev/nvidia*
```

Verify the iGPU and dGPU are usable in Linux:
```
# iGPU
glxinfo -B | grep "OpenGL renderer string"
vulkaninfo | grep --no-group-separator -A 8 VkPhysicalDeviceProperties
glxgears
vkcube

# dGPU
nvidia-smi
nvidia-run glxinfo -B | grep "OpenGL renderer string"
nvidia-run vulkaninfo | grep --no-group-separator -A 8 VkPhysicalDeviceProperties
nvidia-run glxgears
nvidia-run vkcube
```

You can always use `lsof` to see which applications are accessing the dGPU.

And finally, start your VM:
```
virsh start ${VM_NAME}
```

# Final words

This guide is not perfect.

Some applications may be using the dGPU without you knowing about it. For instance if you run [mpv](https://mpv.io/) with `--hwdec=auto-safe` it will use the dGPU for decoding which exactly what you want. You may just not be aware of it.

The libvirt hook was written to handle this. If you try to start the VM with the dGPU still beeing in-use it will fail gracefully and tell you which application is using it.

After using your system for a while you may come across some applications that use the dGPU. You then have to choose if you want to try to prevent them from using the dGPU at all or you just accept the fact that you can't start your VM if that application is running.

As for how to figure out how to prevent an application from using the dGPU I have no good answer. It really depends on the application and in extreme cases it can only be done by patching the source code of that application or using some LD_PRELOAD hack.


This guide only describes an intermediate solution.

While this is a huge step up from not using your dGPU in Linux at all, using it as an offloading renderer has it's drawbacks (as menioned in [Requirements & limitations](#requirements--limitations)). But for now this is as good as it gets. Maybe in a few years we'll have decent gpu hot-unplugging support or maybe the GPU vendors finally decide to grant their consumer GPUs at least rudimentary virtualization support (Intel seems to be heading in vagily that direction).
