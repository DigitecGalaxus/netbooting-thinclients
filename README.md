# Introduction

This is repository ties together all repositories to make netbooting thinclients possible. In this context, a **thin client** is a lightweight operating system containing a small amount of tools only, compared to a full client such as running a Windows operating system. There are a couple of advantages to running a thin client:

- Security: less attack surface, as fewer components are running
- Security: an always up-to-date operating system including applications, as the image can be rebuilt periodically
- Maintenance: the devices do not need to be managed and patched (e.g. SCCM or Intune with Windows)
- Cost: No licenses need to be payed
- Cost: Cheaper devices can be used, no full-blown laptops or PCs are needed. The hard disk can be omitted completely.

The thin client is **booted from the network** and **stateless**, which means that any device in the network can boot the thin client operating system, even if the device does not have a disk. During startup, the thin client image is loaded into RAM and booted from there. This puts two requirements on this solution: The clients need sufficient RAM (we have observed, that it's getting tight with 8GB RAM, so we recommend 16GB) and the network needs to be fast for adequate boot times.

The usecase is mostly for browser-based tasks. Both in retail shops and in warehouses, most of the workflows are browser-only. This means, we do not require a full-blown operating system with complex programs such as development environments, Microsoft Excel or other programs that are installed directly on the client.

This is a step-by-step guide to spin up the netboot infrastructure as well as creating new images for the clients to boot from. In this scenario, you won't need to have a CI / CD Pipeline ready (which we do recommend). This will hopefully help you to get a better insight on how we have structured this project. Please note, that several commands can vary and that you might need to tweak some stuff here and there.

## Boot process

![Boot process](https://raw.githubusercontent.com/DigitecGalaxus/netbooting-thinclients/main/docs/boot_process.svg "Boot process")

1. A new client, which is configured in BIOS to boot via network is booted.
2. The client sends a DHCP broadcast to obtain it's IP and network booting information.
3. The DHCP server (in our example, an Opnsense device) responds with the IP as well as the information, which contains the network boot server's IP address and the file names of the initial bootloader, which the client should fetch from the network boot server.
4. The client fetches the initial bootloader from the network boot server
5. The client runs the initial bootloader, which contains instructions to load the menu.ipxe from the network boot server
6. The menu.ipxe contains the logic to show and boot various thin client images and allows the user to interact with the menu to select the desired image. If no selection is made, the default image is downloaded into RAM and booted. The files being downloaded are:
    - initrd (Initial Ramdisk)
    - kernel (Linux Kernel)
    - squashFS (Filesystem containing the OS)

## Repository Overview

This repository contains a few submodules. Please refer to the README in each of the repositories to understand what it does. Because submodules are used, pull the latest files and update the HEAD to be on the latest commit on the "main" branch. If you haven't yet cloned the repository, no need to hush, just follow the guide and you'll be prompted to do so.

```git
git clone https://github.com/DigitecGalaxus/netbooting-thinclients.git
cd netbooting-thinclients
git submodule update --init --recursive
git submodule foreach git pull origin main
```

## Prerequisites

For the network booting infrastructure to work, there are a few required services and servers.

- Gateway, to redirect PXE netboot requests (we are using an [opnsense gateway](https://opnsense.org/) instance). We won't go into detail how to prepare a gateway.
- A host/VM for the netboot-server (the services are deployed with docker-compose).
- A test client, which will boot the thin client via network.
- A VM or buildpipeline to build the thinclient images; this can be done on your local computer too.

## Prepare the netboot server

We suggest a well supported distribution (such as Ubuntu or Fedora).

### 1. Install Updates and necessary tools

```bash
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install -y git jq ca-certificates curl gnupg file
```

### 2. Install Docker

Either consult https://docs.docker.com/engine/install/ or if you are brave:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
```

### 3. Add user

Either create a new user or use your existing non-root account in order to continue with the installation.

```bash
username="netbootuser"
sudo adduser "$username"
```

### 4. Add user to the docker and sudoers group

```bash
sudo usermod -aG docker "$username"
sudo systemctl restart docker
```

After adding the user, switch to the newly created user:

```bash
su "$username"
```
### 5. Create a SSH key

The certficate is used by various scripts, to move the newly generated files between the correct destinations. Afterwards, add the public key to the authorized_hosts file.

```bash
mkdir -p /home/$USER/.ssh
openssl genrsa -out  /home/$USER/.ssh/netbootserver-priv.pem 4096
openssl rsa -in /home/$USER/.ssh/netbootserver-priv.pem -pubout > /home/$USER/.ssh/netbootserver-pub.pem
ssh-keygen -i -m PKCS8 -f /home/$USER/.ssh/netbootserver-pub.pem >> /home/$USER/.ssh/authorized_keys
```

### 6. Check out the netboot repository

As a preparation for the next step, check out the netboot repository on the netboot server.

```bash
cd "/home/$USER"
git clone https://github.com/DigitecGalaxus/netbooting-thinclients.git
cd netbooting-thinclients
git submodule update --init --recursive
git submodule foreach git pull origin main
```

### 7. Build and deploy the services
***Remove or comment out the services you don't want in the docker-compose.yaml first.***

*The tools to build and run the network booting services need custom environment variable files. you'll find examples with all available options in the respective subfolder. By default, the `docker-compose.yml` looks for the `.env` files in the $HOME directory. This can be adjusted to your preferences.*

```bash
cp ~/netbooting-thinclients/netboot/netboot-services/cleaner/cleaner.env $HOME/
cp ~/netbooting-thinclients/netboot/netboot-services/ipxeMenuGenerator/ipxe-menu-generator.env $HOME/
cp ~/netbooting-thinclients/netboot/netboot-services/monitoring/monitoring.env $HOME/
cp ~/netbooting-thinclients/netboot/netboot-services/sync/sync.env $HOME/
```

Hopefully every variable is kind of self-descriptive. In any case, check the corresponding README.md in the respective subfolder. Once set, we can build and deploy the services. Currently, the docker images are available in a public registry. Therefore, we don't need to build them locally. If you still wish, make sure the tags are matching

```bash
cd ~/netbooting-thinclients/netboot/netboot-services/ipxeMenuGenerator 
docker build -t dgpublicimagesprod.azurecr.io/planetexpress/netboot-ipxe-menu-generators:latest .
cd ~/netbooting-thinclients/netboot/netboot-services/cleaner
docker build -t dgpublicimagesprod.azurecr.io/planetexpress/netboot-cleaner:latest .
cd ~/netbooting-thinclients/netboot/netboot-services/sync
docker build -t dgpublicimagesprod.azurecr.io/planetexpress/netboot-sync:latest .
cd ~/netbooting-thinclients/netboot/netboot-services/monitoring
docker build -t dgpublicimagesprod.azurecr.io/planetexpress/netboot-monitoring:latest .
cd ~/netbooting-thinclients/netboot/netboot-services/tftp
docker build -t dgpublicimagesprod.azurecr.io/planetexpress/netboot-tftp:latest .
cd ~/netbooting-thinclients/netboot/netboot-services/http
docker build -t dgpublicimagesprod.azurecr.io/planetexpress/netboot-http:latest .
cd
```

### 8. Prepare the required assets folder

The stack needs a few folders to be present in order to work properly. Create the following folders:

```bash
mkdir -p $HOME/netboot/config/menus     # This is where the menu.ipxe will be generated and placed
mkdir -p $HOME/netboot/assets/dev       # This is where the dev netbooting artifacts will be stored / synced
mkdir -p $HOME/netboot/assets/prod      # This is where the prod netbooting artifacts will be stored / synced
```

### 9. Start the services

```bash
cd $HOME/netbooting-thinclients/netboot/
docker compose up -d
```

If everything went well, the services should be up and running. You can check the status of the services with `docker compose ps`. The output should look similar to this:

```bash
CONTAINER ID   IMAGE                                                                            COMMAND                  CREATED         STATUS         PORTS                               NAMES
2bb69fa3c382   dgpublicimagesprod.azurecr.io/planetexpress/netboot-ipxe-menu-generator:latest   "/bin/sh -c /usr/loc…"   3 seconds ago   Up 2 seconds                                       netboot-build-main-ipxe-menus
114053f9eefd   dgpublicimagesprod.azurecr.io/planetexpress/netboot-cleaner:latest               "/app/cleaner"           3 seconds ago   Up 2 seconds                                       netboot-cleaner
fb050987d0f1   dgpublicimagesprod.azurecr.io/planetexpress/netboot-http:latest                  "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   netboot-http
b3834bc37a0e   dgpublicimagesprod.azurecr.io/planetexpress/netboot-monitoring:latest            "/entrypoint.sh /etc…"   3 seconds ago   Up 2 seconds   8092/udp, 8125/udp, 8094/tcp        netboot-monitoring
025e3cbbc0db   dgpublicimagesprod.azurecr.io/planetexpress/netboot-sync:latest                  "/scripts/syncer.sh"     3 seconds ago   Up 2 seconds                                       netboot-sync
7d208eec4c5d   dgpublicimagesprod.azurecr.io/planetexpress/netboot-tftp:latest                  "/usr/sbin/in.tftpd …"   3 seconds ago   Up 2 seconds   0.0.0.0:69->69/udp, :::69->69/udp   netboot-tftp
```

Also verify if you can access the netboot server via HTTP. Open a browser and navigate to `http://192.168.1.101/`. You should see a page similar to this:

![Netboot Server](https://raw.githubusercontent.com/DigitecGalaxus/netbooting-thinclients/main/docs/http-netboot-server.png "Netboot Server")

## Setup the pre-requisites for building custom images (on the netbootserver itself)

In this chapter, we will prepare and get any helper-tools, to finally build our ubuntu images and deploy them to the netboot-server.

### 1. Prepare squashfs-tools

Build the docker image for Squashfs. Go back to your git root and do the docker builds.

```sh
cd ~/netbooting-thinclients
netbootingThinclientsPath=$(pwd)
cd "$netbootingThinclientsPath/squashfs-tools"
docker image build -t dgpublicimagesprod.azurecr.io/planetexpress/squashfs-tools:latest .
```

### 2. Prepare Jinja2-Templating

```sh
cd "$netbootingThinclientsPath/jinja2-templating"
docker image build -t dgpublicimagesprod.azurecr.io/planetexpress/jinja2-templating:latest .
```

### 3. Get the private key to the netboot server

To proceed with the thin client image build, get the private key that you created above from `/home/$USER/.ssh/netbootserver-priv.pem`. 

For simplicity, move it to the same location but on the device, where the thin client image is built. It is also possible to build it on the netboot server, however this is not recommended. Some parts of the script assume that it is not the same location and use `scp` to transfer files to the netboot server.

Now everything should be ready, in order to continue with the juicy part of this project - the thinclient repo!

## Build custom thin client image

### 1. Build the thin client artifacts and promote them to the netboot server

Change into the build folder use the build/build.sh script. The output will be three artifact files: initrd, kernel and squashfs. These can then be put into a folder (name it how you please) inside the prod-folder on the netboot server. The menu generator will then pick these folders and artifacts up and generate the boot menu entries.

```sh
cd "$netbootingThinclientsPath/thinclients/build"
./build.sh exportArtifacts=true
```

### 3. Generate the IPXE Menu based on the promoted Images

On the netboot server, wait for the container `netboot-build-main-ipxe-menus` to build the new menu.ipxe. It will do this periodically every minute. Verify that the squashfs files are in the correct folder on the netboot server and if the menu.ipxe has been generated. Ensure that you will have at least one `prod` folder with images in the `assets/prod` folder, otherwise the menu will not be generated.

```bash
ls ~/netboot/assets/dev
ls ~/netboot/assets/prod
ls ~/netboot/config/menus/
cat ~/netboot/config/menus/menu.ipxe | grep "set squash_url"
```

### 5. Verify folder structure and generated files

When you have reached it this far, it looks very promising, that everything should be in place as expected! Now, you need to verify, if this is the case! To check if everything is in place, check the following structure on the netboot server at `/home/$USER/netboot`.

```undefined
├── assets
│   ├── dev
│   │   ├── version1
│   │   │   ├── initrd
│   │   │   ├── vmlinuz
│   │   │   └── thinclient-base.squashfs
│   │   ├── version2
│   │   │   ├── initrd
│   │   │   ├── vmlinuz
│   │   │   └── thinclient-base.squashfs
│   └── prod
│       ├── version1
│       │   ├── initrd
│       │   ├── vmlinuz
│       │   └── thinclient-base.squashfs
│       ├── version2
│       │   ├── initrd
│       │   ├── vmlinuz
│       │   └── thinclient-base.squashfs
├── config
│   └── menus
│       ├── advancedmenu.ipxe
│       ├── menu.ipxe
│       └── netinfo.ipxe
├── docker-compose.yaml
└── netboot-services
    ├── cleaner
    ...

```

### 6. Configure Gateway configuration

The next-server value can be configured for DHCP leases and tells the clients where to attempt a netboot. The configuration is depending on the DHCP server you're using. The `dgpublicimagesprod.azurecr.io/planetexpress/netboot-tftp` image already has the bootloaders in place. 

```
default bios filename:  undionly.kpxe
UEFI 32bit filename:    uefi32.efi
UEFI 64bit filename:    uefi64.efi
```

### 2.  Boot the test client

Now the test client should be able to boot from the network. You can setup a new test client / VM in the same network as the netboot server, and boot the test client. Once those files are available on the netboot server, the containers will serve them via TFTP (Port 69) and HTTP (Port 80). The test client should now be able to boot the test client via network!

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
