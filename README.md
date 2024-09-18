# Custom OS on Buildroot

This repository contains the resources and instructions for building a custom operating system using [Buildroot](https://buildroot.org/) and a sample .img file of **Linux Loginware 6.6.28**. Buildroot is a tool that simplifies the process of building embedded Linux systems by automating the process of cross-compiling libraries and generating file systems.

## Prerequisites

### Hardware Requirements

- A host system running Linux (Ubuntu).
- Target hardware (Raspberry Pi).

### Software Requirements

- **Buildroot**: Install from the official repository.
- **GNU Make**: Ensure it's installed on your system.
- **Cross-compilation tools**: Handled by Buildroot.
  
For Ubuntu-based systems, you can install the necessary tools via:

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  git \
  bc \
  bison \
  flex \
  libssl-dev \
  libncurses5-dev \
  u-boot-tools \
  qemu \
  gcc-arm-linux-gnueabihf \
  gcc-aarch64-linux-gnu \
  libglib2.0-dev \
  libpixman-1-dev \
  gawk \
  curl \
  unzip \
  rsync \
  fakeroot \
  genisoimage \
  cpio \
  device-tree-compiler \
  xz-utils \
  python3 \
  python3-pip \
  x11-xserver-utils \
  pkg-config \
  libfdt-dev \
  cmake

```

### Clone the Repository

To get started, clone the Buildroot source code:

```bash
git clone https://git.buildroot.org/buildroot
cd buildroot
```

## Configuration

### 1. Configure Target Architecture

Buildroot comes with predefined configurations for many platforms. You can list all supported configurations using:

```bash
make list-defconfigs
```

Choose your target architecture, for example, for a Raspberry Pi 4:

```bash
make raspberrypi4_defconfig
```

### 2. Customize Buildroot

You can customize your OS by modifying the configuration using the Buildroot menuconfig interface:

```bash
make menuconfig
```

Options to configure:
**Click on space to select**
- **System Configuration**: choose the system hostname, system banner, and root password (optional).
- Note: If you are unable to locate any particular package or setting, write a script(shell file) to execute the required and add the path to 'Custom Scripts to run before commencing the build' under 'System Configuration'.
- **Toolchain**: select buildroot toolchain (recommended).
- **Kernel**: Choose the kernel version and configure kernel options.
- **Root Filesystem**: Select the type of root filesystem (e.g., ext4, squashfs, etc.).

#### 1. **Python Packages**
   - **Flask**, **SQLAlchemy**, **Flask-SQLAlchemy**, **Flask-CORS**, **pymodbus**, **schedule**, **aiohttp**:
     - Location: 
       
       Target packages  ---> 
         Interpreter languages and scripting  ---> 
           python3  ---> 
             python3-pip
       
     You will need to include `python3-pip` and then install the specific Python packages via a `post-build` script or add them manually in a custom package.

#### 2. **Node.js and npm Packages**
   - **Node.js**:
     - Location: 
       
       Target packages  ---> 
         Interpreter languages and scripting  ---> 
           nodejs
       
   - **npm global packages (serve)**:
     You will need to install `serve` globally after installing `npm`, potentially through a `post-build` script.


#### 3. **Apache2**
   - **Apache HTTP Server**:
     - Location:
       
       Target packages  --->
         Networking applications  ---> 
           apache
       
#### 4. **Search for other packages/know location and select**
In Buildroot's `menuconfig`, you can search for a package by following these steps:

1. Open `menuconfig`:
   ```bash
   make menuconfig
   ```

2. Press `/` on your keyboard to open the search function.

3. Type the name of the package you're searching for (or part of it) and press `Enter`.

For example, if you're looking for `sqlite`, you can type `sqlite`, and it will display all related packages, along with their menu locations. This feature is very useful for quickly finding where a package is located. in the same way you can include sudo, bash, and other essential packages.


Python and Node.js packages might need to be installed post-build since Buildroot doesn’t natively manage `pip` or `npm` dependencies. SQLite, Apache, and Xorg settings can be directly configured in Buildroot's `menuconfig`.

### 3. Adding Custom Packages

First, create a directory structure for the new custom package inside the `package/` directory of your Buildroot project.

```bash
cd buildroot
mkdir -p package/chromium-browser
```

#### 2. **Create a `.mk` Build File**
Create a `chromium-browser.mk` file that defines how to download, configure, compile, and install Chromium.

```bash
nano package/chromium-browser/chromium-browser.mk
```

Example of chromium-browser:

```makefile
CHROMIUM_BROWSER_VERSION = 91.0.4472.114
CHROMIUM_BROWSER_SITE = https://commondatastorage.googleapis.com/chromium-browser-official/chromium-$(CHROMIUM_BROWSER_VERSION).tar.xz

CHROMIUM_BROWSER_DEPENDENCIES = libglib2 openssl libgtk3

# Set the installation directory for chromium
define CHROMIUM_BROWSER_INSTALL_CMDS
    $(INSTALL) -D -m 0755 $(@D)/out/Default/chrome $(TARGET_DIR)/usr/bin/chromium-browser
endef

$(eval $(generic-package))
```

Replace `libglib2`, `openssl`, and `libgtk3` with any required dependencies, and adjust paths as necessary.

#### 3. **Create a Config File (`Config.in`)**
You will also need a configuration file to integrate the package into the Buildroot configuration system.

```bash
nano package/chromium-browser/Config.in
```

Example:

```makefile
config BR2_PACKAGE_CHROMIUM_BROWSER
    bool "chromium-browser"
    help
      Chromium is an open-source web browser project.
```

#### 4. **Add the Package to Buildroot's Main `Config.in`**
You need to link the new package to the main `Config.in` so that it appears in `menuconfig`.

1. Open `package/Config.in`:

   ```bash
   nano package/Config.in
   ```

2. Add the following line in the appropriate section (e.g., under "Networking applications"):

   ```makefile
   source "package/chromium-browser/Config.in"
   ```

#### 5. **Enable Chromium in `menuconfig`**
Run `menuconfig` to enable the new package:

```bash
make menuconfig
```

- Navigate to `Target packages  ---> Networking applications`
- Select `chromium-browser` to enable it.

#### 6. **build the Image**
Once you have configured the package in `menuconfig`, build the image:

```bash
make
```

Buildroot will download, configure, compile, and install Chromium as part of your custom Linux image.

#### 7. **Expected errors**
- No space left on device:
  - Resize you partition
- may need to resize root filesystem:
    - In `menuconfig` go to: filesystem images ---> exact size : change it to higher value as required (in mb)

#### 8. **Post-build Integration (Optional)**
If you need additional tweaks (e.g., setting Chromium to start in kiosk mode), you can use a `post-build` or `post-image` script to customize the image further.

For example, add a script to auto-launch Chromium in kiosk mode:

```bash
mkdir -p board/custom/post-build
nano board/custom/post-build/post-build.sh
```

In the `post-build.sh`:

```bash
#!/bin/bash
# Auto-launch Chromium in kiosk mode
echo '/usr/bin/chromium-browser --kiosk --disable-infobars' > $(TARGET_DIR)/etc/init.d/S99chromium
chmod +x $(TARGET_DIR)/etc/init.d/S99chromium
```

### 4. Rebuilding the System

To rebuild the entire system, use the following command:

```bash
make
```

Buildroot will download and compile the necessary toolchains, libraries, and applications, and create the root filesystem and kernel image for your target platform.

The generated files will be placed in the `output/images` directory.

### 5. Flashing the OS

Once the OS has been built, you can flash it onto your target device. For SD card-based systems (e.g., Raspberry Pi), you can use the `dd` command or use [raspberry pi imager](https://www.raspberrypi.com/software/):

```bash
sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M
```

Replace `/dev/sdX` with your actual SD card device.

### 6. Booting the Device

Insert the SD card to target hardware (raspberrypi 4), power it on, and your custom Buildroot-based OS should boot.

### 7. Customizing Boot Scripts and Services

If you need the system to auto-start specific services (e.g., launching an application in kiosk mode), you can modify the startup scripts. Add custom init scripts under `/etc/init.d/` or use a custom service manager.

For example, to auto-start the Chromium browser in kiosk mode, edit the `/etc/init.d/S99chromium` script:

```bash
#!/bin/sh
# Start Chromium in Kiosk Mode
startx &
sleep 5
# give a bit delay so that X server will start before launching chromium browser
/usr/bin/chromium-browser --kiosk http://localhost
```

Make sure the script is executable:

```bash
chmod +x /etc/init.d/S99chromium
```

## Debugging

In case of issues, log files can be checked on the device. Serial console or SSH can be used to access the system and troubleshoot issues.

## 7. **Expected error after booting**
**fatal server, no screens found:**

make sure fbdev driver, kbd, xorg server and other dependencies of `Xorg` is selected in menuconfig, This should work but the issue was not resolved even after this in my case, If you find a way to solve please push the solution.
    
## Clean Build

If you want to start fresh, you can clean the output:

```bash
make clean
```

This will remove the compiled binaries and allow you to rebuild the system from scratch.
