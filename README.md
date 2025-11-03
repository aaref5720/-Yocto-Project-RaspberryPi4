# -Yocto-Project-RaspberryPi4

## Overview
This guide provides a step-by-step breakdown of The Project.

---

## Setting the Environment
### Install Dependencies (Ubuntu)
```bash
sudo apt install build-essential chrpath cpio debianutils diffstat file gawk gcc git iputils-ping \
libacl1 liblz4-tool locales python3 python3-git python3-jinja2 python3-pexpect python3-pip \
python3-subunit socat texinfo unzip wget xz-utils zstd git inkscape locales make \
python3-saneyaml python3-sphinx-rtd-theme sphinx texlive-latex-extra
```
Verify that `en_US.utf8` locale is available:
```bash
locale --all-locales | grep en_US.utf8
```

### Download Poky
Poky is the reference build system for the Yocto Project.
```bash
git clone -b kirkstone https://github.com/yoctoproject/poky.git
cd poky
```

### Initialize the Build Environment
Run inside `poky`:
```bash
source oe-init-build-env ../build-raspberrypi4-64
```
This creates the `build` directory and sets up environment variables.

### Configure Local Build Settings
Edit `local.conf`:
```bash
MACHINE ?= "raspberrypi4-64"
BB_NUMBER_THREADS = "${@bb.utils.cpu_count()//2}"
PARALLEL_MAKE = "-j 4"
```
- **`MACHINE`** → Raspberry Pi 4 (64 bits) as a target hardware.
- **`BB_NUMBER_THREADS & PARALLEL_MAKE`** → Optimize build using multiple CPU cores.
- **`bb.utils.cpu_count()`** → BitBake utility function that returns the number of CPU cores available on the system.  
- **`//2`** → Python's integer division.

### BitBake Configurations
1. **`build/conf/bblayers.conf`** → Specifies the layers included in the build, like `meta-openembedded` and `meta-yocto`.
2. **`build/conf/local.conf`** → Contains user-specific settings (e.g., `MACHINE`, `DISTRO`).
3. **`meta/conf/layer.conf`** → Defines how BitBake processes each layer, including priorities and dependencies.

### BitBake Layers
- **`meta-poky`** → Provides the Poky reference distribution.
- **`meta-yocto-bsp`** → Contains Board Support Packages (BSPs) for reference hardware.
- **`meta-raspberrypi`** → Adds support for Raspberry Pi boards.
- **`meta-oe`** → Provides additional OpenEmbedded recipes and packages.
- **`meta-python`** → Includes extra Python packages.
- **`meta-networking`** → Provides networking utilities and services.
- **`meta-qt5`** → Adds support for the Qt5 framework.
- **Custom Layers:**
  - **`meta-info-distro`** → Infotainment distribution configurations.
  - **`meta-audio-distro`** → Audio distribution configurations.
  - **`meta-IVI`** → Contains an image recipe with a C++ application, Nano editor and VSOMEIP.

---

## Create Infotainment Distro
### Create the Layer Directory Structure
```bash
mkdir -p meta-info-distro/conf/distro 
touch meta-info-distro/conf/distro/infotainment.conf
```
Edit `infotainment.conf`:
```bash
DISTRO="infotainment"
DISTRO_NAME="Bullet-infotainment"
DISTRO_VERSION="1.0"

MAINTAINER="abdelrahmanmohamed.req@gmail.com"


# SDK Information.
SDK_VENDOR = "-bulletSDK"
SDK_VERSION = "${@d.getVar('DISTRO_VERSION').replace('snapshot-${METADATA_REVISION}', 'snapshot')}"
SDK_VERSION[vardepvalue] = "${SDK_VERSION}"

SDK_NAME = "${DISTRO}-${TCLIBC}-${SDKMACHINE}-${IMAGE_BASENAME}-${TUNE_PKGARCH}-${MACHINE}"
# Installation path --> can be changed to ${HOME}-${DISTRO}-${SDK_VERSION}
SDKPATHINSTALL = "/opt/${DISTRO}/${SDK_VERSION}" 

# Disribution Feature --> NOTE: used to add customize package (for package usage).

# infotainment --> INFOTAINMENT

INFOTAINMENT_DEFAULT_DISTRO_FEATURES = "largefile opengl ptest multiarch vulkan x11 bluez5 bluetooth wifi qt5 info"

# TODO: to be org.

DISTRO_FEATURES ?= "${DISTRO_FEATURES_DEFAULT} ${INFOTAINMENT_DEFAULT_DISTRO_FEATURES} userland"

# install systemd  as init manager 
require conf/distro/include/systemd.inc

# prefered version for packages.
PREFERRED_VERSION_linux-yocto ?= "5.15%"
PREFERRED_VERSION_linux-yocto-rt ?= "5.15%"


# Build System configuration.

LOCALCONF_VERSION="2"

# add poky sanity bbclass
INHERIT += "poky-sanity"

```
Update `local.conf` to use infotainment distro:
```bash
DISTRO ?= "infotainment"
```

