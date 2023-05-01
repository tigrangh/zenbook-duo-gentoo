# zenbook-duo-gentoo
## UX482

## Features
1. UEFI
1. ZFS native encrypted root
1. LUKS encrypted swap

## References
https://blog.paragoncyber.io/docs/gentoo-zfs/setup/  
https://wiki.gentoo.org/wiki/Handbook:AMD64  

https://www.reddit.com/r/Gentoo/comments/s2sww0/elan1200_touchpad_problems/  
https://pclosmag.com/html/Issues/201108/page01.html  
https://github.com/s-light/ASUS-ZenBook-Pro-Duo-UX581GV/blob/master/research_touchpen.md  
https://forums.gentoo.org/viewtopic-t-1061944-start-0.html  
https://www.kernel.org/doc/html/v5.0/admin-guide/pm/cpuidle.html  
https://metebalci.com/blog/a-minimum-complete-tutorial-of-cpu-power-management-c-states-and-p-states/  
https://www.reddit.com/r/linuxhardware/comments/da4zy9/does_anyone_know_how_the_software_support_will_be/  
https://davejansen.com/asus-zenbook-duo-and-fedora-linux/  
https://askubuntu.com/questions/762643/change-brightness-on-asus-laptop  
https://github.com/lakinduakash/asus-screenpad-control  
https://github.com/Plippo/asus-wmi-screenpad  
https://wiki.gentoo.org/wiki/Webcam  

## Prepare the system disk

### Create the partitions
```
localhost ~ # parted -a optimal /dev/nvme0n1
```

```
(parted) mklabel gpt
(parted) mkpart esp fat32 1MIB 513MIB
(parted) mkpart swap linux-swap 513MIB 66049MIB
(parted) mkpart rootfs ext4 66049MIB 100%
(parted) set 1 boot on
(parted) print
```

```
Model: WD Green SN350 2TB (nvme)
Disk /dev/nvme0n1: 2000GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system     Name    Flags
 1      1049kB  538MB   537MB   fat32           esp     boot, esp
 2      538MB   69.3GB  68.7GB  linux-swap(v1)  swap    swap
 3      69.3GB  2000GB  1931GB  ext4            rootfs
```

### Format the drives

```
mkfs.vfat -F32 /dev/nvme0n1p1
```

```
mkswap -f /dev/nvme0n1p2
```
> **Note**  
> _Before creating the zfs pool, find the disk by id_

```
localhost ~ # ls -l /dev/disk/by-id | grep p3
lrwxrwxrwx 1 root root 15 Apr 21 07:39 nvme-WD_Green_SN350_2TB_23032S801729-part3 -> ../../nvme0n1p3
```

```
zpool create -f -o ashift=12 -O mountpoint=none -O relatime=on -O atime=on -O acltype=posixacl -O xattr=sa -O aclinherit=passthrough -O compression=lz4 -O encryption=on -O keyformat=passphrase -O keylocation=prompt -m none zentoo /dev/disk/by-id/nvme-WD_Green_SN350_2TB_23032S801729-part3
```
> **Warning**  
> _Before booting to the new OS itself, will need to set the mountpoint to /_

```
zfs create -o mountpoint=/mnt/gentoo zentoo/root
```

_Set the root flag_
```
zpool set bootfs=zentoo/root zentoo
```

## Continue with the AMD64 handbook

_Start with the following make.conf customizations_
```
USE="X elogind bluetooth"
INPUT_DEVICES="libinput"
VIDEO_CARDS="intel i965 iris dummy alsa"
```

> **Note**  
> dummy videocard is optional: I prefer to have the dummy video output feature.

Follow the official Gentoo AMD64 Handbook, until the stage3 is ready

## When the stage3 is extracted

It is a good moment to install some essential packges

```
emerge -av bash-completion efibootmgr dracut vim cpuid2cpuflags app-misc/mc dosfstools usbutils pciutils lshw wpa_supplicant net-wireless/iw net-dns/bind-tools
```

* __cpuid2cpuflags__ should be the first package installed before anything else, to set the native CPU flags for the system
* __dracut__ and __efibootmgr__ are a part of this guide. Those are used to create the iniramfs to decode the ZFS filesystem, and for the boot configuration, respectively.
* The rest are essential tools for any gentoo setup of mine.

