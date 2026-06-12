# Orange Pi Zero 3W GPU/VPU image builder

The Orange Pi Zero 3W's Allwinner A733 has an Imagination PowerVR BXM-4-64 GPU and an Allwinner Cedar video engine, but Orange Pi's stock images ship no userspace for either, leaving them unusable. Radxa's Cubie A7S uses the same A733 silicon and ships the missing PowerVR and Cedar userspace in its Debian image.

These scripts build an Orange Pi Zero 3W Ubuntu image with a working GPU and VPU by grafting that userspace out of a Radxa image you supply locally, then rebuilding the PowerVR kernel module against Orange Pi's kernel with DKMS. The bootloader, kernel, DTB, Wi-Fi, Bluetooth, desktop, and default user all stay exactly as Orange Pi shipped them.

No proprietary binaries live in this repository. The GPU/VPU files come straight out of the Radxa image on your own machine at build time. There's a longer write-up in [this XDA article](https://www.xda-developers.com/orange-pi-zero-3w-beats-raspberry-pi-5-cant-use-half-hardware/).

## What works

On the tested image pair, the result has:

- PowerVR Vulkan/OpenGL ES/OpenCL userspace
- `pvrsrvkm` rebuilt for Orange Pi's `6.6.98-sun60iw2` kernel, with PowerVR firmware and the Vulkan ICD in place
- Allwinner CedarC/libcedarc userspace
- Hardware video decode/encode through GStreamer's OMX plugins
- Hardware-accelerated WebGL/WebGPU in Chromium via ANGLE-on-Vulkan, installed from the saiarcot895 PPA (Ubuntu's repository `chromium-browser` is only a Snap transition package that's unsuitable for this userspace). Two test launchers are added to the desktop. The first is a WebGL aquarium and the second is a `chrome://gpu` shortcut
- Orange Pi's original bootloader, kernel, DTB, Wi-Fi, Bluetooth, desktop, and default user are all untouched

## Source images

Download both images, then verify each against the checksum the vendor actually publishes. The two cover different files: OrangePi checksums the extracted `.img`, while Radxa checksums the downloaded `.xz`. Each value below is the one that vendor signs, so they're verified at different stages.

**Orange Pi Zero 3W: Ubuntu Jammy XFCE desktop, kernel 6.6.98**
- Download: <http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-Zero-3W.html>
- Archive `Orangepizero3w_1.0.0_ubuntu_jammy_desktop_xfce_linux6.6.98.7z`, which extracts to `Orangepizero3w_1.0.0_ubuntu_jammy_desktop_xfce_linux6.6.98.img`
- SHA256 of the extracted `.img`: `af6697c4f158f63ffdf55f5a17453ef3b9b2895f35b2891e6090d28f65faf264`

**Radxa Cubie A7S: Bullseye KDE, kernel 5.15.147**
- Download: <https://docs.radxa.com/en/cubie/a7s/download>
- Archive `radxa-a733_bullseye_kde_r2.output_512.img.xz`, which extracts to `radxa-a733_bullseye_kde_r2.output_512.img`
- SHA256 of the downloaded `.xz`: `1b5604fed61647ab1b510f24af5968477e8a7a361430aa0864efbed7b5fe6ca2`

```bash
# OPi: verify the .img after extracting the .7z
shasum -a 256 Orangepizero3w_1.0.0_ubuntu_jammy_desktop_xfce_linux6.6.98.img

# Radxa: verify the downloaded archive before extracting
shasum -a 256 radxa-a733_bullseye_kde_r2.output_512.img.xz
```

I only tested this exact image pair. The scripts read partition offsets from each image's own partition table, but the package names and paths are hardcoded to those two BSP images, so a different or newer image may need small path/package-name fixes. When an input doesn't match, the scripts fail at the missing file rather than producing a broken image.

## Build a flashable image

Install Docker, then run this from the repository root:

```bash
docker run --rm --privileged -v "$(pwd):/work" -w /work \
  debian:bookworm-slim bash /work/scripts/build.sh
```

The output is `hybrid-opi66-with-radxa-gpu-vpu.img`. Flash it with your usual imaging tool, or with `dd`.

To rebuild the PowerVR module, `build.sh` runs DKMS against the kernel headers the stock Orange Pi image already ships at `/opt/linux-headers-current-sun60iw2_*.deb`. The build depends on that `.deb` being present there; the pinned image carries it.

macOS:

```bash
diskutil list
sudo diskutil unmountDisk /dev/diskN
sudo dd if=hybrid-opi66-with-radxa-gpu-vpu.img of=/dev/rdiskN bs=4m status=progress
sudo diskutil eject /dev/diskN
```

Login is unchanged: `orangepi`/`orangepi`.

## Userspace tarballs only

To pull just the PowerVR and VPU userspace out of the Radxa image (for an Orange Pi 6.6 system you've already set up) skip the full build:

```bash
docker run --rm --privileged -v "$(pwd):/work" -w /work \
  debian:bookworm-slim bash /work/scripts/make-tarball.sh
```

That produces `pvr-userspace.tar.gz` and `vpu-userspace.tar.gz`. Copy them to the board and extract as root:

```bash
sudo tar xzpf /tmp/pvr-userspace.tar.gz -C /
sudo tar xzpf /tmp/vpu-userspace.tar.gz -C /
sudo ldconfig
sudo udevadm control --reload-rules && sudo udevadm trigger
sudo reboot
```

This does not build the PowerVR kernel module. Use the full image builder if you want DKMS handled for you.

## Scripts

- `scripts/build.sh`: builds the full hybrid image
- `scripts/make-tarball.sh`: pulls the PowerVR/VPU userspace tarballs from the Radxa image

## License and redistribution

The scripts and documentation here are MIT licensed. The Radxa, Imagination Technologies, and Allwinner binaries the scripts copy are not. They come from the Radxa image you provide locally and stay under their original licenses.

Don't commit source images, generated hybrid images, or the userspace tarballs.
