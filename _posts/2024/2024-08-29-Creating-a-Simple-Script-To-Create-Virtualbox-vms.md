---
title: "Scripting VirtualBox VM Creation: Using VBoxManage"
date: 2024-08-29 17:25:26 +0300
categories: [Scripting, bash]
tags: [virtualization, scripting, bash]
image: /assets/img/vm/vbox-logo.png
alt: "Scipting VirtualBox VM Creation: Using VBoxManage"
---

## What i was doing
In the quest for upskilling my devops skills, i found myself in need for several vms each running a different service: jenkins, maven, sonaqube, argocd, kubernetes and defectdojo.

The ubuntu server (no GUI) is necessary in this case since we have a server rack and lots of space and RAM. The lab setup is next.

The steps are clear:
- Download an ubuntu server iso.
- Download virtualbox.
- Create VMs.
- Provision for storage and RAM for performance capacities.
- Install the devops tools

The above can be setup for public access through proxing, but that is a story for another day. 

## The Challenge
The challenge was creating a script to auto create the VMs using `VBoxManage`.

The settings will be as follows:
- Network adaptor : `bridged`
- RAM : `2GB`
- Storage : `8GB`
- OS type : `Ubuntu_64`

Below is the full script with steps as commented. 

## The Script

```bash
#!/bin/bash

# ubuntu iso file
ISO_PATH="ubuntu-20.04.6-live-server-amd64.iso"

# Provisioning the storage and RAM
RAM_SIZE=2048
DISK_SIZE=8000


# network interface.
NETWORK_ADAPTER_NAME="wlp4s0"

# Check if vboxmanage is installed
if ! command -v VBoxManage &> /dev/null; then
    echo "VBoxManage not installed! Please install VirtualBox"
    exit 1
fi

# VM names to create
VM_TITLES=("devops-jenkins" "devops-maven" "devops-sonarqube" "devops-argocd" "devops-kubernetes" "devops-defectdojo")

# Loop through above names to create vms
for VM_TITLE in "${VM_TITLES[@]}"; do
    echo "Creating VM: $VM_TITLE"
    
    # Create the vms
    VBoxManage createvm --name "$VM_TITLE" --ostype Ubuntu_64 --register
    
    # Set memory, boot order, and network settings
    VBoxManage modifyvm "$VM_TITLE" --memory "$RAM_SIZE" --boot1 dvd --nic1 bridged --bridgeadapter1 "$NETWORK_ADAPTER_NAME"
    
    # Create a virtual hard disk in VMDK format
    VBoxManage createmedium disk --filename "$HOME/VirtualBox VMs/$VM_TITLE/$VM_TITLE.vmdk" --size "$DISK_SIZE" --format VMDK
    
    # Attach the disk to the VM
    VBoxManage storagectl "$VM_TITLE" --name "SATA Controller" --add sata --controller IntelAhci
    VBoxManage storageattach "$VM_TITLE" --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium "$HOME/VirtualBox VMs/$VM_TITLE/$VM_TITLE.vmdk"
    
    # Attach the ISO file to the VM
    VBoxManage storagectl "$VM_TITLE" --name "IDE Controller" --add ide
    VBoxManage storageattach "$VM_TITLE" --storagectl "IDE Controller" --port 0 --device 0 --type dvddrive --medium "$ISO_PATH"
    
    echo "VM $VM_TITLE created, ISO attached, and networking set to bridged mode."
done

echo "All VMs have been created with bridged networking and VMDK disks. You can now start them to complete the Ubuntu installation."
```

The machines are created and provisioned with allocated resources.
![error](/assets/img/vm/output.png)

This can be confirmed from the VirtualBox manager interface.
![error](/assets/img/vm/vbox.png)

First step in my devops learning path :-)