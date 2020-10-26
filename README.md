# Raspberry Pi image specs

This is a fork of https://salsa.debian.org/raspi-team/image-specs

It is tweaked to build encrypted root images that I'm using. The image build is suitible for Rpi 3 + 4

## Option 1: Downloading an image

See https://wiki.debian.org/RaspberryPiImages for where to obtain the
latest pre-built image.

## Option 2: Building your own image

Use the [Vagrantfile](Vagrantfile) included to boot up a suitible VM (or to list the dependencies)


### Build a kernel

We want a more up to date kernel with [adiantum](https://github.com/google/adiantum) enabled, as it's like 3x+ faster for encrypted volumes.

#### Code prep

```
git clone -n https://salsa.debian.org/kernel-team/linux.git debian-kernel
cd debian-kernel
git checkout debian/5.9.1-1
(mkdir -p ../orig && cd ../orig && curl -LO http://cdn-fastly.deb.debian.org/debian/pool/main/l/linux/linux_5.9.1.orig.tar.xz)

git am /vagrant/kernelbuild/*.patch
```

#### Host kernel

We need one for the vagrant VM so we can create the initial FS and images.

```
ARCH=amd64
FEATURESET=none
FLAVOUR=lstoll-amd64
export $(dpkg-architecture -a$ARCH)
export PATH=/usr/lib/ccache:$PATH
export DEB_BUILD_PROFILES="cross nopython nodoc pkg.linux.notools"
export MAKEFLAGS="-j$(($(nproc)*2))"
export DEBIAN_KERNEL_DISABLE_DEBUG=

fakeroot make -f debian/rules clean
fakeroot make -f debian/rules orig
fakeroot make -f debian/rules source
fakeroot make -f debian/rules.gen setup_${ARCH}_${FEATURESET}_${FLAVOUR}
fakeroot make -f debian/rules.gen binary-arch_${ARCH}_${FEATURESET}_${FLAVOUR}

```

Additions will be needed. Make sure the latest version of virtualbox is installed (https://www.virtualbox.org/ticket/19845).

#### Target kernel

Build a kernel for the pi

```
ARCH=arm64
FEATURESET=none
FLAVOUR=lstoll-arm64
export $(dpkg-architecture -a$ARCH)
export PATH=/usr/lib/ccache:$PATH
export DEB_BUILD_PROFILES="cross nopython nodoc pkg.linux.notools"
export MAKEFLAGS="-j$(($(nproc)*2))"
export DEBIAN_KERNEL_DISABLE_DEBUG=

fakeroot make -f debian/rules clean
fakeroot make -f debian/rules orig
fakeroot make -f debian/rules source
fakeroot make -f debian/rules.gen setup_${ARCH}_${FEATURESET}_${FLAVOUR}
fakeroot make -f debian/rules.gen binary-arch_${ARCH}_${FEATURESET}_${FLAVOUR}
```

#### Releasing

Publish to a apt repo on S3, for easier updates & distribution

amd64:

```
aws-vault exec <profile> -- bundle exec deb-s3 upload -a amd64 -p -b lstoll-kernels-apt -c buster --sign=B26F07E7 debs/linux-image-5.9.0-1-lstoll-amd64-unsigned_5.9.1-1_amd64.deb
aws-vault exec <profile> -- bundle exec deb-s3 upload -a amd64 -p -b lstoll-kernels-apt -c buster --sign=B26F07E7 debs/linux-headers-5.9.0-1-lstoll-amd64_5.9.1-1_amd64.deb
```

arm64:

```
aws-vault exec <profile> -- bundle exec deb-s3 upload -a arm64 -p -b lstoll-kernels-apt -c buster --sign=B26F07E7 debs/linux-image-5.9.0-1-lstoll-arm64-unsigned_5.9.1-1_arm64.deb
aws-vault exec <profile> -- bundle exec deb-s3 upload -a arm64 -p -b lstoll-kernels-apt -c buster --sign=B26F07E7 debs/linux-headers-5.9.0-1-lstoll-arm64_5.9.1-1_arm64.deb
```

### Build an OS image

For this you will first need to install the following packages on a
Debian Buster (10) or higher system:

* vmdb2
* dosfstools
* qemu-user-static
* binfmt-support

Do note that –at least currently– vmdb2 uses some syntax that is available
only in the version in testing (Bullseye).

This repository includes a master YAML recipe (which is basically a
configuration file) for all of the generated images, diverting as
little as possible in a parametrized way. The master recipe is
[raspi_master.yaml](raspi_master.yaml).

A Makefile is supplied to drive the build of the recipes into images —
`raspi_0w` (for the Raspberry Pi 0, 0w and 1, models A and B),
`raspi_2` (for the Raspberry Pi 2, models A and B), `raspi_3`
(for all models of the Raspberry Pi 3), and `raspi_4` (for all
models of the Raspberry Pi 4). That is, if you want to build the
default image for a Raspberry Pi 3B+, you can just issue:

```shell
   make raspi_3.img
```

You might also want to edit them to customize the built image. If you
want to start from the platform-specific recipe, you can issue:

```shell
   make raspi_3.yaml
```
The recipe drives [vmdb2](https://vmdb2.liw.fi/), the successor to
`vmdebootstrap`. Please refer to [its
documentation](https://vmdb2.liw.fi/documentation/) for further
details; it is quite an easy format to understand.

Copy the generated file to a name descriptive enough for you (say,
`my_raspi.yaml`). Once you have edited the recipe for your specific
needs, you can generate the image by issuing the following (as root):

```shell
    vmdb2 --rootfs-tarball=my_raspi.tar.gz --output \
	my_raspi.img my_raspi.yaml --log my_raspi.log
```

This is, just follow what is done by the `_build_img` target of the Makefile.

## Installing the image onto the Raspberry Pi

Plug an SD card which you would like to entirely overwrite into your SD card reader.

Assuming your SD card reader provides the device `/dev/mmcblk0`
(**Beware** If you choose the wrong device, you might overwrite
important parts of your system.  Double check it's the correct
device!), copy the image onto the SD card:

```shell
sudo dd if=raspi_3.img of=/dev/mmcblk0 bs=64k oflag=dsync status=progress
```

Then, plug the SD card into the Raspberry Pi, and power it up.

The image uses the hostname `rpi0w`, `rpi2`, `rpi3`, or `rpi4` depending on the
target build. The provided image will allow you to log in with the
`root` account with no password set, but only logging in at the
physical console (be it serial or by USB keyboard and HDMI monitor).

## References

* https://salsa.debian.org/raspi-team/image-specs
* https://wiki.debian.org/HowToCrossBuildAnOfficialDebianKernelPackage
