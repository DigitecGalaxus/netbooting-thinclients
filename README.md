# Introduction

This repository enables netbooting lightweight thinclient operating systems. A **thinclient** is a minimal OS with essential tools only, unlike full operating systems like Windows.

## Why thinclients?

**Security Benefits:**
- Reduced attack surface with fewer running components
- Always up-to-date OS and applications through periodic image rebuilds

**Cost & Maintenance:**
- No OS or EDR licensing costs
- No device management overhead (SCCM/Intune)
- Cheaper hardware requirements - no local storage needed
- Simplified maintenance with centralized updates

**Technical Requirements:**
- **RAM:** 16GB recommended (8GB minimum, but performance may suffer)
- **Network:** Fast network essential for reasonable boot times
- **Use Case:** Optimized for browser-based workflows in retail and warehouse environments

## How It Works

Thinclients are **network-booted** and **stateless** - any device can boot the OS without local storage. The entire image loads into RAM during startup, making devices completely interchangeable.

This guide covers setting up the netboot infrastructure and creating custom thinclient images. While we recommend a CI/CD pipeline for production, this manual approach helps you understand the project structure.

### Boot Process

![Boot process](https://raw.githubusercontent.com/DigitecGalaxus/netbooting-thinclients/main/docs/boot_process.svg "Boot process")

1. **DHCP Discovery**: Client requests IP and boot information via DHCP
2. **Boot Server Response**: DHCP server provides network boot server details and bootloader filenames
3. **Bootloader Download**: Client fetches initial bootloader via TFTP
4. **Menu Loading**: Bootloader loads iPXE menu from network boot server
5. **Image Selection**: User selects image or default loads automatically
6. **OS Download**: Client downloads and boots:
   - `initrd` (Initial Ramdisk)
   - `kernel` (Linux Kernel) 
   - `squashfs` (Compressed filesystem containing the OS)

How we accomplished this, can be found in our netboot submodule; see below for instructions.
For more information about netbooting concepts, see [netboot.xyz](https://netboot.xyz/).

## Repository Structure

This repository contains several components as submodules:

```
├── netboot/               # Network boot infrastructure
│   └── netboot-services/  # Docker services (TFTP, HTTP, menu generation)
└── thinclient-base/       # Thinclient image building tools
```

### Getting Started

```bash
git clone https://github.com/DigitecGalaxus/netbooting-thinclients.git
cd netbooting-thinclients
git submodule update --init --recursive
git submodule foreach git pull origin main
```

## Building Thinclient Images

**High-level process:**
1. Customize the base Ubuntu image with required packages and configurations
2. Extract kernel and initrd from the customized system
3. Compress the filesystem using SquashFS
4. Deploy artifacts to the netboot server

**Detailed instructions:** See [`thinclient-base/`](./thinclient-base/) subdirectory for build scripts and customization options.

## Setting Up Netboot Infrastructure

**High-level process:**
1. Deploy netboot services (TFTP, HTTP, menu generator) using Docker Compose
2. Configure DHCP server to point to netboot server
3. Place thinclient artifacts in appropriate directories
4. Menu is automatically generated from available images

**Detailed instructions:** See [`netboot/`](./netboot/) subdirectory for service configuration and deployment.

This project has been brought to you by Planet Express @ Digitec Galaxus AG!

## Troubleshooting / Notes

- Our setup consists of many separate components. It's important to understand the core concepts of all these components in order to understand how to plug them together. Our setup serves only as an inspiration. Feel free to pick and choose what you see fit.

## Contribute

No matter how small, we value every contribution! If you wish to contribute,

1. Please create an issue first - this way, we can discuss the feature and flesh out the nitty-gritty details
2. Fork the repository, implement the feature and submit a pull request
3. Your feature will be added once the pull request is merged