## Continue with the handbook

__Until the kernel configuration stage__

Stop here to make additional kernel configurations

#### To have the WD Green NVME device
```
CONFIG_VMD=y
```

#### Choose Intel idle driver
```
CONFIG_INTEL_IDLE=y
```

#### Embed the wireless and graphics driver blobs into the kernel
```
CONFIG_EXTRA_FIRMWARE="i915/tgl_dmc_ver2_12.bin iwlwifi-cc-a0-72.ucode"
CONFIG_EXTRA_FIRMWARE_DIR="/lib/firmware"
```

#### ZFS is essential for this guide
https://wiki.gentoo.org/wiki/ZFS  

#### Some essential changes for full hardware capability

https://wiki.gentoo.org/wiki/NVMe  
https://wiki.gentoo.org/wiki/USB/Guide  
https://wiki.gentoo.org/wiki/ALSA   
https://wiki.gentoo.org/wiki/Iwlwifi  
https://wiki.gentoo.org/wiki/ACPI#Kernel  
https://wiki.gentoo.org/wiki/Power_management/Guide  
check https://wiki.gentoo.org/wiki/Power_management/Processor  
https://wiki.gentoo.org/wiki/PowerTOP  
https://wiki.gentoo.org/wiki/Intel  

```
Device Drivers  --->
    [*] PCI support  --->
            [*] PCI Express ASPM support
                    Default ASPM Policy (Power Supersave)  --->
```

#### And some more

lpss  
```
CONFIG_X86_INTEL_LPSS=y
CONFIG_MFD_INTEL_LPSS=y
CONFIG_MFD_INTEL_LPSS_ACPI=y
CONFIG_MFD_INTEL_LPSS_PCI=y
```
intel pinctrl  
```
PINCTRL=y
PINCTRL_ITNEL=y
PINCTRL_TIGERLAKE=y
```

```
CONFIG_EDAC=y
CONFIG_EDAC_IGEN6=m
CONFIG_RAS=y
```

asus wmi notebook  
```
ASUS_WMI=y
ASUS_NB_WMI=y
```

```
ACPI_TAD=y

CONFIG_HID_MULTITOUCH=y
CONFIG_I2C_HID_ACPI=y
CONFIG_I2C_HID_CORE=y
```

touchpad and touchscreen related  
```
CONFIG_INPUT_MOUSEDEV=y
CONFIG_MOUSE_PS2_ELANTECH=y
CONFIG_MOUSE_PS2_ELANTECH_SMBUS=y
CONFIG_MOUSE_ELAN_I2C=y
CONFIG_MOUSE_ELAN_I2C_I2C=y
CONFIG_MOUSE_ELAN_I2C_SMBUS=y
CONFIG_MOUSE_GPIO=y
CONFIG_MOUSE_SYNAPTICS_USB=m
CONFIG_TOUCHSCREEN_EKTF2127=y
CONFIG_TOUCHSCREEN_ELAN=y
CONFIG_I2C_CCGX_UCSI=y
CONFIG_I2C_ISCH=y
CONFIG_I2C_ISMT=y
CONFIG_I2C_DESIGNWARE_CORE=y
CONFIG_I2C_DESIGNWARE_PLATFORM=y
CONFIG_I2C_DESIGNWARE_BAYTRAIL=y
CONFIG_I2C_DESIGNWARE_PCI=y
CONFIG_LPC_SCH=y
CONFIG_HID_ELAN=y
```

sound related  
https://forums.gentoo.org/viewtopic-t-1144224-start-0.html  
https://forums.gentoo.org/viewtopic-p-8704599.html?sid=fac3d5c74b23a31a1f41bb6b0e1d97c9  
https://forums.gentoo.org/viewtopic-p-8688283.html  
https://forums.gentoo.org/viewtopic-t-1048524-start-0.html  

