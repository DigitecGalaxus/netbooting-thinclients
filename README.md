# Introduction

This is a step-by-step guide to spin up the netboot infrastructure as well as creating new images for the clients to boot from. In this scenario, you won't need to have a CI / CD Pipeline ready (which we do recommend). This will hopefully help you to get a better insight on how we have structured this project. Please Note, that several commands can vary and that you might need to tweak some stuff here and there.
\
&nbsp;

This image should give you an idea of the whole process, which you are going to tackle within this project.

![Netboot installation and usage flow](https://github.com/DigitecGalaxus/netbooting-thinclients/blob/main/docs/installation_flow.png "Installation and usage Flow")

## Repository Overview

Our Main repository consists of a few submodules. Please check out the README in each of the repositories to understand what it does. We will assume that you are already in the current working directory of the git repo.

```cmd
.
├── CONTRIBUTORS.txt
├── README.md
├── jinja2-templating
├── netboot
├── squashfs-tools
└── ubuntu-base
```

Because we are using submodules, we need to pull the latest files and update our HEAD to be on the latest commit on the "main" branch.

```git
git clone https://github.com/DigitecGalaxus/netbooting-thinclients.git
cd netbooting-thinclients
git submodule update --init --recursive
git submodule foreach git pull origin main
```

If you don't want to work with submodules, we recommend checking out [this gist](https://github.com/DigitecGalaxus/netbooting-thinclients), in order to convert the submodules to local folders in a single repository.
\
&nbsp;

## Prerequisites

For the whole Setup to work, there are a few required services and servers, in order to fully support this project

***NOTE: In this guide we won't setup a caching Server and the monitoring. Therefore you will see on some parts of this guide, that we need to modify a few files***

- Gateway, to redirect PXE Netboot Requests (we are using an [opnsense gateway](https://opnsense.org/) instance)
- a Host/VM for the netboot Server (the services are deployed with docker)
- a Testclient
- a VM or Buildpipeline to build the Thinclient Images
\
&nbsp;

## Prepare the Netboot Server

For our instance, we are using a plain ubuntu 20.04 LTS Server VM in order to host the necessary service for the netbooting infrastructure. Follow this step-by-step guide in order to set it up. We have set a static IP `192.168.1.101` for our netboot server, which we will use later as reference.

### 1. On your ubuntu server, install a few dependencies and tools

```bash
sudo apt-get update && sudo apt-get upgrade && sudo apt-get install git docker.io docker-compose jq curl
```

### 2. Either create a new user or use your existing non-root account in order to continue with the installation. We don't want to install the netboot infrastructure in the root context

```bash
sudo adduser "USERNAME"
```

### 3. Add your user to the docker system group and activate the changes instantly

```bash
sudo gpasswd -a "USERNAME" docker
newgrp docker
```

### 4. Create a self-signed certificate, which is used by the various scripts, to move the newly generated files between the correct destinations. Afterwards, add the Public key to the authorized_hosts file

```bash
openssl genrsa -out  ~/.ssh/netbootserver-priv.pem 1024
openssl rsa -in  ~/.ssh/netbootserver-priv.pem -pubout >  ~/.ssh/netbootserver-pub.pub
cat netbootserver-pub.pub >> ~/.ssh/authorized_keys
```

### 5. In order to deploy the services on our docker-host, execute the `run-compose.sh` and pass the Needed variables to it. This script needs the following Variables passed to it

- netbootServerIP
- pemFilePath
- devCachingServerIP (Optional, only if you want to deploy a cachingServer on a remote site)
- netbootServicesPullToken (Optional, only if you want to use your own registry)

***As we have decided for this guide to not use the caching-servers and the monitoring, we need to either remove or comment two services in our docker-compose.yaml first.***

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
...
```

To deploy the docker-containers, we do:

```bash
cd netboot
./run-compose.sh "192.168.1.101" "~/.ssh/netbootserver-priv.pem"
```

The netboot-installation Path will be `/home/USERNAME/netboot`

### 6. Now we will prepare the initial-bootloader for the TFTP Server

```bash
cd initial-bootloader
./dockerbuild.sh
```

### 7. Move the newly generated files into the config directory of the netboot-server

``` bash
mv ipxe32.efi /home/USERNAME/netboot/config/menus/
mv ipxe64.efi /home/USERNAME/config/menus/
mv undionly.kpxe /home/USERNAME/config/menus/
```

### 8. For the last step in order for clients to netboot, we need to add our netboot-server to our DHCP Server as "next-server". We are using an OPNSense Gateway for our environment. Lookup your documentations for your dhcp-server Device / Software. Also Make sure that the DHCP Server is running and is releasing IP Addresses to clients

On an OPNSense, go to `Services > DHCPv4 > {YOUR LAN}`

```undefined
- Enable network booting
    - set next-server: 192.168.1.101
    - set default bios filename: undionly.kpxe
    - set UEFI 32bit filename: uefi32.efi
    - set UEFI 64bit filename: uefi64.efi
```

### Now you should be able to network boot from the new netboot server. You can setup a new testclient / VM in the same network, and try to netboot. But we are not finished yet. If everything goes as expected, you will receive the following error while network booting

```cmd
tftp://192.168.1.101/menu.ipxe... no such file or directory
```

This is completely fine, as the menu.ipxe will be generated down the line of this guide.
\
&nbsp;

## Setup the Pre-requisites for Building Custom Images

In this chapter, we will prepare and get any helper-tools, to finally build our ubuntu images and deploy them to the netboot-server.

### 1. Build the docker Image for Squashfs. Go back to your git root and do the docker builds

```cmd
cd ../../
docker image build -t anymodconrst001dg.azurecr.io/planetexpress/squashfs-tools:latest ./squashfs-tools/
```

### 2. Build the docker Image for Jinja2-Templating Docker image

```cmd
docker image build -t anymodconrst001dg.azurecr.io/planetexpress/jinja2-templating:latest ./jinja2-templating/
```

### 3. Create the ubuntu-base image

```cmd
cd ubuntu-base && ./build.sh && cd ..
```

Now everything should be ready, in order to continue with the juicy part of this project - the Thinclient repo!
\
&nbsp;

## Build Ubuntu Thinclient Images and promote it to the netboot-server

In this part of the Guide, we will get the last repository and show you, how you can build a custom image, in order for thincients to boot from.

### 1. Change into the Thinclients Repository and build the Image using the build.sh Scripts. The output will be a squashFS File, which contains the image which will be delivered on network-booting

```cmd
cd thinclient/build
./build.sh
```

### 2. Go to the Kernel Folder and build the custom Kernels

This script needs the following Variables passed to it:

- netbootServerIP
- pemFilePath
- netbootUsername

```cmd
cd .. && cd kernel-updates
./get-kernel-updates.sh "~/.ssh/netbootserver-priv.pem" "192.168.1.101" "USERNAME"
```

### 3. Promote the Generated "squashfs" File to the netboot Server

This script needs the following Variables passed to it:

- pemFilePath
- squashfsFileName (this file should be available after executing the `./build.sh` from step 1.)
- netbootServerIP
- netbootUsername
- folderToPromoteTo (either prod or dev)

```cmd
cd .. && cd promote
./promote.sh "~/.ssh/netbootserver-priv.pem" "21-06-22-kde-master-55fa393.squashfs" "192.168.1.101" "USERNAME" "prod"
```

### 4. Generate the IPXE Menu based on the promoted Images

The following script will generate an IPXE Menu, which is used by the netboot server (and which we mentioned earlier as a dependency).

- pemFilePath
- netbootServerIP
- netbootUsername
- squashfsFileName (this file should be available after executing the `./build.sh` from step 1.)

```cmd
./generate-new-ipxe-menus.sh "~/.ssh/netbootserver-priv.pem" "192.168.1.101" "USERNAME" "21-06-22-kde-master-55fa393.squashfs"
```

### 5. When you have reached it this far, it looks very promising, that everything should be in place as expected! Now, you need to verify, if this is the case! To check if everything is in place, check the following directories and files

```cmd
ASSETS:
/home/USERNAME/netboot/assets/kernels/{KernelVersionName}/initrd
/home/USERNAME/netboot/assets/kernels/{KernelVersionName}/vmlinuz
/home/USERNAME/netboot/assets/prod/{ImageName}.squashfs
/home/USERNAME/netboot/assets/prod/{ImageName}-kernel.json

CONFIGS:
/home/USERNAME/netboot/config/menus/advancedmenu.ipxe
/home/USERNAME/netboot/config/menus/ipxe32.efi
/home/USERNAME/netboot/config/menus/ipxe64.efi
/home/USERNAME/netboot/config/menus/menu.ipxe
/home/USERNAME/netboot/config/menus/netinfo.ipxe
/home/USERNAME/netboot/config/menus/undionly.kpxe
```

Make sure to check , if the menu.ipxe has reasonable content and is not empty. Once those files are available and also reachable by your docker containers from the netboot repository, you should be able to spin up that machine!

Go to your testclient and boot from PXE and enjoy a fresh built, stateless ubuntu OS!

This project has been brought to you by Planet Express @ Digitec Galaxus AG!
\
&nbsp;

## Troubleshooting / Notes

- This Guide has been tested on a lab-environment with Microsoft Hyper-V
- This Guide does not contain the usage of the caching-server and monitoring features.
- It shows the manual approach to use the project. We highly recommend using a CI / CD Platform that will take care of all the steps above! You can check out the associated azure-pipelines.yml as reference.
- While there may be issues on the first attempt to build a Docker Image, make sure that you remove the temporarily generated files from your working directory when rebuilding the Docker Image. If you don't do this, the script ***can*** exit with an error.
- If you still have problems following this guide or get along with the setup, consider opening an issue. See [***Contribute***](#contribute)
\
&nbsp;

## Contribute

No matter how small, we value every contribution! If you wish to contribute,

1. Please create an issue first - this way, we can discuss the feature and flesh out the nitty-gritty details
2. Fork the repository, implement the feature and submit a pull request
3. Add yourself to the CONTRIBUTORS.txt file in that pull request
4. Your feature will be added once the pull request is merged
