# Introduction

This is a documentation repository that ties together all the repositories concerned with a thin client that can be network booted.
In this context, a **thin client** is a lightweight operating system containing a small amount of tools only, compared to a full client such as a client running the Windows operating system.
There are a couple of advantages to running a thin client:

- Security: less attack surface, as fewer components are running
- Security: an always up-to-date operating system including applications, as the image can be rebuilt periodically
- Maintenance: the devices do not need to be managed and patched (e.g. SCCM or Intune with Windows)
- Cost: No licenses need to be payed
- Cost: Cheaper devices can be used, no full-blown laptops or PCs are needed. The hard disk can be omitted completely.

The thin client is **booted from the network** and **stateless**, which means that any device in the network can boot the thin client operating system, even if the device does not have a disk. During startup, the thin client image is loaded into RAM and booted from there. This puts two requirements on this solution: The clients need sufficient RAM (we have observed, that it's getting tight with 8GB RAM, so we recommend 16GB) and the network needs to be fast for adequate boot times.

The use case we've implemented this for is for browser-based tasks. Both in retail shops and in warehouses, most of the workflows are browser-only. This means, we do not require a full-blown operating system with complex programs such as development environments, Microsoft Excel or other programs that are installed directly on the client.

This is a step-by-step guide to spin up the netboot infrastructure as well as creating new images for the clients to boot from. In this scenario, you won't need to have a CI / CD Pipeline ready (which we do recommend). This will hopefully help you to get a better insight on how we have structured this project. Please note, that several commands can vary and that you might need to tweak some stuff here and there.

## Boot process

![Boot process](https://github.com/DigitecGalaxus/netbooting-thinclients/blob/main/docs/boot_process.svg "Boot process")

1. A new client, which is configured in BIOS to boot via network is booted.
2.  The client sends a DHCP broadcast to obtain it's IP and network booting information.
3. The DHCP server (in our example, an Opnsense device) responds with the IP as well as the information, which contains the network boot server's IP address and the file names of the initial bootloader, which the client should fetch from the network boot server.
4. The client fetches the initial bootloader from the network boot server
5. The client runs the initial bootloader, which contains instructions to load the menu.ipxe from the network boot server
6. The menu.ipxe contains the logic to show and boot various thin client images and allows the user to interact with the menu to select the desired image. If no selection is made, the default image is downloaded into RAM and booted. The files being downloaded are:
    - initrd (Initial Ramdisk)
    - kernel (Linux Kernel)
    - squashFS (Filesystem containing the OS)

## Repository Overview

This repository contains a few submodules. Please refer to the README in each of the repositories to understand what it does. Because submodules are used, pull the latest files and update the HEAD to be on the latest commit on the "main" branch.

```git
git clone https://github.com/DigitecGalaxus/netbooting-thinclients.git
cd netbooting-thinclients
git submodule update --init --recursive
git submodule foreach git pull origin main
```

## Prerequisites

For the network booting infrastructure to work, there are a few required services and servers.

- Gateway, to redirect PXE netboot requests (we are using an [opnsense gateway](https://opnsense.org/) instance). We won't go into detail how to prepare a gateway.
- a host/VM for the netboot-server (the services are deployed with docker-compose)
- a test client, which will boot the thin client via network
- a VM or buildpipeline to build the thinclient images (This can be done on your local computer too.)

## Prepare the netboot-Server

Setup a VM with plain ubuntu 20.04 LTS server VM in order to host the necessary service for the netbooting infrastructure. Follow this step-by-step guide to set it up. Reserve a static IP for the netboot server (e.g `192.168.1.101` ).

The following steps should be executed on the ubuntu VM.

***Note***
`192.168.1.101` will be used as the example netboot-server IP in this guide.

```sh
export netbootIP="192.168.1.101"
```

### 1. Install tools

```bash
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install -y git docker.io docker-compose jq curl
```

### 2. Add user

Either create a new user or use your existing non-root account in order to continue with the installation. We don't want to install the netboot infrastructure in the root context

```bash
username="netbootuser"
sudo adduser "$username"
```

### 3. Add user to the docker group

```bash
sudo gpasswd -a "$username" docker
```

After adding the user, switch to the newly created user:

```bash
su "$username"
```

### 4. Create a SSH key

The certficate is used by various scripts, to move the newly generated files between the correct destinations. Afterwards, add the public key to the authorized_hosts file.

```bash
mkdir -p /home/$USER/.ssh
openssl genrsa -out  /home/$USER/.ssh/netbootserver-priv.pem 4096
openssl rsa -in /home/$USER/.ssh/netbootserver-priv.pem -pubout > /home/$USER/.ssh/netbootserver-pub.pem
ssh-keygen -i -m PKCS8 -f /home/$USER/.ssh/netbootserver-pub.pem >> /home/$USER/.ssh/authorized_keys
```

### 5. Check out the netboot repository

As a preparation for the next step, check out the netboot repository on the ubuntu VM.

```bash
cd "/home/$USER"
git clone https://github.com/DigitecGalaxus/netboot
```

### 6. Build and deploy the services

***For this guide, the caching-servers and the monitoring is omitted. Therefore, remove or comment out the following services in the docker-compose.yaml first. Additionally, remove the depends_on of the netboot-build-main-ipxe-menus service***

**Note**: If your are interested in setting up a caching server for the network boot server (e.g. for networks with lots of clients), we recommend to check out the [caching-server repository](https://github.com/DigitecGalaxus/netboot-caching).

```yaml
...
#  netboot-syncer:
#    image: anymodconrst001dg.azurecr.io/planetexpress/netboot-sync:latest
#    container_name: netboot-syncer
#    environment: 
#      - devCachingServerIP
#    volumes:
#      - $HOME/netboot/assets:/syncing
#      - $HOME/netboot/caching_server_list.json:/root/caching_server_list.json
#      - $HOME/netboot/caching-server.pem:/root/.ssh/caching-server.pem
#    depends_on: 
#      - netboot-caching-server-fetcher
#    restart: unless-stopped

#  netboot-monitoring:
#    image: anymodconrst001dg.azurecr.io/planetexpress/netboot-monitoring:latest
#    container_name: netboot-monitoring
#    volumes:
#      - $HOME/netboot/caching_server_list.json:/root/caching_server_list.json
#    environment:
#      - influx_username=#{influx_username}#
#      - influx_password=#{influx_password}#
#      - influx_database=#{influx_database}#
#      - influx_url=#{influx_url}#
#      - netbootServerIP
#    depends_on:
#        - netboot-caching-server-fetcher
#    restart: unless-stopped

#  netboot-caching-server-fetcher:
#    image: anymodconrst001dg.azurecr.io/planetexpress/caching-server-fetcher:latest
#    container_name: netboot-caching-server-fetcher
#    environment:
#      - networksServiceURL=#{networksServiceURL}#
#    volumes:
#      - $HOME/netboot/caching_server_list.json:/work/caching_server_list.json
#    restart:
#      unless-stopped

# netboot-build-ipxe-mac-menus:
#    image: anymodconrst001dg.azurecr.io/planetexpress/netboot-ipxe-menu-generator:latest
#    container_name: netboot-build-ipxe-mac-menus
#    environment:
#      - netbootServerIP
#    volumes:
#      - $HOME/netboot/caching_server_list.json:/work/caching_server_list.json
#      - $HOME/netboot/config/menus:/work/menus
#    depends_on:
#      - netboot-caching-server-fetcher
#    restart:
#      unless-stopped
#    command: /work/buildIpxeMacMenus.sh

...
```

*The script to build and run the network booting services needs the following variables passed to it:*

- netbootServerIP
- pemFilePath
- devCachingServerIP (Optional, only if you want to deploy a cachingServer at a remote site)
- netbootServicesPullToken (Optional, only if you want to use your own registry)

To deploy the docker-containers, we do:

```bash
netbootingThinclientsPath=$(pwd)
cd "$netbootingThinclientsPath/netboot"
./run-compose.sh "192.168.1.101" "/home/$USER/.ssh/netbootserver-priv.pem"
```

The netboot-installation Path will be `/home/$USER/netboot`.

Verify that containers were started with

```sh
docker container ls
```

There should now be 3 containers in state "Up" and one container in state "Restarting".

### 7. Prepare the initial-bootloader

```bash
cd "$netbootingThinclientsPath/netboot/initial-bootloader"
./dockerbuild.sh
```

### 8. Copy initial bootloader to netboot volume

``` bash
mv ipxe32.efi /home/$USER/netboot/config/menus/
mv ipxe64.efi /home/$USER/netboot/config/menus/
mv undionly.kpxe /home/$USER/netboot/config/menus/
```

### 9. Configure DHCP Server

For clients to boot from the network, add the netboot-server to the DHCP-server as "next-server". We are using an OPNSense Gateway for our environment. Lookup the documentation for your DHCP-server device / software. Also make sure that the DHCP-server is running and is offering IP addresses to the test client.

On an OPNSense, go to `Services > DHCPv4 > {YOUR LAN}`

```undefined
- Enable network booting
    - set next-server: 192.168.1.101
    - set default bios filename: undionly.kpxe
    - set UEFI 32bit filename: uefi32.efi
    - set UEFI 64bit filename: uefi64.efi
```

### 10.  Boot the test client

Now the test client should be able to boot from the network. You can setup a new test client / VM in the same network as the netboot server, and boot the test client. If everything goes as expected, the test client will fail to boot with the following error:

```sh
tftp://192.168.1.101/menu.ipxe... no such file or directory
```

This is completely fine, as the menu.ipxe will be generated down the line of this guide.

## Setup the pre-requisites for building custom images

In this chapter, we will prepare and get any helper-tools, to finally build our ubuntu images and deploy them to the netboot-server.

### 1. Prepare squashfs-tools

Build the docker image for Squashfs. Go back to your git root and do the docker builds.

```sh
netbootingThinclientsPath=$(pwd)
cd "$netbootingThinclientsPath/squashfs-tools"
docker image build -t anymodconrst001dg.azurecr.io/planetexpress/squashfs-tools:latest .
```

### 2. Prepare Jinja2-Templating

```sh
cd "$netbootingThinclientsPath/jinja2-templating"
docker image build -t anymodconrst001dg.azurecr.io/planetexpress/jinja2-templating:latest .
```

### 3. Build the ubuntu-base image

```sh
cd "$netbootingThinclientsPath/ubuntu-base"
./build.sh
```

### 4. Get the private key to the netboot server

To proceed with the thin client image build, get the private key that you created above from `/home/$USER/.ssh/netbootserver-priv.pem`. For simplicity, move it to the same location but on the device, where the thin client image is built. It is also possible to build it on the netboot server, however this is not recommended. Some parts of the script assume that it is not the same location and use `scp` to transfer files to the netboot server.

Now everything should be ready, in order to continue with the juicy part of this project - the thinclient repo!

## Build custom thin client image

### 1. Get the latest kernel

*This script needs the following variables passed to it:*

- netbootServerIP
- pemFilePath
- netbootUsername

```sh
export netbootIP="192.168.1.101"
export netbootUser="netbootuser"
cd "$netbootingThinclientsPath/thinclients/kernel-updates"
./get-kernel-updates.sh "/home/$USER/.ssh/netbootserver-priv.pem" "$netbootIP" "$netbootUser"
```

### 2. Build the thin client

Change into the thinclients repository and build the image using the build.sh scripts. The output will be a squashFS file, which contains the file system which will be stored on the netboot server.

To be fully functional, the netboot server must be ready and listening on port 80. You can verify this by checking the status of the netboot-http container.

*This script needs the following variables passed to it:*

- netbootServerIP

```sh
cd "$netbootingThinclientsPath/thinclients/build"
./build.sh "$netbootIP"
squashfsAbsolutePath="$(pwd)/$(find . -name "*.squashfs" | head -1)"
```

This script will generate a squashfs file. It uses the current date, latest commit-SHA and branchname to generate the filename.

Note that this build will take a while, especially when the docker cache is empty.

### 3. Upload the generated "squashfs" file to the netboot server

*This script needs the following Variables passed to it:*

- pemFilePath
- squashfsAbsolutePath (this file should be available after executing the `./build.sh` from step 2. Pass the absolute path of it!)
- netbootServerIP
- netbootUsername
- folderToPromoteTo (either prod or dev)

```sh
cd "$netbootingThinclientsPath/thinclients/promote"
./promote.sh "/home/$USER/.ssh/netbootserver-priv.pem" "$squashfsAbsolutePath" "$netbootIP" "$netbootUser" "prod"
```

### 4. Generate the IPXE Menu based on the promoted Images

On the netboot server, wait for the container `netboot-build-main-ipxe-menus` to build the new menu.ipxe. It will do this periodically every two minutes. Verify the created menu.ipxe to contain this line:

```sh
# A line similar to the following should be present in the menu.ipxe (with imageName set to your squashfs filename)
set squash_url http://$netbootIP/prod/{{ imageName }}.squashfs
# Line should be contained in this file
cat /home/$USER/netboot/config/menus/menu.ipxe
```

### 5. Verify folder structure and generated files

When you have reached it this far, it looks very promising, that everything should be in place as expected! Now, you need to verify, if this is the case! To check if everything is in place, check the following structure on the netboot server at `/home/$USER/netboot`.

```undefined
├── assets
│   ├── dev
│   ├── kernels
│   │   ├── 5.11.0-16-generic
│   │   │   ├── initrd
│   │   │   └── vmlinuz
│   │   └── latest-kernel-version.json
│   └── prod
│       ├── 21-10-04-fix-resolve-issues-52b8ee8-kernel.json
│       └── 21-10-04-fix-resolve-issues-52b8ee8.squashfs
├── config
│   └── menus
│       ├── advancedmenu.ipxe
│       ├── ipxe32.efi
│       ├── ipxe64.efi
│       ├── menu.ipxe
│       ├── netinfo.ipxe
│       └── undionly.kpxe
```

Once those files are available on the netboot server, the containers will serve them via TFTP (Port 69) and HTTP (Port 80). The test client should now be able to boot the test client via network!

Go to your testclient and boot from PXE and enjoy a freshly built, stateless ubuntu OS!

This project has been brought to you by Planet Express @ Digitec Galaxus AG!

## Troubleshooting / Notes

- This Guide has been tested on a lab-environment with Microsoft Hyper-V
- It shows the manual approach to use the project. We highly recommend using a CI / CD Platform that will take care of all the steps above! You can check out the associated azure-pipelines.yml as reference.
- While there may be issues on the first attempt to build a Docker Image, make sure that you remove the temporarily generated files from your working directory when rebuilding the Docker Image.
- If you still have problems following this guide or get along with the setup, consider opening an issue. See [***Contribute***](#contribute)

## Contribute

No matter how small, we value every contribution! If you wish to contribute,

1. Please create an issue first - this way, we can discuss the feature and flesh out the nitty-gritty details
2. Fork the repository, implement the feature and submit a pull request
3. Your feature will be added once the pull request is merged
