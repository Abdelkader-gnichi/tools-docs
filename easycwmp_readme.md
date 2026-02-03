# EasyCWMP - Complete Build and Installation Guide

## Table of Contents

- [Introduction](#introduction)
- [System Requirements](#system-requirements)
- [Prerequisites Installation](#prerequisites-installation)
  - [Fedora/RHEL](#fedora-rhel-based-systems)
  - [Ubuntu/Debian](#ubuntu-debian-based-systems)
- [Building Dependencies from Source](#building-dependencies-from-source)
  - [microxml Library](#1-microxml-library)
  - [libuci Library](#2-libuci-library)
- [Building EasyCWMP](#building-easycwmp)
  - [Source Preparation](#source-preparation)
  - [Configuration and Compilation](#configuration-and-compilation)
  - [Installation](#installation)
- [Post-Installation Configuration](#post-installation-configuration)
- [Troubleshooting](#troubleshooting)
- [Usage](#usage)
- [License](#license)

---

## Introduction

EasyCWMP is a lightweight TR-069 (CWMP - CPE WAN Management Protocol) client implementation for embedded devices and Linux systems. This guide provides comprehensive instructions for building and installing EasyCWMP on both Fedora/RHEL and Ubuntu/Debian-based systems.

## System Requirements

- **Operating System**: Fedora 40+, RHEL 9+, Ubuntu 20.04+, Debian 11+
- **Architecture**: x86_64, ARM
- **Disk Space**: ~100 MB for dependencies and build artifacts
- **RAM**: Minimum 512 MB
- **Compiler**: GCC 9.0+ or Clang 10.0+

---

## Prerequisites Installation

### Fedora/RHEL-based Systems

```bash
# Update system packages
sudo dnf update -y

# Install build tools
sudo dnf install -y \
    gcc \
    make \
    automake \
    autoconf \
    libtool \
    pkg-config \
    git

# Install development libraries
sudo dnf install -y \
    libcurl-devel \
    openssl-devel \
    openssl-devel-engine \
    json-c-devel \
    zlib-devel

# Install CMake (for building libuci)
sudo dnf install -y cmake
```

### Ubuntu/Debian-based Systems

```bash
# Update package lists
sudo apt update

# Install build tools
sudo apt install -y \
    build-essential \
    gcc \
    make \
    automake \
    autoconf \
    libtool \
    pkg-config \
    git

# Install development libraries
sudo apt install -y \
    libcurl4-openssl-dev \
    libssl-dev \
    libjson-c-dev \
    zlib1g-dev

# Install CMake (for building libuci)
sudo apt install -y cmake
```

---

## Building Dependencies from Source

EasyCWMP requires two libraries that are typically not available in standard repositories: **microxml** and **libuci**.

### 1. microxml Library

microxml is a lightweight XML parsing library used by EasyCWMP for TR-069 message processing.

**Clone from the correct repository:**

```bash
# Navigate to temporary directory
cd /tmp

# Remove any existing microxml directory
rm -rf microxml

# Clone the microxml repository from pivasoftware
git clone https://github.com/pivasoftware/microxml.git
cd microxml

# The repository includes pre-generated configure script
# If configure doesn't exist or needs regeneration, run:
# autoreconf -fi

# Configure and build
./configure
make
sudo make install
sudo ldconfig
```

**Alternative build method if configure script is missing:**

```bash
cd /tmp/microxml

# Install autotools if needed
sudo dnf install autoconf automake libtool  # Fedora/RHEL
# or
sudo apt install autoconf automake libtool  # Ubuntu/Debian

# Generate build files (warnings are normal)
aclocal
autoconf
automake --add-missing

# Configure and build
./configure
make
sudo make install
sudo ldconfig
```

**Verify installation:**

```bash
ls /usr/local/lib/libmicroxml* /usr/local/include/microxml.h
pkg-config --modversion microxml
```

### 2. libubox Library (Dependency for libuci)

libubox is required by libuci and must be built first.

```bash
# Navigate to temporary directory
cd /tmp

# Clone the libubox repository
git clone https://git.openwrt.org/project/libubox.git
cd libubox

# Build using CMake
mkdir build && cd build
cmake -DBUILD_LUA=OFF -DBUILD_EXAMPLES=OFF ..
make
sudo make install
sudo ldconfig
```

**Verify installation:**

```bash
ls /usr/local/lib/libubox.so /usr/local/include/libubox/
```

### 3. libuci Library

UCI (Unified Configuration Interface) is a configuration management library from OpenWrt.

**Important:** Before building libuci, ensure libubox is installed (see above).

```bash
# Navigate to temporary directory
cd /tmp

# Remove any previous incomplete builds
rm -rf uci

# Clone the UCI repository
git clone https://git.openwrt.org/project/uci.git
cd uci

# Build using CMake (disable Lua bindings to avoid Lua dependency issues)
mkdir build && cd build
cmake -DBUILD_LUA=OFF ..
make
sudo make install
sudo ldconfig
```

**Verify installation:**

```bash
# Check library location (varies by system)
ls /usr/local/lib64/libuci.so /usr/local/include/uci.h
# or on some systems:
ls /usr/local/lib/libuci.so /usr/local/include/uci.h
```

**Note:** If you encounter errors about missing `libubox` during cmake configuration, ensure libubox was successfully installed first.

### Configure Library Path

Ensure the system can find the newly installed libraries:

**For Fedora/RHEL:**

```bash
echo "/usr/local/lib64" | sudo tee /etc/ld.so.conf.d/local-libs.conf
echo "/usr/local/lib" | sudo tee -a /etc/ld.so.conf.d/local-libs.conf
sudo ldconfig
```

**For Ubuntu/Debian:**

```bash
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/local-libs.conf
sudo ldconfig
```

**Configure pkg-config path:**

```bash
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/lib64/pkgconfig:$PKG_CONFIG_PATH
echo 'export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/lib64/pkgconfig:$PKG_CONFIG_PATH' >> ~/.bashrc
# or for zsh users:
echo 'export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/lib64/pkgconfig:$PKG_CONFIG_PATH' >> ~/.zshrc
```

---

## Building EasyCWMP

### Source Preparation

1. **Clone the repository:**

```bash
cd ~/
git clone <easycwmp-repository-url>
cd PMS/contrib/easycwmp
```

2. **Apply necessary patches:**

The EasyCWMP source requires several modifications for compatibility with modern systems.

**Fix configure.ac (duplicate AM_INIT_AUTOMAKE):**

```bash
# Backup the original file
cp configure.ac configure.ac.bak

# Edit configure.ac - merge duplicate AM_INIT_AUTOMAKE calls
# Change lines 2-4 from:
#   AM_INIT_AUTOMAKE
#   AC_CONFIG_SRCDIR([src/easycwmp.c])
#   AM_INIT_AUTOMAKE([subdir-objects])
# To:
#   AM_INIT_AUTOMAKE([subdir-objects])
#   AC_CONFIG_SRCDIR([src/easycwmp.c])

sed -i '2s/AM_INIT_AUTOMAKE/AM_INIT_AUTOMAKE([subdir-objects])/' configure.ac
sed -i '4d' configure.ac
```

**Add missing include to easyipc_cli.c:**

```bash
sed -i '/#include <string.h>/a #include <unistd.h>' src/easyipc_cli.c
```

**Fix JSON-C linking for easyipc:**

```bash
# Edit bin/Makefile.am to add JSON-C library
sed -i '/^easyipc_LDADD =/a \\t$(LIBJSON_LIBS)' bin/Makefile.am
```

### Configuration and Compilation

```bash
# Set required compiler flags
export CFLAGS="-I/usr/local/include -D_XOPEN_SOURCE=700 -D_GNU_SOURCE -DENABLE_CHANGEDUSTATE"
export LDFLAGS="-L/usr/local/lib64 -L/usr/local/lib"

# Generate build system
autoreconf -fi

# Configure the build
./configure --enable-debug --enable-devel --enable-jsonc

# Compile
make

# Verify binaries were created
ls -lh bin/easycwmpd bin/easyipc
file bin/easycwmpd bin/easyipc
```

**Expected output:**

```
bin/easycwmpd: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked
bin/easyipc:   ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked
```

### Installation

```bash
# Install binaries to /usr/local/bin
sudo make install

# Verify installation
which easycwmpd easyipc
easycwmpd --help
```

---

## Post-Installation Configuration

### 1. Create Configuration Directory

```bash
sudo mkdir -p /etc/easycwmp
```

### 2. Library Path Configuration

If you encounter "error while loading shared libraries" errors, ensure the library path is properly configured:

```bash
# Verify library is in cache
ldconfig -p | grep -E "libuci|libmicroxml"

# If not found, update ld.so.conf and refresh cache
sudo ldconfig -v
```

### 3. Environment Variables (Optional)

For convenience, add these to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
# Library paths
export LD_LIBRARY_PATH=/usr/local/lib64:/usr/local/lib:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:/usr/local/lib64/pkgconfig:$PKG_CONFIG_PATH

# Compiler flags for future builds
export CFLAGS="-I/usr/local/include"
export LDFLAGS="-L/usr/local/lib64 -L/usr/local/lib"
```

Apply changes:

```bash
source ~/.bashrc
# or
source ~/.zshrc
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **"autoreconf: command not found"**

```bash
# Fedora/RHEL
sudo dnf install automake autoconf

# Ubuntu/Debian
sudo apt install automake autoconf
```

#### 2. **"aclocal: command not found"**

```bash
# Install automake (provides aclocal)
# Fedora/RHEL
sudo dnf install automake

# Ubuntu/Debian
sudo apt install automake
```

#### 3. **"Package 'microxml' not found"**

Ensure microxml is properly installed and pkg-config can find it:

```bash
# Check installation
ls /usr/local/lib/libmicroxml*
ls /usr/local/include/microxml.h

# Verify pkg-config path
echo $PKG_CONFIG_PATH
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH

# Test pkg-config
pkg-config --exists microxml && echo "Found" || echo "Not found"
pkg-config --modversion microxml
```

If the library exists but pkg-config can't find it, ensure the .pc file is in the right location:

```bash
ls /usr/local/lib/pkgconfig/microxml.pc
```

#### 4. **"libubox/blobmsg.h: No such file or directory" when building libuci**

This means libubox needs to be built first:

```bash
cd /tmp
rm -rf libubox
git clone https://git.openwrt.org/project/libubox.git
cd libubox
mkdir build && cd build
cmake -DBUILD_LUA=OFF -DBUILD_EXAMPLES=OFF ..
make
sudo make install
sudo ldconfig
```

Then rebuild libuci:

```bash
cd /tmp
rm -rf uci
git clone https://git.openwrt.org/project/uci.git
cd uci
mkdir build && cd build
cmake -DBUILD_LUA=OFF ..
make
sudo make install
sudo ldconfig
```

#### 5. **"uci.h: No such file or directory"**

Build and install libuci from source (see [Building Dependencies](#3-libuci-library)). Ensure libubox is installed first.

#### 6. **"openssl/engine.h: No such file or directory"**

This occurs with OpenSSL 3.x. Install the engine development package:

```bash
# Fedora/RHEL
sudo dnf install openssl-devel-engine

# Ubuntu/Debian (usually included in libssl-dev)
sudo apt install libssl-dev
```

#### 7. **"implicit declaration of function 'strptime'"**

Add POSIX feature test macros to CFLAGS:

```bash
export CFLAGS="$CFLAGS -D_XOPEN_SOURCE=700 -D_GNU_SOURCE"
```

#### 8. **"implicit declaration of function 'backup_add_change_du_state'"**

Enable the CHANGEDUSTATE feature:

```bash
export CFLAGS="$CFLAGS -DENABLE_CHANGEDUSTATE"
```

#### 9. **"error while loading shared libraries: libuci.so"**

Update the library cache:

```bash
sudo ldconfig
# If still not found, add the path:
echo "/usr/local/lib64" | sudo tee -a /etc/ld.so.conf.d/local-libs.conf
sudo ldconfig
```

#### 10. **easyipc linking error with JSON-C**

Ensure the Makefile.am includes JSON-C for easyipc:

```bash
grep -A2 "easyipc_LDADD" bin/Makefile.am
# Should include $(LIBJSON_LIBS)
```

#### 11. **CMake errors about Lua when building libuci**

If you see errors about Lua not being found, ensure you're disabling Lua support:

```bash
cd /tmp/uci/build
rm -rf *  # Clean previous build
cmake -DBUILD_LUA=OFF ..
make
```

#### 12. **"Repository not found" when cloning microxml**

Use the pivasoftware repository instead:

```bash
git clone https://github.com/pivasoftware/microxml.git
# instead of
# git clone https://github.com/freenetconf/microxml.git
```

#### 13. **Authentication errors when cloning from GitHub**

If you encounter password authentication errors:

```bash
# Use HTTPS without authentication for public repos
git clone https://github.com/pivasoftware/microxml.git

# Or use SSH if you have SSH keys set up
git clone git@github.com:pivasoftware/microxml.git
```

---

## Usage

### Basic Commands

**Start the EasyCWMP daemon:**

```bash
sudo easycwmpd
```

**Use the IPC client:**

```bash
easyipc <command>
```

### Configuration

EasyCWMP uses UCI (Unified Configuration Interface) for configuration management. Configuration files are typically located in `/etc/easycwmp/`.

**Example basic configuration:**

```
config cwmp
    option enabled '1'
    option acs_url 'http://your-acs-server:7547/'
    option acs_username 'username'
    option acs_password 'password'
    option interface 'eth0'
```

### Running as a Service

**Create a systemd service file** (`/etc/systemd/system/easycwmpd.service`):

```ini
[Unit]
Description=EasyCWMP TR-069 Client
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/easycwmpd
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Enable and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable easycwmpd
sudo systemctl start easycwmpd
sudo systemctl status easycwmpd
```

---

## Complete Build Script

For automated installation, here's a complete build script:

```bash
#!/bin/bash
set -e

echo "====================================="
echo "EasyCWMP Complete Build Script"
echo "====================================="

# Detect OS
if [ -f /etc/fedora-release ] || [ -f /etc/redhat-release ]; then
    OS="fedora"
    PKG_MGR="dnf"
    LIB_PATH="/usr/local/lib64"
elif [ -f /etc/debian_version ]; then
    OS="debian"
    PKG_MGR="apt"
    LIB_PATH="/usr/local/lib"
else
    echo "Unsupported OS"
    exit 1
fi

echo "Detected OS: $OS"

# Install prerequisites
echo "Installing prerequisites..."
if [ "$OS" = "fedora" ]; then
    sudo $PKG_MGR install -y gcc make automake autoconf libtool pkg-config git cmake \
        libcurl-devel openssl-devel openssl-devel-engine json-c-devel zlib-devel
else
    sudo $PKG_MGR update
    sudo $PKG_MGR install -y build-essential gcc make automake autoconf libtool \
        pkg-config git cmake libcurl4-openssl-dev libssl-dev libjson-c-dev zlib1g-dev
fi

# Build microxml
echo "Building microxml..."
cd /tmp
rm -rf microxml
git clone https://github.com/pivasoftware/microxml.git
cd microxml

# Use existing configure script or generate if needed
if [ ! -f configure ]; then
    aclocal
    autoconf
    automake --add-missing
fi

./configure
make
sudo make install

# Build libubox (dependency for libuci)
echo "Building libubox..."
cd /tmp
rm -rf libubox
git clone https://git.openwrt.org/project/libubox.git
cd libubox
mkdir -p build && cd build
cmake -DBUILD_LUA=OFF -DBUILD_EXAMPLES=OFF ..
make
sudo make install

# Build libuci
echo "Building libuci..."
cd /tmp
rm -rf uci
git clone https://git.openwrt.org/project/uci.git
cd uci
mkdir -p build && cd build
cmake -DBUILD_LUA=OFF ..
make
sudo make install

# Configure library paths
echo "Configuring library paths..."
echo "$LIB_PATH" | sudo tee /etc/ld.so.conf.d/local-libs.conf
if [ "$OS" = "fedora" ]; then
    echo "/usr/local/lib" | sudo tee -a /etc/ld.so.conf.d/local-libs.conf
fi
sudo ldconfig

# Set environment variables
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$LIB_PATH/pkgconfig:$PKG_CONFIG_PATH
export CFLAGS="-I/usr/local/include -D_XOPEN_SOURCE=700 -D_GNU_SOURCE -DENABLE_CHANGEDUSTATE"
export LDFLAGS="-L$LIB_PATH -L/usr/local/lib"

# Build EasyCWMP
echo "Building EasyCWMP..."
cd ~/PMS/contrib/easycwmp

# Apply patches
cp configure.ac configure.ac.bak
sed -i '2s/AM_INIT_AUTOMAKE/AM_INIT_AUTOMAKE([subdir-objects])/' configure.ac
sed -i '4d' configure.ac
sed -i '/#include <string.h>/a #include <unistd.h>' src/easyipc_cli.c
sed -i '/^easyipc_LDADD =/a \\t$(LIBJSON_LIBS)' bin/Makefile.am

# Build
autoreconf -fi
./configure --enable-debug --enable-devel --enable-jsonc
make
sudo make install

# Create config directory
sudo mkdir -p /etc/easycwmp

echo "====================================="
echo "Build completed successfully!"
echo "Binaries installed to /usr/local/bin/"
echo "====================================="
easycwmpd --help
```

---

## License

EasyCWMP is licensed under the GNU General Public License v2.0 or later. See the COPYING file in the source distribution for details.

---

## Support and Contributing

For issues, questions, or contributions, please refer to the project's repository or contact the maintainers.

**Project Information:**

- **Version**: 5.9.0
- **Authors**: Mohamed Kallel, Anis Ellouze (PIVA SOFTWARE)
- **Website**: www.pivasoftware.com

---

## Additional Resources

- TR-069 Specification: [Broadband Forum TR-069](https://www.broadband-forum.org/technical/download/TR-069.pdf)
- OpenWrt UCI Documentation: [UCI Configuration](https://openwrt.org/docs/guide-user/base-system/uci)
- microxml Documentation: Included in source repository

---

_Last Updated: November 2025_
_Tested on: Fedora 43, Ubuntu 22.04_