```
CONFIG_EXTRA_FIRMWARE="i915/tgl_dmc_ver2_12.bin iwlwifi-cc-a0-72.ucode intel/sof/sof-tgl.ri intel/sof-tplg/sof-hda-generic-2ch.tplg"
CONFIG_REGMAP_I2C=y
CONFIG_SND_PCM_ELD=y
CONFIG_SND_COMPRESS_OFFLOAD=y
CONFIG_SND_HDA_RECONFIG=y
CONFIG_SND_HDA_EXT_CORE=y
CONFIG_SND_SOC=y
CONFIG_SND_SOC_COMPRESS=y
CONFIG_SND_SOC_TOPOLOGY=y
CONFIG_SND_SOC_ACPI=y
CONFIG_SND_SOC_INTEL_SST_TOPLEVEL=y
CONFIG_SND_SST_ATOM_HIFI2_PLATFORM=y
CONFIG_SND_SST_ATOM_HIFI2_PLATFORM_ACPI=y
CONFIG_SND_SOC_ACPI_INTEL_MATCH=y
CONFIG_SND_SOC_INTEL_MACH=y
CONFIG_SND_SOC_INTEL_HDA_DSP_COMMON=y
CONFIG_SND_SOC_INTEL_SKL_HDA_DSP_GENERIC_MACH=y
CONFIG_SND_SOC_SOF_TOPLEVEL=y
CONFIG_SND_SOC_SOF_PCI_DEV=y
CONFIG_SND_SOC_SOF_PCI=y
CONFIG_SND_SOC_SOF_DEBUG_PROBES=y
CONFIG_SND_SOC_SOF_CLIENT=y
CONFIG_SND_SOC_SOF=y
CONFIG_SND_SOC_SOF_PROBE_WORK_QUEUE=y
CONFIG_SND_SOC_SOF_IPC3=y
CONFIG_SND_SOC_SOF_INTEL_IPC4=y
CONFIG_SND_SOC_SOF_INTEL_TOPLEVEL=y
CONFIG_SND_SOC_SOF_INTEL_COMMON=y
CONFIG_SND_SOC_SOF_INTEL_TGL=y
CONFIG_SND_SOC_SOF_TIGERLAKE=y
CONFIG_SND_SOC_SOF_HDA_COMMON=y
CONFIG_SND_SOC_SOF_HDA_LINK=y
CONFIG_SND_SOC_SOF_HDA_AUDIO_CODEC=y
CONFIG_SND_SOC_SOF_HDA_LINK_BASELINE=y
CONFIG_SND_SOC_SOF_HDA=y
CONFIG_SND_SOC_SOF_HDA_PROBES=y
CONFIG_SND_SOC_SOF_INTEL_SOUNDWIRE_LINK_BASELINE=y
CONFIG_SND_SOC_SOF_XTENSA=y
CONFIG_SND_SOC_I2C_AND_SPI=y
CONFIG_SND_SOC_DMIC=y
CONFIG_SND_SOC_HDAC_HDMI=y
CONFIG_SND_SOC_HDAC_HDA=y
```

bluetooth related
https://wiki.gentoo.org/wiki/Bluetooth_headset#ALSA_.2B_Bluez_5  
https://wiki.gentoo.org/wiki/Bluetooth  

```
CONFIG_BT_RFCOMM=m
CONFIG_INPUT_UINPUT=m
```

SD card reader
https://forums.gentoo.org/viewtopic-t-1006246-start-0.html  

```
CONFIG_MISC_RTSX=m
CONFIG_MISC_RTSX_USB=m
CONFIG_MMC=m
CONFIG_MMC_BLOCK=m
CONFIG_MMC_BLOCK_MINORS=8
CONFIG_MMC_REALTEK_USB=m
```

#### I prefer to have more or less capable kernel

https://wiki.gentoo.org/wiki/Nfs-utils  
https://wiki.gentoo.org/wiki/Samba#Kernel  
https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide#Compressed_kernel_modules  
https://wiki.gentoo.org/wiki/QEMU  
https://wiki.gentoo.org/wiki/Bluetooth  
https://wiki.gentoo.org/wiki/Docker 
https://docs.strongswan.org/docs/5.9/install/kernelModules.html  

## Install ZFS to the new system

```
emerge -av zfs

rc-update add zfs-import boot
rc-update add zfs-mount boot
rc-update add zfs-share default
rc-update add zfs-zed default
```


Follow the handbook to install the kernel to /boot

## When done, 