---

## Enable Systemd for Infotainment Distribution
Poky uses `sysvinit` by default. Switch to `systemd`:

### Create Directory Structure
```bash
mkdir -p meta-IVI/conf/distro/include
touch meta-IVI/conf/distro/include/systemd.inc
```

### Configure Systemd
```bash
# install systemd  as init manager 
DISTRO_FEATURES:append = " systemd" 


# select systemd as init manager 
VIRTUAL-RUNTIME_init_manager = " systemd"
VIRTUAL-RUNTIME_initscripts = " systemd-compat-units"
```

---

## Create Audio Distro
### Create the Layer Directory Structure
```bash
mkdir -p meta-audio-distro/conf/distro
touch meta-audio-distro/conf/distro/audio.conf
```
Edit `audio.conf`:
```bash
DISTRO="audio"
DISTRO_NAME="Bullet-audio"
DISTRO_VERSION="1.0"

MAINTAINER="abdelrahmanmohamed.req@gmail.com"


# SDK Information.
SDK_VENDOR = "-bulletSDK"
SDK_VERSION = "${@d.getVar('DISTRO_VERSION').replace('snapshot-${METADATA_REVISION}', 'snapshot')}"
SDK_VERSION[vardepvalue] = "${SDK_VERSION}"

SDK_NAME = "${DISTRO}-${TCLIBC}-${SDKMACHINE}-${IMAGE_BASENAME}-${TUNE_PKGARCH}-${MACHINE}"
# Installation path --> can be changed to ${HOME}-${DISTRO}-${SDK_VERSION}
SDKPATHINSTALL = "/opt/${DISTRO}/${SDK_VERSION}" 

# Disribution Feature --> NOTE: used to add customize package (for package usage).

# audio --> AUDIO

AUDIO_DEFAULT_DISTRO_FEATURES = "largefile opengl ptest multiarch vulkan bluez5 bluetooth wifi audio_only"

# TODO: to be org.

DISTRO_FEATURES ?= "${DISTRO_FEATURES_DEFAULT} ${AUDIO_DEFAULT_DISTRO_FEATURES} userland"

# prefered version for packages.
PREFERRED_VERSION_linux-yocto ?= "5.15%"
PREFERRED_VERSION_linux-yocto-rt ?= "5.15%"


# Build System configuration.

LOCALCONF_VERSION="2"

# add poky sanity bbclass
INHERIT += "poky-sanity"
```
Update `local.conf` to use infotainment distro:
```bash
DISTRO ?= "audio"
```

---

