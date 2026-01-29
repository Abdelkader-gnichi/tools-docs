# OpenWrt Build System Guide

Complete guide for building OpenWrt packages and firmware from source.

---

## Table of Contents

1. [Prerequisites & Dependencies](#prerequisites--dependencies)
2. [Scenario 1: Building Individual Packages](#scenario-1-building-individual-packages)
3. [Scenario 2: Building Full Firmware](#scenario-2-building-full-firmware)
4. [Troubleshooting](#troubleshooting)
5. [Additional Resources](#additional-resources)

---

## Prerequisites & Dependencies

### System Requirements

- Linux-based OS (Ubuntu/Debian recommended)
- At least 10GB free disk space for buildroot
- Additional 5-10GB per target architecture
- 4GB+ RAM (8GB+ recommended)
- Internet connection for downloading sources

### Installing Build Dependencies

#### For Ubuntu/Debian:

```bash
sudo apt update
sudo apt install -y \
    build-essential \
    clang \
    flex \
    bison \
    g++ \
    gawk \
    gcc-multilib \
    g++-multilib \
    gettext \
    git \
    libncurses5-dev \
    libssl-dev \
    python3-distutils \
    rsync \
    unzip \
    zlib1g-dev \
    file \
    wget \
    python3 \
    python3-pip \
    python3-setuptools \
    qemu-utils \
    swig \
    libelf-dev \
    libfuse-dev
```

#### For Fedora/RHEL/CentOS:

```bash
sudo dnf install -y \
    bash \
    binutils \
    bzip2 \
    gcc \
    gcc-c++ \
    gawk \
    gettext \
    git \
    glibc-devel \
    glibc-static \
    libstdc++-static \
    ncurses-devel \
    openssl-devel \
    perl-ExtUtils-MakeMaker \
    perl-FindBin \
    perl-Thread-Queue \
    python3 \
    rsync \
    unzip \
    wget \
    which \
    zlib-devel
```

#### For Arch Linux:

```bash
sudo pacman -S --needed \
    base-devel \
    clang \
    git \
    ncurses \
    openssl \
    python \
    wget \
    unzip \
    libelf
```

### Clone OpenWrt Repository

```bash
# Clone the latest OpenWrt source
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt

# Or clone a specific stable release (recommended for production)
git clone https://git.openwrt.org/openwrt/openwrt.git -b openwrt-23.05
cd openwrt
```

### Update and Install Feeds

```bash
# Update the feeds
./scripts/feeds update -a

# Install all packages from feeds
./scripts/feeds install -a
```

---

## Scenario 1: Building Individual Packages

This scenario covers adding a custom package to OpenWrt and compiling it separately.

### Step 1: Prepare Your Package Source

Assume you have a package source directory with this structure:

```
my-package/
├── Makefile        # OpenWrt package Makefile
└── src/            # Source code directory
    ├── main.c
    └── Makefile    # Source code Makefile (optional)
```

### Step 2: Add Package to OpenWrt Build System

```bash
# Navigate to OpenWrt root directory
cd /path/to/openwrt

# Copy your package to the package directory
# You can organize it under a category
cp -r /path/to/my-package package/custom/my-package

# Alternative: Create a symbolic link (useful for development)
ln -s /path/to/my-package package/custom/my-package
```

### Step 3: Example Package Makefile

Here's a template for `package/custom/my-package/Makefile`:

```makefile
include $(TOPDIR)/rules.mk

# Package information
PKG_NAME:=my-package
PKG_VERSION:=1.0.0
PKG_RELEASE:=1

# If using external source
# PKG_SOURCE_URL:=https://github.com/user/repo
# PKG_SOURCE_VERSION:=main
# PKG_MIRROR_HASH:=skip

# If source is in the same directory
PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk

# Package definition
define Package/my-package
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=My Custom Package
  DEPENDS:=+libc +libpthread
  URL:=https://example.com
endef

define Package/my-package/description
  Detailed description of what your package does.
  This can span multiple lines.
endef

# Build preparation
define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	$(CP) ./src/* $(PKG_BUILD_DIR)/
endef

# Compile configuration
define Build/Configure
	# Add configure commands if needed
	# $(call Build/Configure/Default)
endef

# Compilation
define Build/Compile
	$(MAKE) -C $(PKG_BUILD_DIR) \
		CC="$(TARGET_CC)" \
		CFLAGS="$(TARGET_CFLAGS)" \
		LDFLAGS="$(TARGET_LDFLAGS)"
endef

# Installation to package
define Package/my-package/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/my-package $(1)/usr/bin/
	
	# Install configuration files (if any)
	# $(INSTALL_DIR) $(1)/etc/config
	# $(INSTALL_CONF) ./files/my-package.config $(1)/etc/config/my-package
	
	# Install init script (if any)
	# $(INSTALL_DIR) $(1)/etc/init.d
	# $(INSTALL_BIN) ./files/my-package.init $(1)/etc/init.d/my-package
endef

# Pre/post install scripts (optional)
define Package/my-package/preinst
#!/bin/sh
# Commands to run before installation
exit 0
endef

define Package/my-package/postinst
#!/bin/sh
# Commands to run after installation
exit 0
endef

$(eval $(call BuildPackage,my-package))
```

### Step 4: Configure Build System

```bash
# Run menuconfig to select your package
make menuconfig
```

**Navigation in menuconfig:**

1. Navigate to your package category (e.g., `Utilities`)
2. Find your package (`my-package`)
3. Press `M` to mark it as a module (builds as .ipk)
4. Press `*` to include it in firmware (builds into firmware)
5. Press `Space` to toggle selection
6. Save configuration and exit (Save → Exit → Exit)

**Alternative: Select via command line**

```bash
# Enable package as module
echo "CONFIG_PACKAGE_my-package=m" >> .config

# Update configuration
make defconfig
```

### Step 5: Build the Package

```bash
# Build only your specific package
make package/my-package/compile V=s

# V=s provides verbose output (helpful for debugging)
# V=sc shows full commands being executed
```

**The compiled package will be located at:**

```bash
bin/packages/<architecture>/custom/my-package_1.0.0-1_<architecture>.ipk
```

Example path:
```
bin/packages/mipsel_24kc/custom/my-package_1.0.0-1_mipsel_24kc.ipk
```

### Step 6: Clean Build (if needed)

```bash
# Clean only your package
make package/my-package/clean

# Clean and rebuild
make package/my-package/clean V=s
make package/my-package/compile V=s
```

### Step 7: Transfer Package to OpenWrt Device

#### Method 1: SCP Transfer

```bash
# Find your compiled package
PACKAGE_PATH=$(find bin/packages -name "my-package*.ipk")

# Transfer to device
scp $PACKAGE_PATH root@192.168.1.1:/tmp/

# Or specify the exact path
scp bin/packages/mipsel_24kc/custom/my-package_1.0.0-1_mipsel_24kc.ipk root@192.168.1.1:/tmp/
```

#### Method 2: Using `rsync`

```bash
rsync -avz bin/packages/mipsel_24kc/custom/my-package_*.ipk root@192.168.1.1:/tmp/
```

### Step 8: Install Package on OpenWrt Device

```bash
# SSH into your OpenWrt device
ssh root@192.168.1.1

# Install the package
opkg install /tmp/my-package_1.0.0-1_mipsel_24kc.ipk

# Or upgrade if already installed
opkg upgrade /tmp/my-package_1.0.0-1_mipsel_24kc.ipk

# Force reinstall (overwrites existing)
opkg install --force-reinstall /tmp/my-package_1.0.0-1_mipsel_24kc.ipk
```

### Step 9: Verify Installation

```bash
# Check if package is installed
opkg list-installed | grep my-package

# List package files
opkg files my-package

# Check package status
opkg status my-package

# Run your program
my-package --help
```

### Step 10: Uninstall Package (if needed)

```bash
opkg remove my-package
```

---

## Scenario 2: Building Full Firmware

This scenario covers building a complete OpenWrt firmware image with custom packages included.
To check if the firmware version is compatible or not for your device, you can use the following link: https://firmware-selector.openwrt.org/

Some version are not in the official openwrt release, but you can find a snapshot in github repository.

### Step 1: Initial Setup

```bash
# Navigate to OpenWrt directory
cd /path/to/openwrt

# Update feeds
./scripts/feeds update -a
./scripts/feeds install -a
```

### Step 2: Add Custom Packages (Optional)

```bash
# Add your custom packages to package directory
cp -r /path/to/my-package package/custom/my-package
cp -r /path/to/another-package package/custom/another-package
```

### Step 3: Configure Target and Packages

```bash
# Start menuconfig
make menuconfig
```

**Important Configuration Steps:**

#### 3.1 Select Target System

Navigate through:
- `Target System` → Select your router's architecture (e.g., `MediaTek Ralink MIPS`, `Atheros AR7xxx/AR9xxx`, etc.)
- `Subtarget` → Select specific subtarget (e.g., `MT7621 based boards`)
- `Target Profile` → Select your exact device model

#### 3.2 Global Build Settings (Optional)

- `Global build settings` →
  - `Kernel build options` → Enable debugging if needed
  - `Package build options` → Configure package compilation options

#### 3.3 Select Base System Packages

Navigate categories:
- `Base system` → Essential packages (busybox, opkg, etc.)
- `Network` → Network tools (firewall, dnsmasq, etc.)
- `LuCI` → Web interface components
  - `Collections` → Select `luci` for full web interface
  - `Applications` → Additional web apps
  - `Themes` → UI themes

#### 3.4 Select Your Custom Packages

- Navigate to your package category (e.g., `Utilities`)
- Mark packages with `*` to include in firmware
- Mark with `M` to build as installable .ipk only

#### 3.5 Additional Useful Packages

```
Network:
  [*] curl
  [*] wget-ssl
  [*] openssh-sftp-server
  
Utilities:
  [*] bash
  [*] htop
  [*] nano
  
Kernel modules:
  [*] USB Support → kmod-usb-storage
  [*] Filesystems → kmod-fs-ext4
```

### Step 4: Save Configuration

```bash
# Save and exit menuconfig
# The configuration is saved to .config

# Optional: Save as custom config for future use
cp .config configs/my-custom-config
```

### Step 5: Download Required Sources

```bash
# Download all source packages
make download V=s

# This may take time depending on your internet connection
# Sources are cached in dl/ directory
```

### Step 6: Build the Firmware

```bash
# Clean previous builds (optional but recommended for first build)
make clean

# Full build with verbose output
make -j$(nproc) V=s

# Or specify number of threads manually
make -j4 V=s

# For quieter output (only errors shown)
make -j$(nproc)
```

**Build Time Expectations:**
- First build: 1-3 hours (depending on hardware)
- Subsequent builds: 10-30 minutes
- Single package rebuild: 1-5 minutes

### Step 7: Locate Firmware Images

After successful build, firmware images are located at:

```bash
bin/targets/<target>/<subtarget>/
```

Example:
```bash
bin/targets/ramips/mt7621/
├── openwrt-ramips-mt7621-device-name-squashfs-sysupgrade.bin
├── openwrt-ramips-mt7621-device-name-initramfs-kernel.bin
├── openwrt-ramips-mt7621-device-name-squashfs-factory.bin
└── sha256sums
```

**Image Types:**

- `*-sysupgrade.bin`: For upgrading existing OpenWrt installation
- `*-factory.bin`: For initial flash from stock firmware
- `*-initramfs-kernel.bin`: RAM-only boot (testing, recovery)

### Step 8: Flash Firmware to Device

#### Method 1: Web Interface (LuCI)

1. Access router web interface: `http://192.168.1.1`
2. Login as root
3. Navigate: `System` → `Backup / Flash Firmware`
4. Click `Flash new firmware image`
5. Upload `*-sysupgrade.bin` file
6. **Uncheck** "Keep settings" for clean install (optional)
7. Click `Flash image` and confirm

#### Method 2: Command Line (SCP + sysupgrade)

```bash
# Transfer firmware to device
scp bin/targets/ramips/mt7621/openwrt-*-sysupgrade.bin root@192.168.1.1:/tmp/

# SSH into device
ssh root@192.168.1.1

# Flash the firmware
sysupgrade -v /tmp/openwrt-*-sysupgrade.bin

# Or flash without keeping configuration
sysupgrade -n -v /tmp/openwrt-*-sysupgrade.bin
```

#### Method 3: TFTP Recovery (if device supports it)

```bash
# Configure your computer's IP to 192.168.1.10
sudo ip addr add 192.168.1.10/24 dev eth0

# Start TFTP server
sudo apt install tftpd-hpa
sudo cp openwrt-*-factory.bin /srv/tftp/firmware.bin
sudo systemctl start tftpd-hpa

# Put router in TFTP recovery mode (device-specific)
# Usually: Power off → Hold reset → Power on → Wait for TFTP request
```

### Step 9: Verify Firmware Installation

```bash
# SSH into device after reboot
ssh root@192.168.1.1

# Check OpenWrt version
cat /etc/openwrt_release

# Check installed packages
opkg list-installed

# Check kernel version
uname -a

# Check available storage
df -h
```

### Step 10: Post-Installation Configuration

```bash
# Update package lists (if internet available)
opkg update

# Install additional packages
opkg install luci-app-adblock

# Configure network settings
vi /etc/config/network

# Configure wireless
vi /etc/config/wireless

# Apply changes
/etc/init.d/network restart
```

---

## Advanced Build Techniques

### Build Specific Components

```bash
# Build only kernel
make target/linux/compile V=s

# Build specific package
make package/network/services/dnsmasq/compile V=s

# Build toolchain only
make toolchain/install V=s

# Build image from compiled packages
make target/install V=s
```

### Create Custom Feed

**Create `feeds.conf.custom`:**

```bash
cp feeds.conf.default feeds.conf

# Add custom feed
echo "src-link custom /path/to/custom/packages" >> feeds.conf

# Update feeds
./scripts/feeds update custom
./scripts/feeds install -a -p custom
```

### Image Builder

For faster rebuilds without recompiling:

```bash
# Download Image Builder
wget https://downloads.openwrt.org/releases/23.05.0/targets/ramips/mt7621/openwrt-imagebuilder-23.05.0-ramips-mt7621.Linux-x86_64.tar.xz

# Extract
tar xf openwrt-imagebuilder-*.tar.xz
cd openwrt-imagebuilder-*/

# Build custom image with specific packages
make image PROFILE="device-name" PACKAGES="luci nano htop my-package"
```

### SDK for Package Development

```bash
# Download SDK
wget https://downloads.openwrt.org/releases/23.05.0/targets/ramips/mt7621/openwrt-sdk-23.05.0-ramips-mt7621_gcc-12.3.0_musl.Linux-x86_64.tar.xz

# Extract
tar xf openwrt-sdk-*.tar.xz
cd openwrt-sdk-*/

# Add your package
cp -r /path/to/my-package package/

# Build
make package/my-package/compile V=s
```

---

## Troubleshooting

### Common Build Errors

#### Error: "Prerequisites not met"

```bash
# Re-run dependency installation
sudo apt install build-essential libncurses5-dev gawk git

# Check prerequisites
./scripts/feeds update -a
```

#### Error: "Hash mismatch"

```bash
# Remove cached download and retry
rm dl/package-name-*
make package/package-name/download V=s
make package/package-name/compile V=s
```

#### Error: Compilation fails

```bash
# Clean and rebuild
make package/my-package/clean
make package/my-package/compile V=sc 2>&1 | tee build.log

# Check build.log for detailed errors
```

#### Error: Out of disk space

```bash
# Clean build artifacts
make clean

# Remove downloaded sources (careful!)
# rm -rf dl/*

# Check disk usage
du -sh ./
```

### Package Installation Issues

#### Error: "Package architecture mismatch"

Ensure you compiled for the correct architecture:

```bash
# Check device architecture
ssh root@192.168.1.1 "opkg print-architecture"

# Rebuild for correct architecture
make menuconfig  # Select correct target
make package/my-package/compile V=s
```

#### Error: "Dependency not found"

```bash
# Install dependencies first
opkg update
opkg install libpthread libc

# Then install your package
opkg install /tmp/my-package.ipk
```

#### Error: "Cannot satisfy dependencies"

```bash
# Force installation (not recommended)
opkg install --force-depends /tmp/my-package.ipk

# Better: Build with correct dependencies in Makefile
```

### Network Transfer Issues

#### SSH connection refused

```bash
# Check if device is accessible
ping 192.168.1.1

# Try telnet if SSH is disabled
telnet 192.168.1.1

# Reset to failsafe mode (device-specific)
# Usually: Power on → Press reset when LED flashes
```

#### SCP permission denied

```bash
# Ensure SSH password/key authentication is set up
ssh root@192.168.1.1 "ls /tmp"

# Or use password authentication
scp -o PreferredAuthentications=password package.ipk root@192.168.1.1:/tmp/
```

---

## Build Environment Management

### Using Docker for Builds

```dockerfile
# Dockerfile for OpenWrt build environment
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    build-essential clang flex bison g++ gawk \
    gcc-multilib g++-multilib gettext git \
    libncurses5-dev libssl-dev python3-distutils \
    rsync unzip zlib1g-dev file wget

WORKDIR /openwrt
CMD ["/bin/bash"]
```

```bash
# Build Docker image
docker build -t openwrt-builder .

# Run container
docker run -it -v $(pwd):/openwrt openwrt-builder

# Inside container
git clone https://git.openwrt.org/openwrt/openwrt.git .
./scripts/feeds update -a
./scripts/feeds install -a
make menuconfig
make -j$(nproc)
```

### Parallel Builds

```bash
# Use all available cores
make -j$(nproc)

# Limit to 4 cores (useful on limited systems)
make -j4

# Single-threaded (for debugging)
make -j1 V=s
```

---

## Best Practices

### Development Workflow

1. **Use version control for packages**
   ```bash
   cd package/custom/my-package
   git init
   git add .
   git commit -m "Initial commit"
   ```

2. **Test before full build**
   ```bash
   # Build package only first
   make package/my-package/compile V=s
   
   # Test on device
   scp bin/packages/*/custom/my-package*.ipk root@192.168.1.1:/tmp/
   ssh root@192.168.1.1 "opkg install /tmp/my-package*.ipk"
   ```

3. **Keep build logs**
   ```bash
   make -j$(nproc) V=s 2>&1 | tee build-$(date +%Y%m%d-%H%M%S).log
   ```

### Configuration Management

```bash
# Save working configuration
./scripts/diffconfig.sh > my-config.diff

# Apply saved configuration
cp my-config.diff .config
make defconfig

# Share complete config
cp .config configs/my-device-config
```

### Cleaning Strategies

```bash
# Clean only build artifacts (keeps downloads)
make clean

# Clean everything including toolchain
make dirclean

# Clean specific package
make package/my-package/clean

# Clean kernel
make target/linux/clean
```

---

## Additional Resources

### Official Documentation

- OpenWrt Build System: https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem
- Package Creation: https://openwrt.org/docs/guide-developer/packages
- UCI Configuration: https://openwrt.org/docs/guide-user/base-system/uci

### Community Resources

- OpenWrt Forum: https://forum.openwrt.org/
- Developer Mailing List: https://lists.openwrt.org/
- GitHub Repository: https://github.com/openwrt/openwrt

### Quick Reference Commands

```bash
# Build System
make menuconfig                    # Configure build
make defconfig                     # Apply default configuration
make download                      # Download sources
make -j$(nproc) V=s               # Build with verbose output
make clean                         # Clean build artifacts

# Package Management
make package/name/download         # Download package sources
make package/name/compile V=s      # Compile specific package
make package/name/install          # Install package to staging
make package/name/clean            # Clean package build

# Feeds Management
./scripts/feeds update -a          # Update all feeds
./scripts/feeds install -a         # Install all packages from feeds
./scripts/feeds install pkg-name   # Install specific package

# Device Management
scp file.ipk root@192.168.1.1:/tmp/  # Transfer package
ssh root@192.168.1.1               # SSH to device
opkg update                        # Update package lists
opkg install /tmp/package.ipk      # Install package
opkg remove package-name           # Remove package
sysupgrade -v firmware.bin         # Flash firmware
```

---

## Conclusion

This guide covers the complete workflow for building OpenWrt packages and firmware. The modular nature of OpenWrt's build system allows you to work efficiently whether you're developing a single package or creating a complete custom firmware image.

**Key Takeaways:**

- **Scenario 1** (Individual packages): Fast iteration, easy testing, minimal build time
- **Scenario 2** (Full firmware): Complete customization, all packages integrated, longer build time

Choose the appropriate scenario based on your development needs. Start with individual package builds for development and testing, then move to full firmware builds for production deployment.