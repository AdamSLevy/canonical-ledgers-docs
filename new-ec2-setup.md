# New ec2 Setup
This guide describes the steps to start and set up a new ec2 instance for
Canonical Ledgers from the `base-image` AMI. Instructions on how to set up the
factom docker images will be described in another guide.

## Requirements
- An AWS account with ec2 admin permissions.
- Web browser
- SSH client
- A user account for you exists in the "base-image" AMI


# Steps

## Launch new ec2 instance
In these steps we will launch a new AWS ec2 instance.

### Select AMI
In the AWS management console, navigate to the "ec2" section under "Services".

Select "Launch Instance".

Select "My AMIs".

Select "base-image" or a more appropriate image if it exists. The "base-image",
which all of our ec2 instances and AMIs are based on, uses Arch Linux and is
configured to prohibit root login and is set up with our default user accounts
with password based sudo access, authorized ssh keys, and a helpful bash
prompt. It also includes ZFS and telegraf preinstalled.

### Select Instance Size
Select an approriately sized instance type. For mainnet we typically run
Authority Nodes in a t2.xlarge and guard nodes as a t2.large. Look at the
running instances in another browser tab to help determine what size we
typically use.

Click "Next: Configure Instance Details".

### Configure Instance Details
Select the proper VPC network: factom-testnet or factom-mainnet.

Select the proper subnet for the appropriate region: a, b, or c. We try to
distribute our nodes across availability zones. We have one subnet per zone.

Make sure "Auto-assign Public IP" is enabled. This should be the default.

Click "Next: Add Storage".

### Add Storage
For the root filesystem the default 8G Magnetic volume is OK.

For factom nodes an additional volume is required. Select "Add New Volume". The
defaults are acceptable. Use a 32G Volume. There is probably an appropriate
snapshot for this factom volume so please ask.

Click "Next: Add Tags".

### Add Tags
No tags are required.

Click "Next: Configure Security Group".

### Configure Security Group
Select the option to "Select an existing security group". Select the
"restricted-ssh" security group. This will allow SSH connections only from the
SSH bastion host. Alternatively select the "admin-ssh" security group if direct
SSH login is OK for this host.

We will add additional security groups later.

Click "Review and Launch".

### Review and Launch
Double check your instance type, volumes, and security groups.

Click "Launch".

### Select keypair
On the "Select an existing key pair or create a new key pair" dialog, select
"Proceed without a key pair" from the first drop down menu. The base-image will
already have the authorized keys set up and since root login via SSH is
prohibited, adding a keypair will make no difference anyway.

Check the box for "I acknowledge that I will not be able to connect to this
instance unless I already know the password built into this AMI."

Click "Launch Instances".

### Set name and lookup IPs
Go back to the "Instances" view and find the newly created instance. Edit the
"Name" column to something appropriate. Look at other names and try to follow
any existing conventions.

With the new instance selected, look at the instance "Description" at the
bottom of the page. Take note of the public and private IP addresses.

## Configure server
In the following steps we will configure the server.
### Connect over SSH
#### Publicly accessible SSH, "admin-ssh"
If the host is publicly accessible, add the following to your `~/.ssh/config`
replacing `<convenient-name>`, `<ip-address>`, and `<your-ssh-key>` with
appropriate values.

```
Host <convenient-name>
    HostName <public-ip-address>
    User <your-login-username>          # Required if the user name differs from your system
    IdentityFile ~/.ssh/<your-ssh-key>
    IdentitiesOnly yes                  # Required if you have many SSH keys in your key agent
    ControlMaster auto                  # Optional, subsequent connections share a socket
    ControlPath ~/.ssh/%r@%h:%p         # Required with ControlMaster, sets the socket location
```

#### SSH via Bastion, "restricted-ssh"
If the "restricted-ssh" security group was used, SSH connections must go
through the SSH Bastion host. In this case you must use the private IP address
which is only visible to the bastion host. Add the following to your
`.ssh/config`.

```
Host bastion bastion.entrycredits.io
    HostName bastion.entrycredits.io
    User <your-login-username>
    IdentityFile ~/.ssh/<your-ssh-key>
    IdentitiesOnly yes
    ControlMaster auto
    ControlPath ~/.ssh/%r@%h:%p

Host <convenient-name>
    HostName <private-ip-address>
    User <your-login-username>
    IdentityFile ~/.ssh/<your-ssh-key>
    IdentitiesOnly yes
    ControlMaster auto
    ControlPath ~/.ssh/%r@%h:%p
    ProxyJump bastion                   # Connect through the bastion host
```
#### Login
Successful login looks something like the following:
```
$ ssh <convenient-name>
Last login: Tue May  8 17:30:54 2018 from 137.229.93.189
Canonical Ledgers, LLC
ip-10-10-2-26
[10:05:53] <your-username>@ip-10-10-2-26:~
$
```
If the connection goes through the SSH bastion you may be prompted for your
Google Authenticator code:
```
$ ssh <convenient-name>
Verification Code:
```
As is standard for command line password prompts on unix, no characters will
appear as you type.

Most everything that follows will require root access.
```
$ sudo su
[sudo] password for <your-username>:
Canonical Ledgers, LLC
ip-10-10-2-26
[10:53:10] root@ip-10-10-2-26:/home/<your-username>
#
```

