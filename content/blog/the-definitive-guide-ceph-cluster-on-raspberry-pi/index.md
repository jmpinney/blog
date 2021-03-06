+++
title = "The Definitive Guide: Ceph Cluster on Raspberry Pi"
date = "2020-02-17 20:58:50"
publishdate = "2015-05-13 01:35:35"
description = "Learn how to deploy a ceph cluster on a Raspberry Pi. Commodity off the shelf hardware meets a budget."
toc = true
tags = ['arm', 'ceph', 'rados', 'raspberry-pi', 'storage']
+++

A Ceph cluster on Raspberry Pi is an awesome way to create a RADOS home storage
solution (NAS) that is highly redundant and low power usage. It's also a low
cost way to get into Ceph, which may or may not be the future of storage
(software defined storage definitely is as a whole). Ceph on ARM is an
interesting idea in and of itself. I built one of these as a development
environment (playground) for home.  It can be done on a relatively small budget.
Since this was a spur of the moment idea, I purchased everything locally. I
opted for the
[Raspberry Pi 2 B](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/)
(for the 4 cores and 1GB of RAM). I'd really recommend going with the Pi 2 B, so
you have one core and 256MB RAM for each USB port (potential OSD). In this guide
I will outline the parts, software I used and some options that you can use for
achieving better performance. This guide assumes you have access to a Linux PC
with an SD card reader. It also assumes you have a working knowledge of Linux in
general and a passing familiarity with Ceph.

## Parts

Although I will explain many options in this guide, this is the minimum you will
need to get a cluster up and running, this list assumes 3 Pi nodes.

- 3 x 3ft Cat6 Cables
- 3 x Raspberry Pi 2 B
- 3 x Raspberry Pi 2 B Case
- 3 x 2 Amp Micro USB Power Supply
- 3 empty ports on a gigabit router
- 3 x Class 10 MicroSD (16GB or more) for OS drive
- 3-12 x USB 2.0 Flash Drives (at least 32GB, better drive for better
  performance)

I used 3 x 64GB flash drives, 3 x 32GB MicroSD and existing ports on my router.
My cost came in at about \$250. You can add to this list based on what you add
to your setup throughout the guide, but this is pretty much the minimum for a
fully functional Ceph cluster.

## Operating System

