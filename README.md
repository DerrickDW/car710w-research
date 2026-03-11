# car710w-research
Reverse engineering the firmware of my Jensen CAR710W head unit to locate and modifiy UI assets such as boot animations and backgrounds.
## 2026-03-08 - First dive into the firmware
Target: Jensen CAR710W firmware
Goal: Locate boot animation asset or background assets, whatever else I can see.

## Tools Used
-wsl
-binwalk
-unsquashfs
-ffprobe
-ffmeg

## Firmware Aquisition
-Downloaed from manufacturer's website
-ISPBOOOT.BIN
-ISPBOOOT_UPDATE_U.BIN

## Commands Executed

### Firmware scan

```bash
binwalk ISPBOOOT.bin```
gave us
-8577024       0x82E000        Squashfs filesystem, little endian, version 4.0, compression:gzip (non-standard type definition), size: 2187539 bytes, 501 inodes, blocksize: 32768 bytes, created: 2022-11-17 11:11:23
10768384      0xA45000        Squashfs filesystem, little endian, version 4.0, compression:gzip (non-standard type definition), size: 250942 bytes, 2 inodes, blocksize: 32768 bytes, created: 2022-11-17 11:11:23
11022336      0xA83000        Squashfs filesystem, little endian, version 4.0, compression:gzip (non-standard type definition), size: 35088130 bytes, 746 inodes, blocksize: 32768 bytes, created: 2022-11-17 11:11:24
46112768      0x2BFA000       Squashfs filesystem, little endian, version 4.0, compression:gzip (non-standard type definition), size: 7848662 bytes, 107 inodes, blocksize: 32768 bytes, created: 2022-11-17 11:11:23
as well as a bit of the file structure
7680276       0x753114        Unix path: /dev/vc/0
7761726       0x766F3E        Unix path: /sys/module/sp_spinand/parameters/hwcfg
7836400       0x7792F0        Unix path: /sys/kernel/debug/gc/galcore_trace
Notables
-U-Boot images
-Linux kernal (4.9.217)
-Raw partitions stored inside the firmware image
All important partitions
-uboot
-env
-ecos
-kernel
-rootfs
-opt
-spsdk
-spapp
-nvm
-pq
-logo
-tcon
-runtime_cfg
-vi
-isp_logo
-vendordata
-pat_logo
-verison_info
-vd_restore
-anm_logo => BIG ONE
-userdata
These partitions are programmed directly to NAND using the firmware flashing scripts
```unsqaushfs rootfs.squashfs```
Example rootfs strutcture:
/bin
/etc
/lib
/usr
/init.rc
/init.gui.rc
This confirms:
Linux userspace enviroment
Busybox based
custom display stack using DirectFB

## Before Chasing boot animation I first tried to get to the backgrounds
carved out isp_logo
```grep -oba OGOL ISPBOOOT.BIN```
firmware contains a PART container holding update images
Structure:
PART/
/isp_begin.bin
/isp_ok.bin
/isp_failed.bin
Images inside use a propietary OGOL format.
-4 bytes magic = OGOL
-32 byte header
-raw pixel data
-Resolution 1024xx600
-Pixel order ABGR

## Decoded images included:

Updating...
<img width="1024" height="600" alt="ogol_56521872_hdr0020_1024x600_RGBA" src="https://github.com/user-attachments/assets/ffb12f84-e68b-4a9c-bdc2-02c1a532dc41" />
Burn success, welcome! 
These were not background image<img width="1024" height="600" alt="ogol_54064240_hdr0020_1024x600_RGBA" src="https://github.com/user-attachments/assets/89adbc38-3f03-4c4f-86c1-3636a58b1907" />
s but images that flash while updating bummer
## Main Logo Regin
-Multiple OGOL blobs exist in the logo region though
-offsets discovered:
54064240
56521872
58979504
59597968
60213424
60828880
61444336

Some decode partially but appear to contain layered or tiled assets rather than full images.

Further investigation required.

## Boot Animation (anm_logo) = The Big Discovery of the night.
```tail -c +$((62063664 + 1)) ISPBOOOT.BIN > anm_logo.mkv```
Returned
anm_logo_0x95000.bin: Matroska data

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             EBML file, Matroska data
Video properties
Container: Matroska (.mkv)
resolution 1072x600 => odd finding that the images were a different resolution
pixel format: yuv420p
framrate: 24FPS
duration 4 seconds
bitrate: ~1220kb/s

We found the boot animation that fires on boot!
![anm_logo](https://github.com/user-attachments/assets/5b76bc69-097c-4d73-bfe5-ab32db67dde3)
Because it uses a standard container and codec, it can be replaced with a custom animation provided the replacement:

remains ≤ 0x95000 bytes

uses compatible codec/resolution

duration roughly similar

This makes boot animation customization straightforward.

Next step here is to replace it and rebuild firmware and throw it on the headunit

## Relevant UI Rendering Code

Binary analysis revealed background rendering functions:

Draw_Background_From_Nand
Set_Background_Transparency
Stop_Animation_Logo

These are located in:

libpfc_service.so
libsppfc.so

These strongly suggest the UI background is read from a NAND partition rather than embedded resources.

Potential partitions involved:

logo
pat_logo
runtime_cfg
vendordata

Further investigation required.

## CarPlay Subsystem

CarPlay related files discovered:

/etc/carplay/oemIcon.png
/etc/carplay/oemIcon120.png

CarPlay services present in binaries:

apple_carplay_server
libspcarplay.so

These may control:

CarPlay background

OEM icon

theme assets

Possibility exists for replacing CarPlay UI resources.

## Key Takeaways

This firmware is surprisingly hackable:

Linux based system

SquashFS rootfs

Standard video container for boot animation

Raw framebuffer images

No obvious encryption on assets

Confirmed customizable components:

Boot animation
Firmware update screens
CarPlay icon assets
Potential UI backgrounds

## Total time spent this night: 1.5HR
