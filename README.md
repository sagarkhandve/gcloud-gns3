[![License](https://img.shields.io/badge/License-MIT-blue)](#license "Go to license section")

### Deploy gns3 Server with Web-UI in Google Cloud [KVM Enabled VM].
![145810451-cbc8619c-a548-49be-9ec3-513e54d2a159](https://user-images.githubusercontent.com/90393971/170816257-31925591-97db-4417-81d3-a37025d82acb.png)

---
## 1. Prepare your Google Cloud environment
Before you begin Set you project default region and zone.<br/>
First, click the Activate Cloud Shell button at the top right of the Google Cloud Console.<br/>
<br/>
Find you Goolge Cloud Projects by typing below command.
```bash
gcloud projects list
```
To set the default project for all gcloud commands, run the command with your **`project_id`**
```bash
gcloud config set project `your project_id`
```
initialize the Google Cloud
```bash
gcloud init
```
---
## 2. Enabling nested virtualization on an instance
**Restrictions**
- Nested virtualization can only be enabled for L1 VMs running on **Haswell processors** or later.
- E2 machine types do not support nested virtualization.
- Nested virtualization is supported only for KVM-based hypervisors running on Linux instances.
  - Hyper-V, ESX, and Xen hypervisors are not supported.
- Windows VMs do not support nested virtualization; that is, host VMs must run a Linux OS.
---
### 2.1 Create a boot disk from a public image
There are two steps required to used nested virtualization:<br/>
- The VM instances for which you want to use nested virtualization must use a custom image with a special license key.<br/>
- To enable nested virtualization on a VM instance, create a custom image with a special license key that enables VMX in VM.<br/>
- Create a boot disk from a public image or from a custom image with an operating system.<br/>

```bash
gcloud compute images list

NAME                            PROJECT              FAMILY                  DEPRECATED  STATUS
debian-10-buster-v20200910      debian-cloud         debian-10                           READY
ubuntu-2004-focal-v20200917     ubuntu-os-cloud      ubuntu-2004-lts                     READY
```
Debian Disk
```bash
gcloud compute disks create disk1 --image-project debian-cloud --image-family debian-10 --zone asia-southeast1-b
```
### 2.2 Create a custom image with a special license key for virtualization
Using the boot disk that you created, create a custom image with the special license key required for virtualization.<br/>
change zone ~**`asia-southeast1-b`**~ to your zone _example:_ **`us-east1-b`**
```bash
gcloud compute images create kvm-image \
  --source-disk disk1 --source-disk-zone asia-southeast1-b \
  --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"
```
Note: After you create the image with the necessary license, you can delete the source disk if you no longer need it.<br/>
### 2.3 Create a VM instance using the new custom image with the license
You must create the instance in a zone that supports the **Haswell** CPU Platform or newer.<br/>
Check your zone for Haswell CPU support
```bash
gcloud compute zones describe asia-southeast1-b
```
Create a VM instance using the new custom image with the license<br/>
Minimum Alternatives:
- ~**`Intel Skylake`**~ to **`Intel Haswell`**<br/>
- ~**`n1-standard-8`**~ to **`n1-standard-1`**<br/>
- ~**`pd-ssd`**~ to **`pd-standard`**<br/>

```bash
gcloud compute instances create gns3server --zone asia-southeast1-b \
              --min-cpu-platform "Intel Skylake" --machine-type=n1-standard-8 \
              --boot-disk-size=30GB --boot-disk-type=pd-ssd \
              --tags http-server,https-server \
              --image kvm-image
```
### 2.4 Create a Firewall rule to allow GNS3 on port **`3080`**
You must have to allow TCP port `3080` in the GCP firewall to access GNS3 Server.
```bash
gcloud compute firewall-rules create gns3 --action=ALLOW --rules=tcp:3080
```
### 2.5 Confirm that nested virtualization is enabled in the VM
Connect to your newly created instance with SSH.
```bash
gcloud compute ssh gns3server
```
Find Out CPU Supports for **Intel VT** Virtualization For KVM
```bash
grep -cw vmx /proc/cpuinfo
```
If the output of the above command is Greater-than zero then we have Virtualization technology enabled on our system.<br/>

---
## 3. Prepare your instance for GNS3-server Installation

- uBridge is required, it interconnects the nodes.<br/>
- Dynamips is required for running IOS routers (using real IOS images) as well as the internal switches and hubs.<br/>
- VPCS is recommended, it is a builtin node simulating a very simple computer to perform connectivity tests using ping, traceroute.<br/>
- Qemu is strongly recommended on Linux, as most node types are based on Qemu, for example, Cisco IOSv and Arista vEOS.<br/>
- libvirt is recommended (Linux only), as it's needed for the NAT cloud<br/>
- Docker is optional (Linux only), some nodes are based on Docker.<br/>
---
### 3.0 Update, Upgrade and Reboot your instance
```bash
sudo apt-get update && sudo apt-get upgrade -y && sudo reboot
```
### 3.1 uBridge - Download, Compile and Install
**Dependencies**
For Ubuntu or other Debian based Linux you need to install this package:
- libpcap-dev
```bash
sudo apt-get install make gcc libpcap-dev git wget -y
```
uBridge - Download, Compile and Install
```bash
cd ~
git clone https://github.com/GNS3/ubridge.git
cd ubridge/
make
sudo make install
cd ~

```
### 3.2 Dynamips - Download, Compile and Install
Dynamips now uses the CMake build system. To compile Dynamips you will need CMake and a working GCC or Clang compiler, as well as the build dependencies.

**Build Dependencies**<br/>
On Debian based systems the following build dependencies are required and can be installed using apt:

- libelf-dev
- libpcap0.8-dev
```bash
sudo apt-get install cmake libelf-dev libpcap0.8-dev -y
```
Dynamips - Download, Compile and Install
```bash
cd ~
git clone https://github.com/GNS3/dynamips.git
cd dynamips/
mkdir build
cd build/
cmake ..
cmake .. -DDYNAMIPS_CODE=stable  -DCMAKE_C_COMPILER=/usr/bin/gcc
make
sudo make install
cd ~

```

### 3.3 Install VPCS

```bash
wget -c http://ppa.launchpad.net/gns3/ppa/ubuntu/pool/main/v/vpcs/vpcs_0.8.2-1~jammy1_amd64.deb
chmod +x vpcs_0.8.2-1~jammy1_amd64.deb
sudo dpkg -i vpcs_0.8.2-1~jammy1_amd64.deb
```

### 3.4 QEMU and NAT (libvirt) - Install

```bash
sudo apt-get install qemu-kvm qemu-system-x86 cpulimit ovmf uml-utilities bridge-utils virtinst libvirt-daemon-system libvirt-clients -y
```

### 3.5 Docker - Install
```bash
sudo apt-get install docker.io -y
```

### 3.6 Installing i386-libraries for IOU -(IOS on UNIX)
**Dependencies:**
- libc 
- libcrypto

First add i386 architecture support then update your system and install requirements.

```bash
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install libssl-dev -y
cd ~
```
### 3.7 Installing Wireshark
**"Should non-superusers be able to capture packets?"** select `Yes`.
```bash
sudo apt-get install wireshark -y
```
### 3.8 Create a user for GNS3Server - User: **`gns3`** Pass: **`gns3`**
```bash
sudo adduser gns3

sudo adduser gns3 sudo
sudo adduser gns3 kvm
sudo adduser gns3 docker
sudo adduser gns3 wireshark
groups gns3
```
---
##  4. Install GNS3 Server
**Dependencies:**
```bash
sudo apt-get install python3-setuptools python3-pip python3-aiohttp python3-psutil python3-jsonschema -y
```
Finally Download and Install GNS3 Server.
```bash
cd ~
git clone https://github.com/GNS3/gns3-server.git
cd gns3-server
sudo python3 setup.py install
cd init
sudo cp gns3.service.systemd /lib/systemd/system/gns3.service
sudo chown root /lib/systemd/system/gns3.service
cd ~
```
### 4.1 Set GNS3 server as a daemon (auto start at boot time)
```bash
sudo systemctl enable gns3
sudo systemctl enable docker
sudo virsh net-autostart default
```
It is better to reboot your instance at this time.
```bash
sudo reboot
```
---
## 5. Testing GNS3Server
login with gns3 user:
```bash
sudo su gns3
cd ~
sudo systemctl status gns3

```
- Make sure you are login with gns3 user
- Add IOS, Qemu, Cisco Virl, IOU/IOL images and use license key to run Cisco IOU on system<br/>

- Find your instance Public IP and connect to it using browser<br/>
- Example: http://0.0.0.0:3080