## Creating Cpp App Recipe `helloworld`
```bash
mkdir -p meta-IVI/recipes-native-cpp/helloworld
cd meta-IVI/recipes-native-cpp/helloworld
recipetool create -o helloworld_1.0.bb https://github.com/embeddedlinuxworkshop/y_t1.git
```
After Generate the Recipe, Add Some Changes, the Final Recipe:
```bash 
# Recipe created by recipetool
# This is the basis of a recipe and may need further editing in order to be fully functional.
# (Feel free to remove these comments when editing.)

# TODO: 1. Decumentation Variables
SUMMARY		= "Example for Native C++ Application for Testing YOCTO"
DESCRIPTION	= "Example for Native C++ Application for Testing YOCTO. Provided by Bullet Guru"
HOMEPAGE	= "http://github.com/embeddedlinuxworkshop/y_t1"

# Unable to find any files that looked like license statements. Check the accompanying
# documentation and source headers and set LICENSE and LIC_FILES_CHKSUM accordingly.
#
# NOTE: LICENSE is being set to "CLOSED" to allow you to at least start building - if
# this is not accurate with respect to the licensing of the software being built (it
# will not be in most cases) you must specify the correct value before using this
# recipe for anything other than initial testing/development!

# TODO: 2. Licence Variables
LICENSE = "CLOSED"
LIC_FILES_CHKSUM = ""

# TODO: 3. Source Code Variables
SRC_URI = "git://github.com/embeddedlinuxworkshop/y_t1;protocol=http;branch=master"

# Modify these as desired
PV = "1.0+git${SRCPV}"
SRCREV = "49600e3cd69332f0e7b8103918446302457cd950"

S = "${WORKDIR}/git"

# TODO: 4. Tasks Excuted through the Build Engine
# NOTE: no Makefile found, unable to determine what needs to be done

APPLICATION = "hello"

do_compile () {
	# Specify compilation commands here
	
	# Compile Cross-Compiler (Compiler Target )
	$CXX "${S}"/main.cpp -o "${APPLICATION}"
}

do_install () {
	# Specify install commands here
	
	# 1. manipulate -> ${WORKDIR}/image
	# 2. Create Directory ${WORKDIR}/image/usr/bin
	install -d "${D}"/"${bindir}"

	#3. installing hello bin in Directory ${WORKDIR}/image/usr/bin 
	install -m 0755 "${APPLICATION}" "${D}"/"${bindir}"
}

# Ignore do_package_qa
do_package_qa[noexec]="1"
```
**do_compile ()**
This function is automatically called during the build process to compile source code.
- `${CXX}` → Uses the C++ compiler set by Yocto.

**do_install ()**
used for copying and setting file permissions.
- `install -d` → Creates the destination directory.
- `${D}` `${bindir}` → Installs the compiled binary into `/usr/bin/` inside the target filesystem.
- `install -m 0755` → Copies the file and sets permissions (rwxr-xr-x).

Build the Recipe:
```bash
bitbake helloworld
```

---

## Integrate Nano
```bash
mkdir -p meta-IVI/recipes-editors/nano
cd meta-IVI/recipes-editors/nano
recipetool create -o nano_1.0.bb https://ftp.gnu.org/gnu/nano/nano-7.2.tar.xz
bitbake nano
```
Install the dependencies required for building Nano
```bash 
sudo apt install autoconf automake autopoint gcc gettext git groff make pkg-config texinfo
```
Fetch and unpack the source code
```bash
bitbake -c fetch nano
bitbake -c unpack nano
```
Find the WORKDIR path:
```bash
bitbake -e nano | grep -i "^WORKDIR="
```
Navigate to the `WORKDIR/git` path.

Run autogen.sh to generate the configure script:
```bash
./autogen.sh
```
Build the Recipe:
```bash
bitbake nano
```

---

## Integrate Audio
Create the `classes/` directory
```bash
cd meta-IVI
mkdir -p classes
touch classes/audio.bbclass
```
Edit the class: 
```bash
IMAGE_INSTALL:append = " pavucontrol pulseaudio pulseaudio-module-dbus-protocol pulseaudio-server \
        pulseaudio-module-loopback pulseaudio-module-bluetooth-discover alsa-ucm-conf pulseaudio-module-bluetooth-policy alsa-topology-conf alsa-state alsa-lib alsa-tools \
        pulseaudio-module-bluez5-device pulseaudio-module-bluez5-discover alsa-utils alsa-plugins packagegroup-rpi-test can-utils net-tools gstreamer1.0 \
        iproute2 iputils libsocketcan bluez5 i2c-tools hostapd iptables"
```

---

