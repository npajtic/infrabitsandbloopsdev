---
layout: single
title: "Generate QCOW2 images with Packer"
date: 2026-04-05
tags:
  - QCOW2
  - Packer
author: Nikola Pajtic
category: Cloud
toc: true
toc_label: "My Table of Contents"
toc_icon: "cog"
---

# Introduction
This post explores how to build QCOW2 images with Packer. Once built, they can be uploaded to different cloud providers and provision new instances. 

This post is the first post in a series of Kubernetes The Hard Way. I built my own images to simplify the process of provisioning new Kubernetes clusters. I really did not want to repeat all pre-requisit steps every time a new host comes up (or I mess up some configs somewhere and therefore need to re-bootstrap a host).

All the steps bellow assume the Ubuntu 22.04.05 LTS release.

# Qcow2 Packer all the way
## Step 1: Install Packer
On a local machine:
```shell
sudo apt update
sudo apt install -y packer
```
## Step 2: Set up the project and its structure 
The project directory looks like this:
```shell
root_dir
|
|-- builds
   |
   |-- dev.pkrvars.hcl   
|
|-- files
   |
   |-- 99-custom.conf   
|
|-- http
   |
   |-- meta-data   
|
|-- packer.pkr.hcl
|-- variables.pkr.hcl
|-- user-data.tpl
|-- build_packer_image.sh
```

## Step 3: the packer file
The main packer file is an HCL file, which in this case was ubuntu.pkr.hcl and it should be created with:
```hcl
packer {
  required_plugins {
    qemu = {
      source  = "github.com/hashicorp/qemu"
      version = ">= 1.0.0"
    }
  }
}

# ---- QEMU SOURCE ----
source "qemu" "ubuntu" {
  iso_url      = "https://releases.ubuntu.com/22.04.5/ubuntu-22.04.5-live-server-amd64.iso"
  iso_checksum = "sha256:9bc6028870aef3f74f4e16b900008179e78b130e6b0b9a140635434a46aa98b0"

  output_directory = "output-${var.image_name}"
  vm_name          = "${var.image_name}.qcow2"

  disk_size   = "10G"
  format      = "qcow2"
  accelerator = "kvm"
  disk_interface = "virtio"
  net_device = "virtio-net"

  memory = 2048
  cpus   = 2

  ssh_username         = var.username
  ssh_password         = var.password
  ssh_timeout          = "20m"

  http_directory = "http"

  boot_wait = "20s"

  boot_command = [
    "<wait>",
    "c",
    "<wait>",
    "linux /casper/vmlinuz autoinstall ip=dhcp ds=nocloud-net\\;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ ---",
    "<enter>",
    "initrd /casper/initrd",
    "<enter>",
    "boot",
    "<enter>"
  ]

  shutdown_command = "sudo shutdown -P now"
}

# ---- BUILD ----
build {
  sources = ["source.qemu.ubuntu"]

  provisioner "file" {
    source      = "files/99-custom.conf"
    destination = "/tmp/99-custom.conf"
  }

  # k8s setup
  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y apt-transport-https ca-certificates curl wget curl vim openssl git net-tools",

      "echo 'datasource_list: [ OpenStack, ConfigDrive, NoCloud ]' | sudo tee /etc/cloud/cloud.cfg.d/99-datasource.cfg",

      # Remove installer config
      "sudo rm -f /etc/netplan/00-installer-config.yaml",

      # Disable swap
      "sudo swapoff -a",
      "sudo sed -i '/ swap / s/^/#/' /etc/fstab",

      # Load kernel modules
      "echo 'overlay' | sudo tee /etc/modules-load.d/containerd.conf",
      "echo 'br_netfilter' | sudo tee -a /etc/modules-load.d/containerd.conf",
      "sudo modprobe overlay",
      "sudo modprobe br_netfilter",

      # Sysctl settings
      "echo 'net.bridge.bridge-nf-call-iptables = 1' | sudo tee /etc/sysctl.d/k8s.conf",
      "echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/k8s.conf",
      "echo 'net.bridge.bridge-nf-call-ip6tables = 1' | sudo tee -a /etc/sysctl.d/k8s.conf",
      "sudo sysctl --system" 
    ]
  }

  # SSH hardening
  provisioner "shell" {
    inline = [
      "sudo mv /tmp/99-custom.conf /etc/ssh/sshd_config.d/99-custom.conf",
      "sudo chown root:root /etc/ssh/sshd_config.d/99-custom.conf",
      "sudo chmod 644 /etc/ssh/sshd_config.d/99-custom.conf",

      # validate config
      "sudo sshd -t"
    ]
  }

  # cleanup
  provisioner "shell" {
   inline = [
    # guest agent
    "sudo systemctl enable qemu-guest-agent",

    # cleanup
    "sudo apt-get clean",
    "sudo rm -rf /var/lib/apt/lists/*",

    # reset identity
    "sudo truncate -s 0 /etc/machine-id",
    "sudo rm -f /var/lib/dbus/machine-id",
    "sudo rm -f /etc/ssh/ssh_host_*",

    "sudo cloud-init clean --logs",
    "sudo rm -rf /var/lib/cloud/*",

    # UFW
    "sudo ufw default deny incoming",
    "sudo ufw default allow outgoing",
    "sudo ufw allow OpenSSH",
    "sudo ufw disable",

    "sudo rm -f /home/admin/.ssh/authorized_keys",

    "sync"
  ]
 }
}
```
Next, let us have some variables in variables.pkr.hcl file:
```hcl
variable "ssh_ip" {
  type        = string
  description = "Allowed SSH source IP"
  default     = "1.1.1.1"
}

variable "hostname" {
  type    = string
  default = "ubuntu"
}

variable "username" {
  type    = string
  default = "username"
}

variable "password" {
  type    = string
  default = "topsecret"
}

variable "region" {
  type    = string
  default = "contabo"
}

variable "image_name" {
  type    = string
  default = "ubuntu-base"
}

variable "ssh_public_key_path" {
  type        = string
  description = "Path to SSH public key"
}

variable "shapassword" {
  type        = string
  default = ""
}

```
Variables can be overridden through builds/ files. In this example, let's set up an override with:
```hcl
hostname       = "ubuntu"
username       = "username"
password       = "topsecret"
image_name     = "dev-ubuntu-22.04-base"
ssh_public_key_path  = "~/.ssh/dev_id_rsa.pub"
```
Next, we need to create user-data and meta-data files in http/ directory, which will be served to cloud-init during the build phase. I decided to template a user-data file via an external script and a tpl file. The user-data.tpl looks like this:
```hcl
#cloud-config
autoinstall:
  version: 1

  identity:
    hostname: $hostname
    username: $username
    password: "$shapassword"

  ssh:
    install-server: true
    allow-pw: true
    authorized_keys:
     - $ssh_public_key


  storage:
    layout:
      name: direct

  packages:
    - qemu-guest-agent
    - ufw

  late-commands:
    - curtin in-target -- systemctl enable ssh
    - curtin in-target -- sh -c "echo '$username ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/${username}"

# ---- runtime cloud-init (Contabo-compatible) ----

users:
  - default

disable_root: true
preserve_hostname: false

system_info:
  default_user:
    name: $username
    lock_passwd: false
    groups: [sudo]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash

ssh_pwauth: true

# -- network
network:
  version: 2
  ethernets:
    all:
      match:
        name: "*"
      dhcp4: true

# ---- firewall (SAFE stage) ----

runcmd:
  # reset UFW (clean state)
  - ufw --force reset

  # default policies
  - ufw default deny incoming
  - ufw default allow outgoing

  # TEMP safety rule (avoid lockout during setup)
  - ufw allow 22/tcp

  # allow SSH only from your IP(s)
  - ufw allow from 1.1.1.1/20 to any port 22 proto tcp
  - ufw allow from 10.0.0.0/8 to any port 22 proto tcp
  - ufw allow from 172.16.0.0/12 to any port 22 proto tcp
  - ufw allow from 192.168.0.0/16 to any port 22 proto tcp

  # enable firewall
  - ufw --force enable

  # remove temporary open SSH rule
  - ufw delete allow 22/tcp
```
And the shell script that would generate http/user-data file:
```bash
#!/usr/bin/env bash
set -euo pipefail

VARS_FILE="${1:-pkvars.hcl}"

# ========== RENDER USER-DATA ====================
echo "Rendering user-data from ${VARS_FILE}..."

VARS_JSON=$(hcl2json < "$VARS_FILE")

export hostname=$(echo "$VARS_JSON" | jq -r '.hostname')
export username=$(echo "$VARS_JSON" | jq -r '.username')

password=$(echo "$VARS_JSON" | jq -r '.password')
export shapassword=$(mkpasswd -m sha-512 $password)

ssh_key_path=$(echo "$VARS_JSON" | jq -r '.ssh_public_key_path')

ssh_key_path="${ssh_key_path/#\~/$HOME}"
export ssh_public_key=$(cat "$ssh_key_path")

envsubst < user-data.tpl > http/user-data

echo "http/user-data generated"

# ========== PACKER BUILD =======================
echo "Starting packer build from ${VARS_FILE}..."
packer build -var-file=${VARS_FILE} .
```
Lastly, the http/meta-data file looks fairly generic:
```hcl
instance-id: iid-local01
local-hostname: ubuntu
```
Finally, to build an image, run the script as:
```bash
./build_packer_image.sh builds/dev.pkrvars.hcl
```
Once the build is complete, there will be a separate output/ directory created with a new qcow2 image.

