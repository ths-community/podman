# podman to run gICS

gICS - like all tools provided by "(unabhängige) Treuhandstelle Greifswald" (https://www.ths-greifswald.de/en/researchers-general-public/gics/) comes in a 
containerized format. However - as of this writing - all scripts and documentation refer to docker as runtime - which is OK for workstations
but only a 2nd best choice when it comes to servers, 24/7 operations, small resource footprint (= costs) and attack surface and executable disaster recovery plans.

The purpose of this article is to demonstrate how gICS can be operated in a more server-like environment optimized for better security and 
low resource footprint for 24/7 operations with a simple disaster recovery plan.

## disclaimer

As with all community contributions this is neither a fully-supported product nor in any way associated with my employer or any other company or legal entity.
It may contain bugs and may lack handling of certain situations and incidents. It may have been reduced and simplified for clarity to understand
the concepts rather than provide a bullet-proof solution.


## what is podman? 

There are plenty of much better and more comprehensive articles out there which explain who has initiated podman as a docker drop in replacement and why -
for the purpose of this community contribution I would like to highlight first the *common ground* that both docker and podman both use container runtime
libraries based on the OCI standard - this provides a huge level of compatibilities between both options.

however, podman addresses and fixed a few by-design issues from docker like

- need of a root-privilegded service
- potentially insecure communication between containers by docker.sock
- insecurities in multi-user setups

which is why especially Enterprise Linux distros like RedHat (incl Fedora) and SUSE decided to stop support for docker in favor of podman.

However, podman is available on all major Linux distros (caution it may come in *outdated* versions when installed by the *standard* package manager)
as well as docker (CE = community Edition) can still be installed on Enterprise Linux at your own risk / without vendor support

Frankly speaking many concepts for podman are rather a kind of backport from kubernetes - paired with the simplicity of docker

please refer to  

https://indico.cern.ch/event/757415/contributions/3421994/attachments/1855302/3047064/Podman_Rootless_Containers.pdf

for instance as one out of many good articles regarding this topic - for more detailed docker/podman comparision and reasoning

## install podman on fedora CoreOS

The easiest way to install podman is to install a host operating system in which it is already included by default - like CoreOS (nowadays also referred as
FCOS = Fedora Core OS). As the name indicates it is now a variant of Fedora to provide a secure minimal Linux designed to run container workloads (with podman by default) -
CoreOS is also used by RedHat Openshift - https://www.redhat.com/en/technologies/cloud-computing/openshift/what-was-coreos

Key features are a minimized Linux with current kernel and fixes and hence low attack surface. After installation most parts are readonly and updates are
applied to a layered filesystem (like in containers itself) - this makes rollback easy in case of issues and ensures integrity of (most of the) code.
And the built-in updater service (zincati) installs updates automatically usually once or twice a month - and can be paused in case of issues.

You will probably miss a graphically user interface or a package manager which allows to install more packages - both are against the basic concepts.
E.g. if a python3 runtime is needed then a container containing python3 is spin up to do the processing - instead of installing python3 into the OS itself.  
Even for debugging in CoreOS itself it uses a special debug container. It is cleaner - and easier to get rid of things which are no longer needed.

There is a very similar alternative to Fedora (IBM RedHat) CoreOS which is "flatcar Linux" by "kinvolk.io" (now a Microsoft(!) company)

 
### CoreOS download 



go to 

https://fedoraproject.org/

and scroll down on that page - Fedora CoreOS should appear in the list of various options and flavours - likely at the very end of the page

click "download now" and pay attention that the version is "stable" and your architecture is likely "x86_64" and "bare metal" - "Live DVD ISO"
is the correct version if you are operating on own hardware - and not in any of the cloud environments.

please remember where exactly this ISO file was downloaded



### CoreOS - prepare login keys

"security first" requires that login by default is key-based - not username/password (which is considered vastly insecure).
There is an option for username/password for the local console only - which is out of scope here

so the first thing to do is create a public/private keypair 

and then add the public key to a so-called ignition file which is used to automatically configure Fedora CoreOS during install.
Ignition file can be used to apply many different configuration setting - but we will only focus here for the login key as the bare minimum.

`ssh-keygen -f coreos`

which creates two files in the current directory

```
coreos
coreos.pub

```

First you may want to copy away the file "coreos" which contains the private key to a safe place.

### CoreOS - prepare ignition file for basic config

To continute in the installation of CoreOS we need to prepare a so-called 'ignition' file (find a version *without* key content in folder 'ignition').
Then the *public* key must be added to the ignition file (copy the full file content in [ ] at "sshAuthorizedKeys" )

```

{
  "ignition": {
    "version": "3.2.0"
  },
  "passwd": {
    "users": [
      {
        "groups": [
          "docker",
          "wheel",
          "sudo"
        ],
        "name": "core",
        "sshAuthorizedKeys": [
          "ssh-rsa AAAAB3NzaC1y...your-public-key-in-this-line...er7Q7/7 coreos"
        ]
      }
    ]
  }
}

```

### CoreOS - put ignition file onto a webserver

The CoreOS installtion procedure will need to find the ignition file in the network - on a webserver. This can be any webserver either
in your local network or in the Internet. You may want to check with a browser that this file is delivered from an URL like:

`https://anyserver.org/assets/config.ign`

**TIPP**

If you don't have a webserver you can use github.com for this purpose.

To do this ... create a github account on github.com for free if you do not have one already. 

Then go this community workspace

`https://github.com/ths-community/podman`

and use the 'fork' function to get 1:1 copy - then you can change the ignition file in the 'config' folder and add you public key.
After the file is committed (saved) you will get the full ignition file with URL

`https://raw.githubusercontent.com/YOUNAME/main/ignition/config.ign`

where YOURNAME is the name of your github account.

Note: You may feel uncomfortable to put a public key to a public webserver - but this is what public keys are made for.
However, there is a (very small) risk that if a hacker comes in possession ob private(!) keys they may use those to test which public key will fit
and then use this knowledge to made it part of an attack plan. But if a hacker has your private keys already then publicly visible public keys
are your least problem then.

### CoreOS - install

CoreOS is made for *server* use-cases - here is a word of warning:

**CAUTION!!!** - the installtion procedure will always grab the full harddisk and overwrite existing content without any further warning.
It is *NOT* made for co-existence with other operating systems like Windows, Linux oder MacOS which you know from desktop setups.

There are two easily ways to deal with this "special feature"

* Use a type1 hypervisor like Proxmox, Hyper-V oder ESXi - or at least VirtualBox or VMware player/workstation - to provide a virtual machine with seperated virtual disks
* Use a seperate hardware PC or notebook which does NOT contain anything (any more) that is still needed

In a hypervisior / virtual target machine you can configure to use the ISO as virtual DVD / CDROM -

In case of a real hardware (called: bare metal) you need to burn the ISO file onto a DVD (not recommended) or
make a raw copy to an USB device (recommended) - to make such a bootable USB stick use a tool like "rufus" under windows - or in Linux it is

`dd bs=100M if=filename.iso of=/dev/sdX`

where `/dev/sdX` is usually a device like `/dev/sde` - you can insert the USB stick into the machine and call `sudo dmesg` and at the end of the log you 
can easily see which device was last inserted into the system
 
Next power on your virtual - or real hardware - machine. YOu should see the "Fedora CoreOS" boot screen 
which automatically boots after a few seconds. Wait for the boot to complete - which is when it allows you to enter commands like `ip a`
to check the network configuation.

### CoreOS - shutdown after install

As printed on the screen the installer to complete the installation requires to enter:

`sudo coreos-installer install /dev/sda --ignition-url https://YOUR.URL/config.ign`

if you use github.com as webserver then it would look like this:

`sudo coreos-installer install /dev/sda --ignition-url https://raw.githubusercontent.com/YOUNAME/main/ignition/config.ign`

after the installation is complete then shutdown the server with 

`init 0`

### CoreOS - first boot after install

remove the USB stick from the server - or remove CD/DVD ISO from the virtual machine config - and start the server again.

This boot should start as before but now shows a 'login' message - don't try to login here, it is useless because we have not configured a password.
Instead login via network - you should also see the IP address of this server on the console - then go to another machine which has the private(!) key
file 'coreos' and login over network via key:

`ssh -i coreos IP-ADDR`

where IP-ADDR is the ip address shown on the CoreOS boot screen e.g. 192.168.55.34

### CoreOS - add persistant volume

This step is not necessary but strongly recommended for disaster reovery - as described above every CoreOS install completely overwrites the
entire bootdisk - hence it is recommended to have a second disk to hold the data - which is seperated from the bootdisk.
This 'data' disk can also be a high-quality USB Disk - only use one which allows a high number of write operations e.g. SanDisk Ultra (Extreme).

On virtual machine just configure a second virtual disk, size should be 30GB

Either way (second disk is a real harddisk, high-quality USB Stick or a virtual disk) assuming this data disk appears as device /dev/sdb
- in case of doubt check disk in output of `sudo dmesg`

**Partioning**

enter `sudo fdisk /dev/sdb` then add a primary partition - on a blank device this goes by 'n'(ew) > 1 (first partition) > 'p'(rimary) and accept suggested 
start cylinder and size - you may want to review your setting with 'p'(rint) - when it is OK press 'w'(rite) to commit the change and write 
this data to the disk

**create filesystem**

enter `sudo mkfs /dev/sdb1` to create a standard filesystem on this data disk - the 1 in the device name refers to the partition number 1 you just created

**create mount directory**

enter `sudo mkdir /var/disk1` to create an empty directory to which the data disk will be mounted. It is important to locate this in `/var/`
because most parts of the boot disk is read-only and hence does not allow creation of directories.

**automount data disk at boot**

create a file `/etc/fstab` (find sample in folder 'etc' here in this repo) with the following content:

`/dev/sdb1 /var/disk1 ext4 defaults 0 0`

e.g. with

`sudo echo '/dev/sdb1 /var/disk1 ext4 defaults 0 0' >> /etc/fstab`

or your favorite text editor (e.g. vi, vim, nano)

**test mount**

now mount the new connection to the data disk with

`sudo mount /var/disk1`

check if the second disk is added correctly - e.g. by

`df -k | grep sd`

should show that `/dev/sda` (boot disk) as well as `/dev/sdb` (data disk) is correctly added to the system.

create a new directory on this data disk

`sudo mkdir /var/disk1/containers`

which will host all persistant data of our containers - and ...

**change ownership**

and change ownership to 'core' which is the default unprivileged user in CoreOS - which will also run all containerized workloads in order to
avoid use of 'root'

`sudo chown core /var/disk1/containers`
 
**login setting**

there is one more thing which we need to do in order to allow processes started by regular user 'core' to remain existant after logoff:

`sudo loginctl enable-linger core`

after this setting we will use 'sudo' or 'root' access rarely and most actions will be run under user 'core' even create/start/stop/remove of containers.

**reboot**

in order to test that everything is setup correctly please reboot again now with

`sudo reboot`


## install podman on a different Host OS 

your host OS may have already package to install "podman" easily - but note that these packages may contain older version -
"podman" is actively delevoped and new features and versions are issued frequently.
Please go to https://podman.io or use your favorite search engine to find recent installation packages.