## Integrate rpi-play for iPhone
```bash
mkdir -p meta-IVI/recipes-rpiplay
cd meta-IVI/recipes-rpiplay
recipetool create -o rpi-play_1.0.bb https://github.com/FD-/RPiPlay.git
```
### Integrate its Dependencies:
from RPiPlay Repo
```
The following packages are required for building on Raspbian:
- cmake (for the build system)
- libavahi-compat-libdnssd-dev (for the bonjour registration)
- libplist-dev (for plist handling)
- libssl-dev (for crypto primitives)
- ilclient and Broadcom's OpenMAX stack as present in /opt/vc in Raspbian.
```
**ilclient** 
ilclient exists, but in a different location in `userland` package
so we need to correct the CMake paths. 
```bash
find . -name "*ilclient*"
```
```
/home/aref/yocto/build_raspberrypi4-64/tmp-glibc/work/cortexa7t2hf-neon-vfpv4-oe-linux-gnueabi/userland/20220323-r0/image/usr/src/hello_pi/libs/ilclient
```
When bitbake builds `rpi-play`, it pulls dependencies from the sysroot of its dependencies (like userland). But in this case, userland installs `ilclient` locally inside its own image/ directory, and not in the sysroot, so RPiPlay can't find it.
### Step 1: Adjusting userland Sysroot
Since userland installs ilclient inside its own image directory but not in sysroot, we need to tell Yocto to `include/usr/src/hello_pi/libs` in the sysroot of rpiplay.
```bash
cd meta-raspberrypi/recipes-graphics/userland/userland_git.bb
```
Add the following line inside `userland_git.bb`:
```bash
SYSROOT_DIRS:append="${prefix}/src"
```
This ensures that `/usr/src/hello_pi/libs` is copied into the sysroot, making ilclient available for `rpi-play`.
Patch CMakeLists.txt Instead of /opt/vc, CMakeLists.txt should look in /usr/src/hello_pi/libs.

### Step 2: Patching CMake
Navigate to the rpi-play source directory after unpacking:

the path of working directory of the rpi-play
```bash
cd /home/aref/yocto/build_raspberrypi4-64/tmp-glibc/work/cortexa7t2hf-neon-vfpv4-oe-linux-gnueabi/rpi-play/1.0+gitAUTOINC+64d0341ed3-r0/git
```

Modify renders/CMakeLists.txt to replace `/opt/vc` with `/usr/`.
Generate a patch:
```bach
bitbake -c devshell rpi-play
git diff > 0001_include_dir.patch
``` 
change each  `/opt/vc` with  `/usr/`

the new patch file:
```patch
diff --git a/renderers/CMakeLists.txt b/renderers/CMakeLists.txt
index e561250..915ba92 100755
--- a/renderers/CMakeLists.txt
+++ b/renderers/CMakeLists.txt
@@ -17,20 +17,20 @@ set( RENDERER_LINK_LIBS "" )
 set( RENDERER_INCLUDE_DIRS "" )
 
 # Check for availability of OpenMAX libraries on Raspberry Pi
-find_library( BRCM_GLES_V2 brcmGLESv2 HINTS ${CMAKE_SYSROOT}/opt/vc/lib/ )
-find_library( BRCM_EGL brcmEGL HINTS ${CMAKE_SYSROOT}/opt/vc/lib/ )
-find_library( OPENMAXIL openmaxil HINTS ${CMAKE_SYSROOT}/opt/vc/lib/ )
-find_library( BCM_HOST bcm_host HINTS ${CMAKE_SYSROOT}/opt/vc/lib/ )
-find_library( VCOS vcos HINTS ${CMAKE_SYSROOT}/opt/vc/lib/ )
-find_library( VCHIQ_ARM vchiq_arm HINTS ${CMAKE_SYSROOT}/opt/vc/lib/ )
+find_library( BRCM_GLES_V2 brcmGLESv2 HINTS ${CMAKE_SYSROOT}/usr/lib/ )
+find_library( BRCM_EGL brcmEGL HINTS ${CMAKE_SYSROOT}/usr/lib/ )
+find_library( OPENMAXIL openmaxil HINTS ${CMAKE_SYSROOT}/usr/lib/ )
+find_library( BCM_HOST bcm_host HINTS ${CMAKE_SYSROOT}/usr/lib/ )
+find_library( VCOS vcos HINTS ${CMAKE_SYSROOT}/usr/lib/ )
+find_library( VCHIQ_ARM vchiq_arm HINTS ${CMAKE_SYSROOT}/usr/lib/ )
 
 if( BRCM_GLES_V2 AND BRCM_EGL AND OPENMAXIL AND BCM_HOST AND VCOS AND VCHIQ_ARM )
   # We have OpenMAX libraries available! Use them!
   message( STATUS "Found OpenMAX libraries for Raspberry Pi" )
-  include_directories( ${CMAKE_SYSROOT}/opt/vc/include/ 
-  	${CMAKE_SYSROOT}/opt/vc/include/interface/vcos/pthreads 
-  	${CMAKE_SYSROOT}/opt/vc/include/interface/vmcs_host/linux 
-  	${CMAKE_SYSROOT}/opt/vc/src/hello_pi/libs/ilclient )
+  include_directories( ${CMAKE_SYSROOT}/usr/include/ 
+  	${CMAKE_SYSROOT}/usr/include/interface/vcos/pthreads 
+  	${CMAKE_SYSROOT}/usr/include/interface/vmcs_host/linux 
+  	${CMAKE_SYSROOT}/usr/src/hello_pi/libs/ilclient )
 
   option(BUILD_SHARED_LIBS "" OFF)
   add_subdirectory(fdk-aac EXCLUDE_FROM_ALL)
@@ -38,7 +38,7 @@ if( BRCM_GLES_V2 AND BRCM_EGL AND OPENMAXIL AND BCM_HOST AND VCOS AND VCHIQ_ARM
 
   set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DHAVE_LIBOPENMAX=2 -DOMX -DOMX_SKIP64BIT -ftree-vectorize -pipe -DUSE_EXTERNAL_OMX   -DHAVE_LIBBCM_HOST -DUSE_EXTERNAL_LIBBCM_HOST -DUSE_VCHIQ_ARM -Wno-psabi" )
   
-  aux_source_directory( ${CMAKE_SYSROOT}/opt/vc/src/hello_pi/libs/ilclient/ ilclient_src )
+  aux_source_directory( ${CMAKE_SYSROOT}/usr/src/hello_pi/libs/ilclient/ ilclient_src )
   set( DIR_SRCS ${ilclient_src} )
   add_library( ilclient STATIC ${DIR_SRCS} )
```
Copy the patch to the recipe folder
```bach
cp 0001_include_dir.patch  /home/aref/yocto/build_raspberrypi4-64/tmp-glibc/work/cortexa7t2hf-neon-vfpv4-oe-linux-gnueabi/rpi-play/1.0+gitAUTOINC+64d0341ed3-r0/git/patches
```
Add Remaining Dependencies:
```bash 
DEPENDS = "userland openssl avahi mdns libplist gstreamer1.0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-vaapi gstreamer1.0-plugins-bad"
RDEPENDS_${PN} = " avahi libplist gstreamer1.0-plugins-base gstreamer1.0-plugins-good "
```