### we create the initramfs
```
dracut -H --kver 6.1.19-gentoo --force
```

### Prepare EFI boot files

Granted that /boot is mounted to the UEFI partition
```
ls -l /boot/EFI/gentoo/
total 26392
-rwxr-xr-x 1 root root 11407542 Apr 21 12:53 initramfs.img
-rwxr-xr-x 1 root root 15609888 Apr 21 12:52 vmlinuz.efi
```
```
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Gentoo 0" --loader "\EFI\gentoo\vmlinuz.efi" --unicode "root=ZFS=zentoo/root ro initrd=\EFI\gentoo\initramfs.img"
```

### Finally follow the guide till the end

Don't forget to do the following before rebooting
```
zfs set mountpoint=/ zentoo/root
zpool export zentoo
```

## Appendix

### xinitrc

```
cat /etc/X11/xinit/xinitrc
xrandr --output eDP-1 --auto --pos 0x0 --primary
xrandr --output DP-3 --auto --pos 0x0 --scale 2x2.12
xrandr --dpi 168
xrandr --setmonitor Virt1 1920/309x1080/174+0+0 DP-3
xrandr --setmonitor Virt2 1920/309x1080/174+1920+0 none

xinput disable "ELAN9008:00 04F3:2D55"

export GDK_DPI_SCALE=1.75# 1.5

exec dbus-launch openbox-session
```

```
cat /usr/share/X11/xorg.conf.d/41-inputs.conf 
Section "InputClass"
	Identifier "ELAN9008:00 04F3:2D55"
	# automatically matches for "ELAN9008:00 04F3:2D55 Stylus Pen (0)" as well
	MatchProduct "ELAN9008:00 04F3:2D55"
        Driver "libinput"
        Option "TransformationMatrix" "0.5 0 0 0 0.989192 0 0 0 1"
EndSection

Section "InputClass"
        Identifier "this is terrible (AVRCP)"
        MatchProduct "this is terrible (AVRCP)"
        Driver "libinput"
        Option "Ignore" "true"
EndSection
```

### screens brightness control related

```
cat /etc/udev/rules.d/99-asus.rules 
ACTION=="add", SUBSYSTEM=="leds", KERNEL=="asus::screenpad", RUN+="/bin/chmod a+w /sys/class/leds/%k/brightness"
ACTION=="add", SUBSYSTEM=="backlight", KERNEL=="intel_backlight", RUN+="/bin/chmod a+w /sys/class/backlight/%k/brightness"
```

### bluetooth headset
```
cat /etc/conf.d/bluealsa 
# Config file for /etc/init.d/bluealsa

# Allow additional options to be set
BLUEALSA_CONF="-i hci0 -p a2dp-sink -p a2dp-source -p hfp-ag -p hfp-hf -p hsp-ag -p hsp-hs"
```

```
cat ~/.asoundrc
pcm.!default built_in
#pcm.!default blue_head
#pcm.!default blue_speaker

pcm.blue_speaker {
  hint.show on
  hint.description "Bluetooth Music"
  
  type asym
  playback.pcm {
    type plug
    slave.pcm "bluealsa:PROFILE=a2dp"
  }

  capture.pcm {
    type plug
    slave.pcm "hw:sofhdadsp,7"
  }
}

pcm.blue_head {
  hint.show on
  hint.description "Bluetooth Headset"

  type asym
  playback.pcm {
    type plug
    slave.pcm "bluealsa:PROFILE=sco"
  }

  capture.pcm {
    type plug
    slave.pcm "bluealsa:PROFILE=sco"
  }
}

pcm.built_in {
  hint.show on
  hint.description "Built in"

  type asym
  playback.pcm {
    type plug
    slave.pcm {
      type dmix
      ipc_key 8626696
      ipc_gid 18
      ipc_perm 0660
      slave.pcm "hw:sofhdadsp,0"
    }
  }

  capture.pcm {
    type plug
    slave.pcm "hw:sofhdadsp,7"
  }
}
```

### openbox hotkeys