Raspbian. The testing repository for Raspbian has the many packages of Ceph
0.80.9 and dependencies pre-compiled. Everything you'll need for this tutorial
and is the "de facto" OS of choice for flexibility on Raspberry Pi. You can
download the Raspbian image
here: [Raspbian Download](http://downloads.raspberrypi.org/raspbian_latest).
Once you have the image, you'll want to put it on an SD card. For this
application I recommend using at least a 16GB MicroSD card (Class 10
preferably - OS drive speed matters for Ceph monitor processes). To transfer the
image on Linux, you can use DD. run the lsblk command to display your devices
once you've inserted the card into your card reader. Then you can use dd  to
transfer the image to your SD. The command below assumes the image name is
raspbian-wheezy.img  and that it lives in your present working directory. The
above command also assumes that your SD card is located at /dev/mmcblk0 adjust
these accordingly and make sure that your SD card doesn't contain anything
important and is empty.

`sudo dd bs=4M if=raspbian-wheezy.img of=/dev/mmcblk0`

This command will take a few minutes to complete. Once it does run sync to flush
all cache to disk and make sure it is safe to remove the device. You'll then
boot up into Raspbian, re-size the image to the full size of your MicroSD, set a
memorable password, overclock if you want. Once this is done there are a few
modifications to make. We'll get into this in the installation section below. I
don't recommend using too large of a MicroSD as later in this tutorial we will
image the whole OS from our first MicroSD for deployment to our other Pi nodes.

## Hardware Limitations

The first limitation to consider is overall storage space. Ceph OSD processes
require roughly 1MB of RAM per GB of storage. Since we are co-locating monitor
processes the effective storage limitation is 512GB per Pi 2 B (4 x 128GB
sticks) RAW (before Ceph replication or erasure coding overhead). Network speed
is also a factor as discussed later in document. You will hit network speed
limitations before you hit the speed limitations of the Pi 2 B's single USB 2.0
bus (480Mbit).

## Network

In this setup I used empty ports on my router. I run a local DNS server on my
home router and use static assignments for local DNS. You may want to consider
just using a flat 5 or 8 port (depending on number of nodes you plan to have)
gigabit switch for the cluster network and WiPi modules for the public
(connected to your router via WiFi). The nice thing about using a flat layer 2
switch is that if all the Pi nodes are in the same subnet, you don't have to
worry about a gateway and it also keeps the cost down (compared to using router
ports) while reducing the network overhead (for Ceph replication) on your home
network. Using a dedicated switch for the cluster network will also increase
your cluster performance, especially considering the 100Mbit limitations of the
Pi 2 B's network port. By using
a [BGN Dongle for Pi](http://thepihut.com/products/usb-wifi-adapter-for-the-raspberry-pi)  and
a dedicated switch for the cluster network, you will get a speedier cluster.
This will use one of your 4 USB ports and thus, you will get one less OSD per
Pi. Keep in mind, depending on if you use replication or erasure coding
private traffic can be 1-X times greater then client IO  (X being 3 in a
standard replication profile) if that matters for your application. Of course
this is all optional and for additional "clustery goodness". It really depends
on budget, usage - etcetera.

## Object Storage Daemons

In this guide, I co-located OSD journals on the OSD drives. For better
performance, you can use a faster USB like the SanDisk Extreme 3.0 (keep in mind
that you'll be limited by the 60MB/s speed of USB 2.0). Using a dedicated
(faster) journal drive will yield much better performance. But you don't really
need to worry about it unless you are using multiple networks as outlined above.
If you are not, 4 decent USB sticks will saturate your 100Mbit NIC per node.
There is a lot more to learn about Ceph architecture that I cover in this
article and I highly recommend you do so [here](http://ceph.com/docs/v0.80.5/).

### OSD Filesystem

XFS is the default in Ceph Firefly. I prefer
[BTRFS](https://btrfs.wiki.kernel.org/index.php/Main_Page) as an OSD filesystem
for multi-fold reasons and I use it in this tutorial.

## Installation

Assuming you have setup your network and operating system - have 3 nodes and the
hardware you want to use - we can begin. The first thing to do is wire up power
and network as you see fit. After that, you'll want to run through the initial
raspi-config on what will become your admin node. Then it's time to make some
changes. Once your admin node is booted and configured, you have to edit
/etc/apt/sources.list . Raspbian Wheezy has archaic versions of Ceph in the main
repository, but the latest firefly version in the testing repository. Before we
delve into this, I find it useful to install some basic tools and requirements.
Connect via SSH or directly to terminal and issue this command from the Pi:

`sudo apt-get install vim screen htop iotop btrfs-tools lsb-release gdisk`

From this point forward we will assume you are connecting to your Pi nodes via
SSH. You've just installed BTRFS-tools, vim (better then vi) and some
performance diagnostics tools I like. Now that we have vim  it's time to edit
our sources:

`vi /etc/apt/sources.list`

You'll see the contents of your sources file. Which will look like this:

`deb http://mirrordirector.raspbian.org/raspbian/ wheezy main contrib non-free rpi`

### Uncomment line below then 'apt-get update' to enable 'apt-get source'

`#deb-src http://archive.raspbian.org/raspbian/ wheezy main contrib non-free rpi`

Modify it to look like this:

`deb http://mirrordirector.raspbian.org/raspbian/ testing main contrib non-free rpi`

### Uncomment line below then 'apt-get update' to enable 'apt-get source'

`#deb-src http://archive.raspbian.org/raspbian/ testing main contrib non-free rpi`

We've replaced wheezy  with testing .Once this is done, then issue this command:

`sudo apt-get update`

Once this process has completed is time to start getting the OS ready for Ceph.
Everything we do in this section up to the point of imaging the OS is needed for
nodes that will run Ceph. First we will create a ceph user and give it
password-less sudo access. To do so issue these commands:

```bash
ssh user@ceph-server
sudo useradd -d /home/ceph -m ceph
sudo passwd ceph
```

Set the password to a memorable one as it will be used on all of your nodes in
this guide. Now we need to give the ceph user sudo access

```bash
echo "ceph ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph
sudo chmod 0440 /etc/sudoers.d/ceph
```

We'll be using ceph-deploy later and it's best to have a defult user to login as
all the time. Issue this command:

```bash
mkdir -p ~/.ssh/
```

Then create this file using vi:

```bash
vi ~/.ssh/config
```

I assume 3 nodes in this tutorial and a naming convention of piY, where Y is the
node number starting from 1.

```
Host pi1
Hostname pi1
User ceph
Host pi2
Hostname pi2
User ceph
Host pi3
Hostname pi3
User ceph
```

Save the file and exit. As far as hostnames, you can use whatever you want of
course. As I mentioned, I run local DNS and DHCP with static assignments. If you
do not, you'll need to edit /etc/hosts  so that your nodes can resolve
each-other. You can do this after the OS image, as each node will have a
different IP. Now it's time to install the ceph-deploy tool. Raspbian wget  can
be strange with HTTPS so we will ignore the certificate (do so at your own
peril):

```bash
wget --no-check-certificate -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob\_plain;f=keys/release.asc' | sudo apt-key add -
echo deb http://ceph.com/debian-firefly/ wheezy main | sudo tee /etc/apt/sources.list.d/ceph.list
```

Now that we've added the Ceph repository, we can install ceph-deploy:

`sudo apt-get update && sudo apt-get install ceph-deploy ceph ceph-common`

Since we are installing ceph from the Raspbian repositories, we need to change
the default behavior of ceph-deploy:

`sudo vi /usr/share/pyshared/ceph_deploy/hosts/debian/install.py`

Change

```
def install(distro, version_kind, version, adjust_repos):
codename = distro.codename
machine = distro.machine_type
```

To

```
def install(distro, version_kind, version, adjust_repos):
adjust_repos = False
codename = distro.codename
machine = distro.machine_type
```

This will prevent ceph-deploy from altering repos as the Ceph armhf (Rasberry
Pi's processor type) repos are mostly empty. Finally, we should revert the
contents of /etc/apt/sources.list :

`sudo vi /etc/apt/sources.list`

You'll see the contents of your sources file. Which will look like this:

```
deb http://mirrordirector.raspbian.org/raspbian/ testing main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://archive.raspbian.org/raspbian/ testing main contrib non-free rpi
```

Modify it to look like this:

```
deb http://mirrordirector.raspbian.org/raspbian/ wheezy main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://archive.raspbian.org/raspbian/ wheezy main contrib non-free rpi
```

We've replaced testing  with wheezy .Once this is done, then issue this command:

```
sudo apt-get update
```

## Kernel Tweaks

We are also going to tweak some kernel parameters for better stability. To do so
we will edit /etc/sysctl.conf .

`vi /etc/sysctl.conf`

At the bottom of the file, change add the following lines:

```
vm.swappiness=1
vm.min_free_kbytes = 32768
kernel.pid_max = 32768
```

## Imaging the OS

Now we have a good baseline for deploying ceph to our other Pi nodes. It's time
to stop our admin node and image the drive (MicroSD). Issue:

`sudo halt`

Then unplug power to your Pi node and remove the MicroSD. Insert the microSD in
your SD adapter, then the SD adapter into your Linux PC. You'll need at least as
much free drive space on your PC as the size of the MicroSD card.Where
/dev/mmcblk0 is your SD card and pi-ceph.img is your image destination, run:

`sudo dd if=/dev/mmcblk0 of=ceph-pi.img bs=4M`

This can take a vary long time depending on the size of your SD and you can
compress it with gzip  or xz  for long term storage (empty space compresses
really well it turns out). Once the command returns, run sync  to flush the
cache to disk and make sure you can remove the MicroSD

## Imaging Your Nodes OS Drives

Now that you have a good baseline image on your PC, you are ready to crank out
"Ceph-Pi" nodes - without redoing all of the above. To do so, insert a fresh
MicroSD into your adapter and then PC. Then assuming ceph-pi.img  is your OS
image and /dev/mmcblk0 is your MicroSD card run:

`sudo dd if=ceph-pi.img of=/dev/mmcblk0 bs=4M`

Repeat this for a many nodes as you intend to deploy.

## Create a Ceph Cluster on Raspberry Pi

Insert your ceph-pi MicroSD cards into your Pi nodes and power them all on.
You've made it this far, now it's time to get "cephy". Deploying with
ceph-deploy is a breeze. First we need to SSH to our admin node, make sure you
have setup IPs, network and /etc/hosts on all Pi nodes if you are not using
local DNS and DHCP with static assignments. We need to generate and distribute
an SSH key for password-less authentication between nodes. To do so run (leave
the password blank):

```
ssh-keygen
Generating public/private key pair.
Enter file in which to save the key (/ceph-client/.ssh/id\_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /ceph-client/.ssh/id\_rsa.
Your public key has been saved in /ceph-client/.ssh/id\_rsa.pub.

Now copy the key to all nodes (assuming 3 with the naming convention from above):

ssh-copy-id ceph@pi1
ssh-copy-id ceph@pi2
ssh-copy-id ceph@pi3
```

You will be prompted for the password you created for the ceph user each time to
establish initial authentication. Once that is done and you are connected to
your admin node (1st node in the cluster) as the pi user you'll want to create
an admin node directory:

```
mkdir -p ~/ceph-pi-cluster
cd ~/ceph-pi-cluster
```

### Creating an initial Ceph Configuration

We are going to create an initial Ceph configuration assuming all 3 pi nodes as
monitors. If you have more, keep in mind - you always want an odd number of
monitors to avoid a
[split-brain](<http://en.wikipedia.org/wiki/Split-brain_(computing)>) scenario.
To to this run:

`ceph-deploy new pi1 pi2 pi3`

Now there are some special tweaks that should be made for best stability and
performance within the hardware limitations of the Raspberry Pi 2 B. To apply
these changes we'll need to edit the ceph.conf here on the admin node before it
is distributed. To do so:

`vi ~/ceph-pi-cluster/ceph.conf`

After the existing lines add:

```ini
# Disable in-memory logs
debug_lockdep = 0/0
debug_context = 0/0
debug_crush = 0/0
debug_buffer = 0/0
debug_timer = 0/0
debug_filer = 0/0
debug_objecter = 0/0
debug_rados = 0/0
debug_rbd = 0/0
debug_journaler = 0/0
debug_objectcatcher = 0/0
debug_client = 0/0
debug_osd = 0/0
debug_optracker = 0/0
debug_objclass = 0/0
debug_filestore = 0/0
debug_journal = 0/0
debug_ms = 0/0
debug_monc = 0/0
debug_tp = 0/0
debug_auth = 0/0
debug_finisher = 0/0
debug_heartbeatmap = 0/0
debug_perfcounter = 0/0
debug_asok = 0/0
debug_throttle = 0/0
debug_mon = 0/0
debug_paxos = 0/0
debug_rgw = 0/0
osd heartbeat grace = 8

[mon]
mon compact on start = true
mon osd down out subtree_limit = host

[osd]

# Filesystem Optimizations

osd journal size = 1024

# Performance tuning

max open files = 327680
osd op threads = 2
filestore op threads = 2

#Capacity Tuning
osd backfill full ratio = 0.95
mon osd nearfull ratio = 0.90
mon osd full ratio = 0.95

# Recovery tuning

osd recovery max active = 1
osd recovery max single start = 1
osd max backfills = 1
osd recovery op priority = 1

# Optimize Filestore Merge and Split

filestore merge threshold = 40
filestore split multiple = 8
```

### Creating Initial Monitors

Now we can deploy our spiffy ceph.conf, create our initial monitor daemons,
deploy our authentication keyring and chmod it as needed. We will be deploying
to all 3 nodes for the purposes of this guide:

```
ceph-deploy mon create-initial
ceph-deploy admin pi1 pi2 pi3
for i in pi1 pi2 pi3;do ssh $i chmod 644 /etc/ceph/ceph.client.admin.keyring;done
```

### Creating OSDs (Object Storage Daemons)

Ready to create some storage? I know I am. Insert your USB keys of choice into
your Pi USB ports. For the purposes of this guide I will be deploying 1 OSD (USB
key) per Pi node. I will also be using the BTRFS filesystem and co-locating the
journals on the OSDs with a default journal size of 1GB (assuming 2 \* 40MB/s
throughput max and a default filestor max sync interval of 5). This value is
hard coded into our ceph-pi config above. The formula is:

`osd journal size = {2 * (expected throughput * filestore max sync interval)}`

So let's deploy our OSDs. Once our USBs are plugged in, use lsblk to display the
device locations. To make sure our drives are clean and have a GPT partition
table, use the gdisk  command for each OSD on each node. Assuming /dev/sda  as
our OSD: gdisk /dev/sda Create a new partition table, write it to disk and exit.
Do this for each OSD on each node. You can craft a bash for  loop if you are
feeling "bashy" or programmatic. Once all OSD drives have a fresh partition
table you can use ceph-deploy to create your OSDs (using BTRFS for this guide)
where pi1 is our present node and /dev/sda is the OSD we are creating:

`ceph-deploy osd create --fs-type btrfs pi1:/dev/sda`

Repeat this for all OSD drives on all nodes (or write a for loop). Once you've
created at least 3 you are ready to move on.

## Checking Cluster Health

Congratulations! You should have a working Ceph-Pi cluster. Trust, but verify.
Get the health status of your cluster using this command:

`ceph -s`

and for a less verbose output

`ceph health`

## What to do now?

Use your storage cluster! Create an RBD, mount it - export NFS or CIFS. There is
a lot of reading out there. Now you know how to deploy a Ceph cluster on
Raspberry Pi.