edit `rpi-play_1.0.bb`
```bash
# Recipe created by recipetool
# This is the basis of a recipe and may need further editing in order to be fully functional.
# (Feel free to remove these comments when editing.)

# WARNING: the following LICENSE and LIC_FILES_CHKSUM values are best guesses - it is
# your responsibility to verify that the values are complete and correct.
#
# The following license files were not able to be identified and are
# represented as "Unknown" below, you will need to check them yourself:
#   LICENSE
#   lib/llhttp/LICENSE-MIT
#   lib/playfair/LICENSE.md
LICENSE = "Unknown"
LIC_FILES_CHKSUM = "file://LICENSE;md5=1ebbd3e34237af26da5dc08a4e440464 \
                    file://lib/llhttp/LICENSE-MIT;md5=f5e274d60596dd59be0a1d1b19af7978 \
                    file://lib/playfair/LICENSE.md;md5=c7cd308b6eee08392fda2faed557d79a"

SRC_URI = "git://github.com/FD-/RPiPlay.git;protocol=https;branch=master \
    file://0001_include_dir.patch"

# Modify these as desired
PV = "1.0+git${SRCPV}"
SRCREV = "64d0341ed3bef098c940c9ed0675948870a271f9"

S = "${WORKDIR}/git"

# NOTE: the following library dependencies are unknown, ignoring: bcm_host openmaxil vcos brcmEGL brcmGLESv2 plist-2 vchiq_arm plist
#       (this is based on recipes that have previously been built and packaged)
DEPENDS = "userland openssl avahi mdns libplist gstreamer1.0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-vaapi gstreamer1.0-plugins-bad"
RDEPENDS_${PN} = " avahi libplist gstreamer1.0-plugins-base gstreamer1.0-plugins-good "

inherit cmake pkgconfig

# Specify any options you want to pass to cmake using EXTRA_OECMAKE:
EXTRA_OECMAKE = ""
TARGET_LDFLAGS += "-Wl,--copy-dt-needed-entries"
EXTRA_OEMAKE:append = " LDFLAGS='${TARGET_LDFLAGS}'"

# do_recipesysroot_main(){
#     mkdir -p "${WORKDIR}/recipe-sysroot/opt/vc"
#     cp -r  "${WORKDIR}/recipe-sysroot/usr/src" "${WORKDIR}/recipe-sysroot/opt/vc"
#     cp -r  "${WORKDIR}/recipe-sysroot/usr/include" "${WORKDIR}/recipe-sysroot/opt/vc"
#     cp -r  "${WORKDIR}/recipe-sysroot/usr/lib" "${WORKDIR}/recipe-sysroot/opt/vc"
#     cp -r  "${WORKDIR}/recipe-sysroot/usr/bin" "${WORKDIR}/recipe-sysroot/opt/vc"
#     cp -r  "${WORKDIR}/recipe-sysroot/usr/share" "${WORKDIR}/recipe-sysroot/opt/vc"
# }
# addtask do_recipesysroot_main before do_configure

```
Build `rpi-play` with the Patched CMakeLists.txt and Dependencies: 
```bash
bitbake rpi-play
```