```
  <keybind key="XF86MonBrightnessDown">
    <action name="Execute">
    <command>
    sh -c 'BRIGHTNESS=$(cat /sys/class/backlight/intel_backlight/brightness); BRIGHTNESS=$((BRIGHTNESS - 500)); BRIGHTNESS=$((BRIGHTNESS > 0 ? BRIGHTNESS : 0)); echo "$BRIGHTNESS" > /sys/class/backlight/intel_backlight/brightness'
    </command>
    </action>
  </keybind>
  <keybind key="XF86MonBrightnessUp">
    <action name="Execute">
    <command>
    sh -c 'BRIGHTNESS=$(cat /sys/class/backlight/intel_backlight/brightness); BRIGHTNESS=$((BRIGHTNESS + 500)); BRIGHTNESS=$((19200 > BRIGHTNESS ? BRIGHTNESS : 19200)); echo "$BRIGHTNESS" > /sys/class/backlight/intel_backlight/brightness'
    </command>
    </action>
  </keybind>
  <keybind key="W-F4">
    <action name="Execute">
    <command>
    sh -c 'BRIGHTNESS=$(cat /sys/class/leds/asus::screenpad/brightness); BRIGHTNESS=$((BRIGHTNESS - 10)); BRIGHTNESS=$((BRIGHTNESS > 0 ? BRIGHTNESS : 0)); echo "$BRIGHTNESS" > /sys/class/leds/asus::screenpad/brightness'
    </command>
    </action>
  </keybind>
  <keybind key="W-F5">
    <action name="Execute">
    <command>
    sh -c 'BRIGHTNESS=$(cat /sys/class/leds/asus::screenpad/brightness); BRIGHTNESS=$((BRIGHTNESS + 10)); BRIGHTNESS=$((255 > BRIGHTNESS ? BRIGHTNESS : 255)); echo "$BRIGHTNESS" > /sys/class/leds/asus::screenpad/brightness'
    </command>
    </action>
  </keybind>
<keybind key="Print">
    <action name="Execute">
    <command>
    sh -c 'ENABLED=$(xinput --list-props "ELAN9008:00 04F3:2D55" | grep "Device Enabled" | awk "{print \$4}"); if [ "$ENABLED" == "1" ]; then xinput disable "ELAN9008:00 04F3:2D55"; else xinput enable "ELAN9008:00 04F3:2D55"; fi'
    </command>
    </action>
  </keybind>
  <keybind key="XF86Launch6">
    <action name="Execute">
    <command>
    sh -c 'ZEROPOS=$(xrandr | grep eDP-1 | awk "{print \$4}" | grep -o "+0+0"); if [ -z "$ZEROPOS" ]; then xrandr --output eDP-1 --auto --pos 0x0; xdotool mousemove 960 540; xinput set-prop "ELAN9008:00 04F3:2D55" "Coordinate Transformation Matrix" 0.5 0 0 0 0.989192 0 0 0 1; xinput set-prop "ELAN9008:00 04F3:2D55 Stylus Pen (0)" "Coordinate Transformation Matrix" 0.5 0 0 0 0.989192 0 0 0 1; else xrandr --output eDP-1 --auto --pos 1920x0; xdotool mousemove 2880 540; xinput set-prop "ELAN9008:00 04F3:2D55" "Coordinate Transformation Matrix" 0.5 0 0.5 0 0.989192 0 0 0 1; xinput set-prop "ELAN9008:00 04F3:2D55 Stylus Pen (0)" "Coordinate Transformation Matrix" 0.5 0 0.5 0 0.989192 0 0 0 1; fi; echo $([[ -z "$ZEROPOS" ]])'
    </command>
    </action>
  </keybind>
  <keybind key="XF86Launch7">
    <action name="Execute">
    <command>
    sh -c 'ENABLED=$(xinput --list-props "AT Translated Set 2 keyboard" | grep "Device Enabled" | awk "{print \$4}"); if [ "$ENABLED" == "1" ]; then xinput disable "AT Translated Set 2 keyboard"; xinput disable "ELAN1200:00 04F3:3168 Touchpad"; xinput disable "ELAN9009:00 04F3:2C1B"; else xinput enable "AT Translated Set 2 keyboard"; xinput enable "ELAN1200:00 04F3:3168 Touchpad"; xinput enable "ELAN9009:00 04F3:2C1B"; fi'
    </command>
    </action>
  </keybind>
```