## Step 4: Compress & Optimize QCOW2
The final step is to compress and optimize the image and get it ready for uploading into a cloud provider. This process will reduce the size significantly.
```shell
qemu-img convert -O qcow2 -c output/ubuntu-custom.qcow2 output/ubuntu-final.qcow2
```
## Step 5: Upload the image 
If you are running your infrastructure in AWS or GCP, you can store your images within each given platform. 

However, in my case, I was using Contabo to run my K8s learning clusters on a few VPS instances. In order to use my newly built images, I had to somehow upload them into Contabo. The problem was - to upload an image, you had to provide a public URL to it, which Contabo will then use to pull the image into their image store. 

Since I had no other VPS laying around that would serve as network storage, I used an S3 bucket with additional features:

### Step 5.1: Create a bucket and upload an image
First, log into AWS, create a public bucket and then upload an image there
```shell
aws s3 cp ubuntu-final.qcow2 s3://your-bucket/
```
### Step 5.2: Generate a signed URL with a TTL on it
```shell
aws s3 presign s3://your-bucket/ubuntu-final.qcow2 --expires-in 3600
```
The result would look like:
```shell
https://your-bucket.s3.amazonaws.com/ubuntu-final.qcow2?X-Amz-Signature=...
```
Which I then gave to Contabo to pull my image into their local store. 

It goes without saying that this approach should not be used for production-grade systems. I used Contabo VPS for my own learning and K8s test environments, so there was no actual production data nor senstive information.

### 5.3: A local http server
If you can serve an HTTP traffic from your local machine (port forwarding configured, etc etc), a small local http server could do the trick:
```shell
python3 -m http.server 8000
```

# Summary
The steps above described a simple build that generated a single base image. This is fine for small setups. However, for any larger project, I strongly recommend using Jsonnet instead of HCL2. Jsonnet gives a lot more flexibility and templating options, while avoiding code repetitions that come with HCL. 