## Create the Image Recipe: `ivi-test-image.bb`
### Create Directory Structure
```bash
mkdir -p meta-IVI/recipes-core/images
touch meta-IVI/recipes-core/images/ivi-test-image.bb
```

### Define Image Recipe
```bash
# Base this image on rpi-test-image
require recipes-core/images/rpi-test-image.bb

# Summary of the Image
SUMMARY="IVI Testing Image That Include RPI Functions and helloworld Package Recipes"

### MACHINE_FEATURES ###
MACHINE_FEATURES:append=" bluetooth wifi alsa"

### IMAGE INSTALLATION ###
IMAGE_INSTALL:append=" helloworld openssh nano vsomeip"

# if Distro ?= "infotainment"
inherit ${@bb.utils.contains("DISTRO_FEATURES", "info", "populate_sdk_qt5", "", d)}
IMAGE_INSTALL:append="${@bb.utils.contains("DISTRO_FEATURES", "info", " rpi-play qtbase-examples qtquickcontrols qtbase-plugins libsocketcan qtquickcontrols2 qtgraphicaleffects qtmultimedia qtserialbus qtquicktimeline qtvirtualkeyboard", " ", d)}"

# if Distro ?= "audio"
inherit ${@bb.utils.contains("DISTRO_FEATURES", "audio_only", "audio", "", d)}

### IMAGE_FEATURES ###
##########################################################
## 1. IMAGE_INSTALL --> ssh                             ##
## 2. do_rootfs -->                                     ##
##    - allow root access through ssh                   ##
##    - access root through ssh using empty password    ##
##########################################################
IMAGE_FEATURES:append=" ssh-server-openssh"
```
**Base Image** 
`require`: Defines the core structure of the image by inheriting from an existing base image `rpi-test-image`.

**Inheritance** 
`inherit`: Some images inherit special classes that modify their behavior 

- Inherit `audio.bbclass` for `audio` distro.
```bash 
inherit ${@bb.utils.contains("DISTRO_FEATURES", "audio_only", "audio", "", d)}
``` 
- Inherit populate_sdk_qt5 classes for `infotainment` distro to create a Qt5 SDK.
```bash
inherit ${@bb.utils.contains("DISTRO_FEATURES", "info", "populate_sdk_qt5", "", d)}
```

**Package Installation** 
`IMAGE_INSTALL`: Specifies additional software packages to be included in the image `nano`, `helloworld`, `openssh`, `vsomeip`.

- Install `rpi-play` and `qt pachakges`for `infotainment` distro.
```bash
IMAGE_INSTALL:append="${@bb.utils.contains("DISTRO_FEATURES", "info", " rpi-play qtbase-examples qtquickcontrols qtbase-plugins libsocketcan qtquickcontrols2 qtgraphicaleffects qtmultimedia qtserialbus qtquicktimeline qtvirtualkeyboard"", " ", d)}"
```

**Image Features** 
`IMAGE_FEATURES`: Defines additional capabilities like SSH, debugging tools, or package management `ssh-server-openssh`, `debug-tweaks`.

**Machine Features** 
`MACHINE_FEATURES`: Defines hardware-specific features available for the target machine `alsa`, `wifi`, `bluetooth`.

---

## Building an Image
Choose the Desired Distro `audio` or `infotainment` in `local.conf` and Run: 
```bash
bitbake ivi-test-image
```
---


