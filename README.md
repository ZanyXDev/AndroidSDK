# AndroidSDK

Android SDK development environment Docker image

[![Docker Hub](https://img.shields.io/badge/Docker%20Hub-info-blue.svg)](https://hub.docker.com/r/thyrlian/android-sdk/)
[![Build Status](https://travis-ci.org/thyrlian/AndroidSDK.svg?branch=master)](https://travis-ci.org/thyrlian/AndroidSDK)
[![Android Dev Digest](https://img.shields.io/badge/AndroidDevDigest-%23127-green.svg)](https://www.androiddevdigest.com/digest-127/)
[![Android开发技术周报 ](https://img.shields.io/badge/Android%E5%BC%80%E5%8F%91%E6%8A%80%E6%9C%AF%E5%91%A8%E6%8A%A5-%23114-yellowgreen.svg)](http://www.androidweekly.cn/android-dev-weekly-issue-114/)

<img src="https://github.com/thyrlian/AndroidSDK/blob/master/logo.png?raw=true" width="200">

## Goals

* It contains the complete Android SDK enviroment, is able to perform all regular Android jobs.
* Solves the problem of "*It works on my machine, but not on XXX machine*".
* Some tool (e.g. [Infer](https://github.com/facebook/infer)), which has complex dependencies might be in conflict with your local environment.  Installing the tool within a Docker container is the easiest and perfect solution.
* Directly being used as Android CI build enviroment.

## Philosophy

Provide only the barebone SDK (the latest official minimal package) gives you the most flexibility in tailoring your own SDK tools for your project.  You can maintain an external persistent SDK directory, and mount it to any container.  In this way, you don't have to waste time on downloading over and over again, meanwhile, without having any unnecessary package.

## Caveat

Run Android SDK update directly within the **Dockerfile** or inside the **container** would fail if the storage driver is `AUFS` (by default), it is due to some file operations (during updating) are not supported by this storage driver, but changing it to `Btrfs` would work.

What happens if the update fails?
```bash
ls $ANDROID_HOME/tools/
#=> empty, nothing is there
# tools such as: android, sdkmanager, emulator, lint and etc. are gone

android
#=> bash: android: command not found

sdkmanager
#=> bash: /opt/android-sdk/tools/bin/sdkmanager: No such file or directory
```

To know more about the storage driver:

* Check Docker's current storage driver option
```console
docker info | grep 'Storage Driver'
```

* Check which filesystems are supported by the running host kernel
```console
cat /proc/filesystems
```

To prevent this problem from happening, and you don't wanna bother dealing with storage driver.  The only solution is to mount an external SDK volume from host to container.  Then you are free to try any of below approaches.

* Update SDK in the usual way but directly inside container.
* Update SDK from host directory (**Remember**: the host machine must be the same target architecture as the container - `x86_64 Linux`).

If you by accident update SDK on a host machine which has a mismatch target architecture than the container, some binaries won't be executable in container any longer.

```bash
gradle <some_task>
#=> Error: java.util.concurrent.ExecutionException: java.lang.RuntimeException: AAPT process not ready to receive commands

$ANDROID_HOME/build-tools/x.x.x/aapt
#=> aapt: cannot execute binary file: Exec format error

adb
#=> adb: cannot execute binary file: Exec format error
```

## Getting Started

```bash
# build the image
# set the working directory to the project's root directory first
docker build -t android-sdk android-sdk
# or pull the image
docker pull thyrlian/android-sdk

# below commands assume that you've pulled the image

# copy the pre-downloaded SDK to the mounted 'sdk' directory
docker stop $(docker ps -aqf "ancestor=thyrlian/android-sdk") &> /dev/null && docker rm $(docker ps -aqf "ancestor=thyrlian/android-sdk") &> /dev/null; docker run -d -v $(pwd)/sdk:/sdk thyrlian/android-sdk && docker exec -it `docker ps -aqf "ancestor=thyrlian/android-sdk"` bash -c 'cp -a $ANDROID_HOME/. /sdk' && docker stop $(docker ps -aqf "ancestor=thyrlian/android-sdk") > /dev/null && docker rm $(docker ps -aqf "ancestor=thyrlian/android-sdk") > /dev/null

# go to the 'sdk' directory on the host, update the SDK
# ONLY IF the host machine is the same target architecture as the container
# JDK required on the host
echo "y" | sdk/tools/android update sdk ...

# mount the updated SDK to container again
# if the host SDK directory is mounted to more than one container
# to avoid multiple containers writing to the SDK directory at the same time
# you should mount the SDK volume in read-only mode
docker run -it -v $(pwd)/sdk:/opt/android-sdk:ro thyrlian/android-sdk /bin/bash

# you can mount without read-only option, only if you need to update SDK inside container
docker run -it -v $(pwd)/sdk:/opt/android-sdk thyrlian/android-sdk /bin/bash
```

## SSH

It is also possible if you wanna connect to container via SSH.  There are two different approaches.

* Build an image on your own, with a built-in `authorized_keys`

```bash
# Put your `id_rsa.pub` under `android-sdk/authorized_keys` (as many as you want)

# Build an image
docker build -t android-sdk android-sdk

# Run a container
docker run -d -p 2222:22 -v $(pwd)/sdk:/opt/android-sdk:ro android-sdk
```

* Copy a local `authorized_keys` file to a container

```bash
# Create a local authorized_keys file

# Run a container
docker run -d -p 2222:22 -v $(pwd)/sdk:/opt/android-sdk:ro thyrlian/android-sdk

# Copy the local authorized_keys file to the running container
docker cp $(pwd)/authorized_keys `docker ps -aqf "ancestor=thyrlian/android-sdk"`:/root/.ssh
```

That's it!  Now it's up and running, you can ssh to it

```console
ssh root@<your_ip_address> -p 2222
```

And, in case you need, you can still attach to the running container (not via ssh) by

```console
docker exec -it <container_id> /bin/bash
```

<img src="https://github.com/thyrlian/AndroidSDK/blob/master/SSH.png?raw=true">

## NFS

You can host the Android SDK in one place, and share it with different containers.  One solution is using NFS (Network File System).

Here are instructions for configuring a NFS server (on Ubuntu):

```bash
sudo apt-get update
sudo apt-get install -y nfs-kernel-server
sudo mkdir -p /var/nfs/android-sdk

# put the Android SDK under /var/nfs/android-sdk
# if you haven't got any, run below commands
sudo apt-get install -y wget zip
cd /var/nfs/android-sdk
sudo wget -q $(wget -q -O- 'https://developer.android.com/sdk' | grep -o "\"https://.*android.*tools.*linux.*\"" | sed "s/\"//g")
sudo unzip tools_*-linux*.zip
sudo rm tools_*-linux*.zip
sudo mkdir licenses
echo 8933bad161af4178b1185d1a37fbf41ea5269c55 | sudo tee licenses/android-sdk-license > /dev/null
echo 84831b9409646a918e30573bab4c9c91346d8abd | sudo tee licenses/android-sdk-preview-license > /dev/null
echo d975f751698a77b662f1254ddbeed3901e976f5a | sudo tee licenses/intel-android-extra-license > /dev/null

# configure and launch NFS service
sudo chown nobody:nogroup /var/nfs
echo "/var/nfs         *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee --append /etc/exports > /dev/null
sudo exportfs -a
sudo service nfs-kernel-server start
```

## Emulator

Running emulator inside container is not a problem, but the performance is quite limited.

According to [Google's documentation](https://developer.android.com/studio/run/emulator-acceleration.html#accel-vm):

> **VM acceleration restrictions**

> Note the following restrictions of VM acceleration:

> * You can't run a VM-accelerated emulator inside another VM, such as a VM hosted by VirtualBox, VMWare, or Docker. You must run the emulator directly on your system hardware.

> * You can't run software that uses another virtualization technology at the same time that you run the accelerated emulator. For example, VirtualBox, VMWare, and Docker currently use a different virtualization technology, so you can't run them at the same time as the accelerated emulator.

### Preconditions on the host machine

Read [How to Start Intel Hardware-assisted Virtualization (hypervisor) on Linux](https://software.intel.com/en-us/blogs/2012/03/12/how-to-start-intel-hardware-assisted-virtualization-hypervisor-on-linux-to-speed-up-intel-android-x86-emulator) for more details.

Read [KVM Installation](https://help.ubuntu.com/community/KVM/Installation) if you haven't got KVM installed on the host yet.

* Check the capability of running KVM

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
# or
egrep -c ' lm ' /proc/cpuinfo
# when output > 0 it means the host CPU supports hardware virtualization.

sudo kvm-ok
# seeing below info means you can run your virtual machine faster with the KVM extensions
INFO: /dev/kvm exists
KVM acceleration can be used
```

* Load KVM module on the host
```console
modprobe kvm_intel
```

* Check if KVM module is successfully loaded
```console
lsmod | grep kvm
```

### How to run emulator

* Check available emulator system images from remote SDK repository (on the host machine)
```console
sdk/tools/android list sdk --extended --no-ui --all
```

* Download emulator system image(s) (on the host machine)
```bash
echo "y" | sdk/tools/android update sdk --no-ui --all --filter <system_image_1,system_image_2,...>
# e.g.:
# sys-img-x86_64-android-24
# sys-img-x86-android-24
# sys-img-x86_64-google_apis-24
# sys-img-x86-google_apis-24
# sys-img-arm64-v8a-android-24
# sys-img-armeabi-v7a-android-24
```

* Run Docker container in privileged mode, so that it's able to access to all devices on the host
```console
docker run -it --privileged -v $(pwd)/sdk:/opt/android-sdk:ro thyrlian/android-sdk /bin/bash
```

* Check acceleration ability
```bash
emulator -accel-check

# when succeeds
accel:
0
KVM (version 12) is installed and usable.
accel

# when fails (probably due to unprivileged mode)
accel:
8
/dev/kvm is not found: VT disabled in BIOS or KVM kernel module not loaded
accel
```

* List existing Android targets
```bash
android list targets
# ==================================================
Available Android targets:
----------
id: 1 or "android-24"
     Name: Android 7.0
     Type: Platform
     API level: 24
     Revision: 2
     Skins: HVGA, QVGA, WQVGA400, WQVGA432, WSVGA, WVGA800 (default), WVGA854, WXGA720, WXGA800, WXGA800-7in
 Tag/ABIs : default/arm64-v8a, default/armeabi-v7a, default/x86, default/x86_64, google_apis/x86, google_apis/x86_64
# ==================================================
```

* Create a new Android Virtual Device
```bash
echo "no" | android create avd -n <name> -t <target> -b <abi>
# e.g.:
echo "no" | android create avd -n avd24 -t android-24 -b default/x86_64
```

* List existing Android Virtual Devices
```bash
android list avd
# ==================================================
Available Android Virtual Devices:
    Name: avd24
    Path: /root/.android/avd/avd24.avd
  Target: Android 7.0 (API level 24)
 Tag/ABI: default/x86_64
    Skin: WVGA800
# ==================================================

# or

emulator -list-avds
# ==================================================
avd24
# ==================================================
```

* Launch emulator in background
```console
emulator64-x86 -avd <virtual_device_name> -noaudio -no-boot-anim -no-window -noskin -accel on &
```

* Check the virtual device status
```bash
adb devices
# ==================================================
List of devices attached
emulator-5554	offline
# "offline" means it's still booting up
# ==================================================

# ==================================================
List of devices attached
emulator-5554	device
# "device" means it's ready to be used
# ==================================================
```

Now you can for instance run UI tests on the emulator (just remember, the performance is POOR):
```console
<your_android_project>/gradlew connectedAndroidTest
```

## Android Commands Reference

* Check installed Android SDK tools version
```console
cat $ANDROID_HOME/tools/source.properties | grep Pkg.Revision
cat $ANDROID_HOME/platform-tools/source.properties | grep Pkg.Revision
```

* List installed and available packages
```console
sdkmanager --list
```

* List available packages from remote SDK repository
```console
android list sdk -e
```

* Update the SDK
```bash
# by name
echo "y" | android update sdk --no-ui --all --filter tools,platform-tools,extra-android-m2repository,build-tools-25.0.0,android-25

# by id
echo "y" | android update sdk -u -a -t 1,2,3,...,n
```

* Stop emulator
```console
adb -s <device_sn> emu kill
```

## Change Log

### 1.2

```console
docker pull thyrlian/android-sdk:1.2
```

**What's New**
* Upgraded Gradle from 3.2.1 to 3.3

Component | Version
--------- | -------
Ubuntu | 16.04.1 LTS (Xenial Xerus)
Java | 1.8.0_121
Gradle | 3.3
AndroidSDK | 25.2.3
OpenSSH | 7.2p2

---

### 1.1

```console
docker pull thyrlian/android-sdk:1.1
```

**What's New**
* Added SSH server and configurations

Component | Version
--------- | -------
Ubuntu | 16.04.1 LTS (Xenial Xerus)
Java | 1.8.0_121
Gradle | 3.2.1
AndroidSDK | 25.2.3
OpenSSH | 7.2p2

---

### 1.0

```console
docker pull thyrlian/android-sdk:1.0
```

Component | Version
--------- | -------
Ubuntu | 16.04.1 LTS (Xenial Xerus)
Java | 1.8.0_121
Gradle | 3.2.1
AndroidSDK | 25.2.3

## License

Copyright (c) 2016-2017 Jing Li. It is released under the [Apache License](https://www.apache.org/licenses/LICENSE-2.0). See the [LICENSE](https://raw.githubusercontent.com/thyrlian/AndroidSDK/master/LICENSE) file for details.

By continuing to use this Docker Image, you accept the terms in below license agreement.

* [Android Software Development Kit License Agreement](https://raw.githubusercontent.com/thyrlian/AndroidSDK/master/EULA/AndroidSoftwareDevelopmentKitLicenseAgreement) (or read it [here](https://developer.android.com/studio/terms.html))
* [Android SDK Preview License Agreement](https://raw.githubusercontent.com/thyrlian/AndroidSDK/master/EULA/AndroidSDKPreviewLicenseAgreement)
* [Intel Android Extra License](https://raw.githubusercontent.com/thyrlian/AndroidSDK/master/EULA/IntelAndroidExtraLicense)