### Set Hostname
Set the fully qualified domain name for the server. This should be the primary
domain name used to reach the host. You will have been told what this should be
before starting this process.
```
# hostnamectl set-hostname <fully-qualified-domain-name>
```
If you log out and log back in the hostname should be properly displayed in
your bash prompt.

### ZFS
If this server will become a factom node we need to create the zpool on the
second disk.

#### Verify/Create Disk Partitions
Verify that the disk is properly attached and that there is a single partition
that includes the entire disk space.
```
# lsblk
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda    202:0    0   8G  0 disk
└─xvda1 202:1    0   8G  0 part /
xvdb    202:16   0  32G  0 disk
└─xvdb1 202:17   0  32G  0 part
```
We can see that the disk is called `xvdb` and it has a single partition that
covers the entire 32G disk.

If `xvdb` is not present then a step was missed when launching the ec2
instance. You will need to create a new volume in the same availability zone as
this ec2 instance and attach it to this instance.

If `xvdb1` is either not present or its `SIZE` does not match the size of
`xvdb` then it will need to be partitioned.

**The following operation is destructive! If you aren't sure that you should
repartition please ask!**

Use `fdisk` to partition the disk.
```
# fdisk /dev/xvdb
```
We will create a single partition that spans the entire disk with a partition
type of 'Solaris Root' to be compatible with ZFS.

Enter the following commands hitting `[enter]` after each line in addition to
any `[enter]` specified here:
```
g
n
[enter]
[enter]
[enter]
t
47
p
w
```

The `p` command prints the partition table before we write it with `w`. This is
your chance to make sure you didn't make a mistake.

**Use your eyes and brain! Read the output carefully and ask questions if you
aren't sure!**

The output will look like the following:
```
# fdisk /dev/xvdb

Welcome to fdisk (util-linux 2.32).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): g
Created a new GPT disklabel (GUID: F19BC6AB-4780-1C41-A6AD-422AF9DC78C6).

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-67108830, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-67108830, default 67108830):

Created a new partition 1 of type 'Linux filesystem' and of size 32 GiB.

Command (m for help): t
Selected partition 1
Partition type (type L to list all types): 47
Changed type of partition 'Linux filesystem' to 'Solaris root'.

Command (m for help): p
Disk /dev/xvdb: 32 GiB, 34359738368 bytes, 67108864 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: F19BC6AB-4780-1C41-A6AD-422AF9DC78C6

Device     Start      End  Sectors Size Type
/dev/xvdb1  2048 67108830 67106783  32G Solaris root

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

#### Create the zpool
Find the partition uuid for the `xvdb1` disk partition.
```
$ ls -l /dev/disk/by-partuuid
total 0
lrwxrwxrwx 1 root root 11 May  9 11:34 14cf3f28-08a6-6741-8f97-1d6b8ea619b6 -> ../../xvdb1
lrwxrwxrwx 1 root root 11 May  8 17:20 e4126fe1-01 -> ../../xvda1
```
Create a new zpool called `zdock` using this partition uuid. This pool will
hold all of our docker volumes that run factomd.
```
# zpool create -m none -O compression=lz4 zdock 14cf3f28-08a6-6741-8f97-1d6b8ea619b6
# zpool list
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zdock  31.8G   116K  31.7G         -     0%     0%  1.00x  ONLINE  -
```

#### Create the ZFS datasets
We will create one dataset for the entire `/var/lib/docker` folder and an
additional dataset specifically for `/var/lib/docker/volumes` which is where
our factom database will be written.

```
# zfs create -o mountpoint=/var/lib/docker zdock/docker
# zfs create zdock/docker/volumes
# zfs list
NAME                   USED  AVAIL  REFER  MOUNTPOINT
zdock                  162K  30.8G    24K  none
zdock/docker            48K  30.8G    24K  /var/lib/docker
zdock/docker/volumes    24K  30.8G    24K  /var/lib/docker/volumes
# zfs get compression
NAME                  PROPERTY     VALUE     SOURCE
zdock                 compression  lz4       local
zdock/docker          compression  lz4       inherited from zdock
zdock/docker/volumes  compression  lz4       inherited from zdock
```
Use the last two commands verify the correct mountpoint and the compression
property.

### Docker
If this server will be a factom node you will need docker.

Check whether docker is already installed.
```
$ pacman -Q docker
docker 1:18.04.0-1
```
If instead you get the output `error: package 'docker' was not found`, install
it with
```
# pacman -Sy docker
```
Enable the docker service.
```
# systemctl enable docker
```

### Enable Telegraf

Telegraf is used on all systems for full system monitoring. Enable it to start
on boot.
```
# systemctl enable telegraf
```

#### Install additional telegraf config packages
If this will be a factom node, you should install the docker and factom
telegraf monitoring configs.

```
# pacman -Sy telegraf-conf-input-factomd telegraf-conf-input-docker
```

### Update and reboot
Perform a full system update and reboot.
```
# pacman -Syu
```
Hit enter when prompted to confirm the update.

**Always review the output for any messages or errors after the update
completes or else breakage may occur!** Always ask if anything appears out of
the ordinary.

Reboot the system.
```
# reboot
```
In about a minute or two attempt to login again. The new hostname should appear
in the welcome message.
