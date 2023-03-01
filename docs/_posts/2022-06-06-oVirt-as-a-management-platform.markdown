---
layout: posts
title:  "Deploy oVirt as a VM Image Management Platform"
author: Minjong Ha
published: true
date:   2022-06-16 10:48:00 +0900
---

## Abstract
oVirt is an open-source distributed virtualization solution, designed to manage your entire enterprise infrastructure. oVirt uses the trusted KVM hypervisor and is built upon several other community projects, including libvirt, Gluster, PatternFly, and Ansible. [oVirt Official Site](https://www.ovirt.org/)

Usually, oVirt presents a distributed system for the VMs and helps to deploy the cloud infrastructure.
In this post, I will explain how to deploy the oVirt and describe the architecture.
Then I will talk about why I tried to deploy oVirt as an effective VM image footpring management.


## Introduction

Managing VM images is important since the users requires various image shape.
Differences in images may appear in the OS, in versions within the same OS, or in applications depending on usage within the same version.
VM service manager performs numerous major and minor updates and it causes fragmentation of images.
For this reason, We need to use an infrastructure to manage and update the images effectively.

I choosed oVirt as an infrastructure for image management for some reasons.
First, it is open-source software.
There is no additional cost for running after I prepare the machines to deploy.
Second, it presents web-console GUI.
Through the web-console, I can easily create VMs and templates and run the VM itself using web-presented vnc.
It makes VM image update easy and helps the manager in convinient.
Third, it is easy to add hosts and storage.
It is also possible via web-console to add hosts and storage domains.
I can add NFS or glusterFS based storage domain when I need more storage.

There are many VM supports in oVirt as an DaaS, however, I focused on the features for VM image management and describe the reason in the above paragraph.
Especially, what caught my eye in particular is the templated image.
It looks attractive to managing the image versions as a template.


## oVirt: Architecture
I deployed a CentOS 7 as a hosted engine, and three oVirt-nodes (4.3) as host and storage domains.
Also, to manage FQDNs for each nodes, I implemented docker-dnsmasq on the another machine as an internal DNS server.
It could be replaced manual /etc/hosts editting, which is more annoying to manage.

Followings represent the overall architecture of ovirt-engine I deployed.

* 5 nodes
> * ovirt-engine: CentOS 7 + oVirt 4.3 installed
> * ovirt-node 0 - 3: oVirt-node 4.3 installed
>> * ovirt-node 0 - 2: host and glusterFS storage
>> * ovirt-node 3: host and NFS storage

Each engine and nodes has same hardware specification: 128 GB SSD + 1 TB HDD with Intel CPU.
They are simple and domestic PC, not the servers.

## Deployment
<!--- need image to describe the architecture --->
<!--- describe install sequences --->

### oVirt-Engine
First, I deployed the oVirt-engine on CentOS 7.
Following codes represent the commands that I use to install the engine.

```bash
yum update

reboot

yum install http://resoureces.ovirt.org/pub/yum-repo/ovirt-release43.rpm

yum install ovirt-engine

engine-setup
# install ovirt-engine based on default configuration
# FQDN: ovirt-node-*.ovirt.net (same as hostname, FQDN in DNS)
# Organazation name for certificate: ovirt.net
# (FQDN and Organazation name coule be changed)
```

After the installation, You can access the oVirt-engine web console through the "https://engine.ovirt.net".
ID and PW is defined during the engine-setup. 
Usually, ID is "admin".


### oVirt-Nodes
I installed oVirt-node 4.3 on the computers using ISO installed USB.
After I installed the oVirt-node OS, I set the ssh authorization for each nodes.

1. open oVirt-engine web console
2. select "add host" in Computing section.
3. copy the public ssh key from web console.
4. Inside the each nodes...(root)

```bash
mkdir .ssh
chmod 700 .ssh
vi .ssh/authorized_key
${paste_public_ssh_key_from_hosted_engine}
```

After I finished above sequences, I add the oVirt-nodes as hosts for oVirt-engine.


### Storage: glusterFS
(192.168.103.82: oVirt-engine, 192.168.103.83-85: oVirt-node-0~2)

1. Configure Firewall

Run following commands in node 83

```bash
iptables -I INPUT -p all -s 192.168.103.82 -j ACCEPT
iptables -I INPUT -p all -s 192.168.103.84 -j ACCEPT
iptables -I INPUT -p all -s 192.168.103.85 -j ACCEPT
```

Run following commands in node 84

```bash
iptables -I INPUT -p all -s 192.168.103.82 -j ACCEPT
iptables -I INPUT -p all -s 192.168.103.83 -j ACCEPT
iptables -I INPUT -p all -s 192.168.103.85 -j ACCEPT
```

Run following commands in node 85

```bash
iptables -I INPUT -p all -s 192.168.103.82 -j ACCEPT
iptables -I INPUT -p all -s 192.168.103.83 -j ACCEPT
iptables -I INPUT -p all -s 192.168.103.84 -j ACCEPT
```

Configurate in 83, which is the main glusterfs server

```bash
ssh-keygen
ssh-copy-id root@ovirt-node-*.ovirt.net
ssh-keyscan -H ovirt-node*.ovirt.net
```

Cofigurate in 83, 84, 85

```bash
service glusterd start
gluster peer probe ovirt-node*.ovirt.net
```

gluster requires dedicated storage to deploy.
I formatted the 1TB HDD (/dev/sdb)

```bash
fdisk /dev/sdb
n
w
mkfs.xfs -f -i size=512 /dev/sd
```

Setup the gluster volume on the https://ovirt-node-0.ovirt.net
> Virtualization - Hosted Engine - Hyperconverged
>/dev/sdb1


### Storage: NFS

Create NFS directory and configuration

```bash
mkdir /exports/data
chown 36:36 /exports/data
chmod g+s /exports/data
```

Automount NFS storage device

```bash
mkfs.xfs -f -i size=512 /dev/sdb1 /exports

vi /etc/fstab
#/dev/sdb1 /exports/data xfs defaults 0 0 

mount -a
```

Configurate NFS accessible

```bash
vi /etc/exports
exports/data *(rw)

exportfs -r
```

Configurate NFS service accessible from external network
```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
```

Add storage domain (ovirt-node-3.ovirt.net:/exports/data) in ovirt-engine web console


# Result
If you followed above guides and deploy the oVirt-engine success, it presents 4 hosts (ovirt-node-0 ~ 3) and 2 storage (glusterFS, NFS).
You can upload the image and make that image a VM.
Then, you can template the VM.

Each templates has their own name, description, comments.
It is also possible to manage the templates with versions using sub-template feature.

# Appendix
## oVirt template image download in the CLI with REST API

I also implement a CLI template downloader using Python 3. 
I used oVirt-engine REST API.
> [ovirt-template-manager](https://github.com/minjong-ha/ovirt-template-manager)

"oVirt-engine" issues certification to the client, and client requests multiple features including download template, create VM and etc.

## oVirt-engine REST API
oVirt-engine presents REST API.
Above ovirt template downloader using Python uses the REST API.
> [ovirt-engine REST API](http://ovirt.github.io/ovirt-engine-api-model/)


[oVirt-engine Deployment Notion Document (korean)](https://seen-fact-e72.notion.site/VM-oVirt-92ca20a41c1741f4ac4b39f0c97f56a2)
