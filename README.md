# Native Immich Installation Guide

This guide provides instructions and helper scripts to install [Immich](https://github.com/immich-app/immich) natively (without Docker) on a Proxmox LXC container. It’s an updated version of [Immich Native](https://github.com/arter97/immich-native/tree/master) adapted for a setup that uses:
- Proxmox and LXC containers
- An NVIDIA GeForce 1080ti for GPU transcoding and machine learning
- Custom builds for [libvips](https://github.com/libvips/libvips) and [sharp](https://www.npmjs.com/package/sharp) to support HEIC/HEIF thumbnails

*Note: Officially, Immich can be installed via Docker, but this guide avoids excessive virtualization.*

## Warning ⚠

*This guide is specifically tailored for a setup using Proxmox and LXC containers running Debian 12. Please ensure compatibility with your system before proceeding.*


## Table of Contents
- [Native Immich Installation Guide](#native-immich-installation-guide)
  - [Warning ⚠](#warning-)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Preparing the LXC Container](#preparing-the-lxc-container)
    - [NFS Setup](#nfs-setup)
    - [NVIDIA GPU Setup](#nvidia-gpu-setup)
  - [Installing Required Software](#installing-required-software)
    - [Node.js](#nodejs)
    - [PostgreSQL](#postgresql)
    - [Redis](#redis)
    - [FFMPEG (Jellyfin version)](#ffmpeg-jellyfin-version)
    - [ImageMagick](#imagemagick)
    - [libheif](#libheif)
    - [libvips](#libvips)
  - [Cloning and Setting Up Immich Native](#cloning-and-setting-up-immich-native)
    - [Environment Preparation](#environment-preparation)
    - [Database Setup](#database-setup)
    - [Immich Installation \& Configuration](#immich-installation--configuration)
  - [Using the Custom install.sh Script](#using-the-custom-installsh-script)
  - [Installation Completion \& Verification](#installation-completion--verification)
  - [Uninstallation Instructions](#uninstallation-instructions)
  - [Acknowledgements](#acknowledgements)

---

## Prerequisites

- Proxmox with LXC containers
- An NVIDIA GPU (GeForce 1080ti recommended)
- Basic Linux command-line knowledge

---

## Preparing the LXC Container
### NFS Setup
1. Install NFS common:
   ```bash
   sudo apt install nfs-common -y
   ```
2. Update `/etc/fstab` to enable auto-mount of your storage.

### NVIDIA GPU Setup
1. Install the NVIDIA driver (without the kernel module):
   ```bash
   ./NVIDIA-Linux-x86_64-550.120.run --no-kernel-module
   ```
2. Install CUDA:
   ```bash
   wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
   sudo dpkg -i cuda-keyring_1.1-1_all.deb
   sudo apt-get update
   sudo apt-get -y install cuda-toolkit-12-6
   echo 'export PATH=/usr/local/cuda-12.6/bin:$PATH' >> ~/.bashrc
   source ~/.bashrc
   nvcc --version
   ```

---

## Installing Required Software

### Node.js

1. Setup Node.js LTS:
   ```bash
   curl -fsSL https://deb.nodesource.com/setup_lts.x -o nodesource_setup.sh
   sudo -E bash nodesource_setup.sh
   sudo apt update
   sudo apt install nodejs -y
   node -v # Expect something like v22.14.0
   npm -v # Expect something like 10.9.2
   ```

### PostgreSQL

1. Install PostgreSQL and required packages:
   ```bash
   sudo apt install -y curl ca-certificates gnupg lsb-release
   curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /usr/share/keyrings/postgresql-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/postgresql-keyring.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list
   sudo apt update
   sudo apt install -y postgresql-17 postgresql-17-pgvector
   sudo systemctl enable postgresql
   sudo systemctl start postgresql
   ```

### Redis

1. Install and verify Redis:
   ```bash
   sudo apt install -y redis-server
   sudo systemctl enable redis-server
   sudo systemctl start redis-server
   redis-cli ping # Should return "PONG"
   ```

### FFMPEG (Jellyfin version)

1. Install Jellyfin ffmpeg:
   ```bash
   sudo apt install -y curl gnupg apt-transport-https
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://repo.jellyfin.org/jellyfin_team.gpg.key | gpg --dearmor -o /etc/apt/keyrings/jellyfin.gpg
   echo "deb [signed-by=/etc/apt/keyrings/jellyfin.gpg] https://repo.jellyfin.org/debian bookworm main" | sudo tee /etc/apt/sources.list.d/jellyfin.list
   sudo apt update
   sudo apt install -y jellyfin-ffmpeg7
   sudo ln -sf /usr/lib/jellyfin-ffmpeg/ffmpeg /usr/bin/ffmpeg
   sudo ln -sf /usr/lib/jellyfin-ffmpeg/ffprobe /usr/bin/ffprobe
   ffmpeg -version # Check for "Jellyfin" in the version output
   ```

### ImageMagick

1. Install ImageMagick development libraries:
   ```bash
   sudo apt install -y imagemagick libmagickcore-dev libmagickwand-dev
   ```

### libheif
   ```bash
   cd ~
   wget https://github.com/strukturag/libheif/releases/download/v1.19.5/libheif-1.19.5.tar.gz
   tar -xzf libheif-1.19.5.tar.gz
   cd libheif-1.19.5
   mkdir build
   cd build
   cmake .. -DCMAKE_INSTALL_PREFIX=/usr
   make
   sudo make install
   sudo ldconfig
   ```

### libvips

1. **Install Requirements:**
   ```bash
   sudo apt update
   sudo apt install -y git cmake meson ninja-build build-essential pkg-config \
   libglib2.0-dev libjpeg-dev libpng-dev libtiff-dev libexif-dev libxml2-dev liborc-0.4-dev \
   libaom-dev libarchive-dev libcairo2-dev libcgif-dev libexpat1-dev libffi-dev libfontconfig1-dev libfreetype-dev \
   libfribidi-dev libharfbuzz-dev libimagequant-dev liblcms2-dev libpango1.0-dev libpixman-1-dev \
   librsvg2-dev libspng-dev libwebp-dev zlib1g-dev libcgif-dev libcfitsio-dev libopenslide-dev libmatio-dev \
   libpoppler-glib-dev
   ```

1. **Compile nifti_clib:**
   ```bash
   cd ~
   git clone https://github.com/neurolabusc/nifti_clib.git
   cd nifti_clib
   mkdir build && cd build
   cmake -DCMAKE_INSTALL_PREFIX=/usr/local ..
   make -j$(nproc)
   sudo make install
   ```

2. **Compile libvips:**
   ```bash
   cd ~
   git clone https://github.com/libvips/libvips.git
   cd libvips
   sudo meson setup build --prefix=/usr --buildtype=release \
   -Dopenjpeg=enabled -Dimagequant=enabled -Dheif=enabled -Dpoppler=enabled \
   -Drsvg=enabled -Dopenexr=enabled -Dopenslide=enabled -Dmatio=enabled \
   -Dnifti=enabled -Dcfitsio=enabled -Dcgif=enabled -Dmagick=enabled -Dmodules=enabled
   ninja -C build
   sudo ninja -C build install
   sudo ldconfig
   ```

3. **Verify Installation:**
   ```bash
   vips --version
   pkg-config --modversion vips
   ```

---

## Cloning and Setting Up Immich Native

### Environment Preparation

1. **Clone the Repository:**
  ```bash
  cd ~
  git clone https://github.com/arter97/immich-native
  cd immich-native
  ```

2. **Add Immich User:**
  ```bash
  sudo adduser --home /var/lib/immich/home --shell=/sbin/nologin --no-create-home --disabled-password --disabled-login immich
  sudo mkdir -p /var/lib/immich
  sudo chown immich:immich /var/lib/immich
  sudo chmod 700 /var/lib/immich
  ```

### Database Setup

1. **Create the Database and User:**
   ```bash
   sudo -u postgres psql
   ```

   Then inside the PostgreSQL shell, run:
   ```sql
   create database immich;
   create user immich_user with encrypted password 'YOUR_STRONG_RANDOM_PW';
   grant all privileges on database immich to immich_user;
   ALTER USER immich_user WITH SUPERUSER;
   CREATE EXTENSION IF NOT EXISTS vector;
   \q
   ```

### Immich Installation & Configuration

1. **Install Additional Dependencies:**
   ```bash
   sudo apt install --no-install-recommends -y \
     python3-venv python3-dev uuid-runtime autoconf build-essential unzip jq perl \
     libnet-ssleay-perl libio-socket-ssl-perl libcapture-tiny-perl libfile-which-perl \
     libfile-chdir-perl libpkgconfig-perl libffi-checklib-perl libtest-warnings-perl \
     libtest-fatal-perl libtest-needs-perl libtest2-suite-perl libsort-versions-perl \
     libpath-tiny-perl libtry-tiny-perl libterm-table-perl libany-uri-escape-perl \
     libmojolicious-perl libfile-slurper-perl liblcms2-2 wget
   ```

2. **Define the Application Location:**
   Choose `/opt/immich` as the installation directory:
   ```bash
   sudo mkdir /opt/immich
   ```

3. **Prepare Required Media Directories:**
   Immich expects specific folders (refer to [Immich System Integrity](https://immich.app/docs/administration/system-integrity)). Create them as follows:
   ```bash
   for dir in encoded-video library upload profile thumbs backups; do
     sudo mkdir -p /media/photos/$dir
     sudo touch /media/photos/$dir/.immich
   done
   ```

4. **Configure the Environment File:**
   Update the `env` file with:
   - `DB_USERNAME=immich_user`
   - `DB_PASSWORD=YOUR_STRONG_RANDOM_PW`
   - `IMMICH_HOST=0.0.0.0`
   - `UPLOAD_LOCATION=/media/photos`
   - `IMMICH_MEDIA_LOCATION=/media/photos`

   Then copy it to the installation directory:
   ```bash
   sudo cp env /opt/immich/
   sudo chown immich:immich /opt/immich/env
   ```

---
## Using the Custom install.sh Script 

Your local `install.sh` script already includes all the necessary modifications compared to the original repository. These customizations are:

- **Application Location:** The script is set to use `/opt/immich` as the installation directory:
  ```bash 
  IMMICH_PATH=/opt/immich 
  ```

- **Install Sharp with Custom Options:** Instead of the default command, the script installs Sharp using:
  ```bash 
  npm install node-addon-api node-gyp
  SHARP_LIBVIPS_EXTERNAL=1 PKG_CONFIG_PATH=/usr/lib/x86_64-linux-gnu/pkgconfig npm install --build-from-scratch sharp 
  ```

- **Network Binding:** The section that forces binding to `127.0.0.1` is commented out, allowing Immich to listen on all interfaces.

Simply run your custom script to proceed with the installation: 
```bash 
./install.sh 
``` 

--- 

## Installation Completion & Verification 

After running the installation script:
- The Immich application should be available at [http://localhost:2283](http://localhost:2283).
- Immich will automatically start at system boot.
- You can now install the mobile app from your preferred app store to enjoy all its features.

--- 

## Uninstallation Instructions 

To uninstall Immich, follow these steps:

1. **Remove Systemd Services:** 
   ```bash 
   systemctl list-unit-files --type=service | grep "^immich" | while read i unused; do
     sudo systemctl stop $i
     sudo systemctl disable $i
   done

   sudo rm /lib/systemd/system/immich*.service
   sudo systemctl daemon-reload
   ```

2. **Remove Immich Files:** 
   ```bash 
   sudo rm -rf /opt/immich 
   ```

3. **Delete Immich User:** 
   ```bash 
   sudo deluser immich 
   ```

4. **Remove the Immich Database:** 
   ```bash 
   sudo -u postgres psql 
   ```

   Then run: 
   ```sql 
   drop database immich; 
   drop user immich_user; 
   \q 
   ```

5. **Optionally Remove Dependencies:** 
   Review `/var/log/apt/history.log` to remove packages installed specifically for Immich.

--- 

## Acknowledgements 

Thanks to the original [arter97/immich-native](https://github.com/arter97/immich-native/tree/master) project for providing the base work. Contributions and improvements are welcome via pull requests.