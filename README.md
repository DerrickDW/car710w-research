car710w-research

Reverse engineering the firmware of my Jensen CAR710W head unit to locate and modify UI assets such as boot animations and backgrounds.

2026-03-08 — First Firmware Dive

Target: Jensen CAR710W firmware
Goal: Locate boot animation or UI background assets.

Tools Used

WSL

binwalk

unsquashfs

ffprobe

ffmpeg

Firmware Acquisition

Downloaded from the manufacturer website:

ISPBOOOT.BIN

ISPBOOOT_UPDATE_U.BIN

Firmware Scan

Initial scan:

binwalk ISPBOOOT.BIN

Key results:

8577024    0x82E000   Squashfs filesystem
10768384   0xA45000   Squashfs filesystem
11022336   0xA83000   Squashfs filesystem
46112768   0x2BFA000  Squashfs filesystem

Filesystem references discovered inside the firmware:

/dev/vc/0
/sys/module/sp_spinand/parameters/hwcfg
/sys/kernel/debug/gc/galcore_trace
Notable Discoveries

U-Boot images

Linux kernel 4.9.217

Raw NAND partitions embedded in firmware

Firmware Partition Layout

Partitions discovered from flashing scripts:

uboot
env
ecos
kernel
rootfs
opt
spsdk
spapp
nvm
pq
logo
tcon
runtime_cfg
vi
isp_logo
vendordata
pat_logo
version_info
vd_restore
anm_logo
userdata

These partitions are programmed directly to NAND during firmware flashing.

Filesystem Extraction
unsquashfs rootfs.squashfs

Example extracted structure:

/bin
/etc
/lib
/usr
/init.rc
/init.gui.rc

This confirms the system uses:

Linux userspace

BusyBox utilities

DirectFB graphics stack

Attempt #1 — Background Images

First attempt was to locate UI background images.

Searching firmware for OGOL image blobs:

grep -oba OGOL ISPBOOOT.BIN

Firmware contains a PART container with update images.

Structure
PART
 ├── isp_begin.bin
 ├── isp_ok.bin
 └── isp_failed.bin

Images use a proprietary OGOL format.

OGOL format
4 bytes  magic = OGOL
32 byte  header
raw pixel data
resolution: 1024x600
pixel order: ABGR
Decoded Update Images

Update screen images successfully decoded.

Updating screen
<img width="1024" height="600" src="https://github.com/user-attachments/assets/ffb12f84-e68b-4a9c-bdc2-02c1a532dc41" />
Burn success screen
<img width="1024" height="600" src="https://github.com/user-attachments/assets/89adbc38-3f03-4c4f-86c1-3636a58b1907" />

These appear only during firmware flashing, not during normal operation.

Logo Region Investigation

Multiple OGOL blobs discovered:

54064240
56521872
58979504
59597968
60213424
60828880
61444336

Some decode partially but appear to contain layered assets rather than full background images.

Further investigation required.

Boot Animation Discovery (anm_logo)

Major discovery of the session.

Extraction:

tail -c +$((62063664 + 1)) ISPBOOOT.BIN > anm_logo.mkv

Result:

Matroska video file
Video properties
Container: MKV
Codec: MPEG4 (XviD)
Resolution: 1072x600
Pixel format: yuv420p
Framerate: 24 FPS
Duration: 4 seconds
Bitrate: ~1220 kb/s

Boot animation preview:

This confirms the head unit plays a standard MKV video during boot.

Boot Animation Modding

Boot animation can be replaced provided the new video:

stays under 0x95000 bytes

uses compatible codec

uses similar resolution

stays around 4 seconds

This makes boot animation customization relatively straightforward.

Next step is rebuilding the firmware and testing a custom animation.

UI Rendering Code

Binary analysis revealed UI rendering functions:

Draw_Background_From_Nand
Set_Background_Transparency
Stop_Animation_Logo

Located in:

libpfc_service.so
libsppfc.so

This strongly suggests UI backgrounds are read directly from NAND.

Possible partitions involved:

logo
pat_logo
runtime_cfg
vendordata

Further investigation required.

CarPlay Subsystem

CarPlay related assets discovered:

/etc/carplay/oemIcon.png
/etc/carplay/oemIcon120.png

CarPlay services present:

apple_carplay_server
libspcarplay.so

Potential targets for customization:

CarPlay background

OEM icon

theme assets

Key Takeaways

The firmware is surprisingly mod-friendly.

Characteristics:

Linux based

SquashFS root filesystem

standard MKV boot animation

raw framebuffer images

no obvious encryption

Confirmed customizable components:

boot animation

update images

CarPlay icons

possible UI backgrounds

Session Time

Total time spent: ~1.5 hours
