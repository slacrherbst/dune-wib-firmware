# DUNE WIB Firmware

Central repository for development of DUNE WIB firmware and software.

  * [Development](#development)
    + [Getting Started](#getting-started)
    + [Organization](#organization)
    + [Version Control](#version-control)
  * [Building from Scratch](#building-from-scratch)
    +  [Generate bitstream](#generate-bitstream)
    +  [Export hardware definition](#export-hardware-definition)
    +  [Create PetaLinux images](#create-petalinux-images)
    +  [Creating a bootable SD image](#creating-a-bootable-sd-image)
  * [Test drive the linux system with QEMU](#test-drive-the-linux-system-with-qemu)


## Development

### Getting Started

1. Install Vivado 2019.1
2. Clone this repository
3. Source `settings64.sh` from the Vivado install
4. Run `./init.sh` to create `DUNE_WIB` Vivado project

### Organization

* Project source (Verilog or VHDL) organized under `src/`
* Imported IPs should go in `ip/` (setup as Vivado IP Repository)
* Place constraints in `constr/`

### Version Control

Xilinx recommends using version control on `.tcl` scripts, which are used to generate a project, rather than the project itself.
This methodology is followed here, and all sources are outside of the project directory hierarchy, and should be committed to this repository when changed.
All new files should be added as **remote** to the project, and stored according to the Organization section.
As a result, any changes to the project will only be local, unless the `DUNE_WIB.tcl` file is recreated and committed to the repository.
To do this from Vivado, 
1. `File`->`Project`->`Write Tcl...`
2. Choose the `DUNE_WIB.tcl` file in this repository.
3. Ensure the following are checked
    * "Copy sources to new project"
    * "Recreate Block Diagrams using Tcl"
4. Commit the new `.tcl` to the repository.

## Building from Scratch

Building the project is broken into stages:

1. Generate the Ultrascale+ bitstream and boot images.
2. Export hardware definition to a PetaLinux project for the WIB.
3. Configure and build the PetaLinux distribution for the WIB.
4. Create a bootable disk image for the WIB SD card boot mode.

If an earlier stage is modififed, later stages typically need to be rerun.

### Generate bitstream

1. Generate bitstream in Vivado.
2. Export bitstream (`File`->`Export`->`Export Bitstream File`) to `DUNE_WIB.bit`.

### Export hardware definition

Only necessary if block diagram has changed:

1. Export hardware (`File`->`Export`->`Export Hardware`).
2. Copy `DUNE_WIB/DUNE_WIB.sdk/DUNE_WIB.hdf` to `DUNE_WIB.hdf`.

### Create PetaLinux images

PetaLinux 2019.1 is required to build the software for the root filesystem 
image and the kernel to boot the WIB. 

You can either build the Docker image provided in `linux/petalinux-2019.1` and 
use that environment, or install the packages listed in the `Dockerfile` on a 
machine with PetaLinux 2019.1 already installed. See the 
(container readme)[linux/petalinux-2019.1/README.md] for further instructions.

Perform only step 4 if you only want to update the FPGA bitstream. The generated
files can be copied to the SD card boot partition.

1. `cd` into the `linux/` folder.
2. For a new repository, or if block diagram has changed, run `petalinux-config --get-hw-description=../` ensuring that the hardare definition `../DUNE_WIB.hdf` is up to date.
3. For a new repository, or if block diagram has changed, run `petalinux-build` to build the linux system. This can take a long time, but caches build progress for future builds.
4. Run `./make_bootloader.sh` to generate `../BOOT.BIN` and `../image.ub`.
5. Copy (at least) `BOOT.BIN` to SD card boot partition for a new bitstream (`image.ub` as well, to update kernel).
6. Reboot the WIB.

### Creating a bootable SD image

The `linux/make_sd_image.sh` script uses `mtools` and `losetup` to create a
`rootfs.img` file that can be copied to an SD card and boot the WIB. 

1. Ensure your `BOOT.BIN` and `image.ub` files are up-to-date and that `petalinux-build` has been run recently.
2. `cd` into the `linux/` folder.
3. Run `./make_sd_image.sh` to create `../rootfs.img`
4. Assuming your SD card is `/dev/sdX`, run `sudo dd if=../rootfs.img of=/dev/sdX bs=256M status=progress`
5. Run `sync` to ensure the data is written to disk.
6. The SD card is ready to boot the WIB.

## Test drive the linux system with QEMU

With PetaLinux's QEMU one can test the boot process before deploying to 
hardware. This also provides an environment to test software on the linux system
without hardware, but note that QEMU does not by default simulate the PL or even
all of the standard Zynq Ultrascale+ hardware (e.g. ethernet).

You can either build the Docker image provided in `linux/petalinux-2019.1` and 
use that environment, or install the packages listed in the `Dockerfile` on a 
machine with PetaLinux 2019.1 already installed. See the
(container readme)[linux/petalinux-2019.1/README.md] for further instructions.

From the `linux/` directory, boot the image with:

`petalinux-boot --qemu --uboot --qemu-args "-drive file=../rootfs.img,if=sd,format=raw,index=1"`

QEMU will first launch `uboot` in a virtual Ultrascale+ device, which will read 
the provided SD card image `../rootfs.img` and start the Linux kernel on the 
boot partition.

Additional options can be passed to QEMU with `--qemu-args` or directly to
`petalinux-boot` to modify the simulated hardware.
`
