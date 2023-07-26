# podman to run gICS

gICS - like all tools provided by "(unabhängige) Treuhandstelle Greifswald" (https://www.ths-greifswald.de/en/researchers-general-public/gics/) comes in a 
containerized format. However - as of this writing - all scripts and documentation refer to docker as runtime - which is OK for workstations
but only a 2nd best choice when it comes to servers, 24/7 operations, small resource footprint (= costs) and attack surface and executable disaster recovery plans.

The purpose of this article is to demonstrate how gICS can be operated in a more server-like environment optimized for better security and 
low resource footprint for 24/7 operations with a simple disaster recovery plan.

## disclaimer

As with all community contributions this is neither a fully supported product nor in any way associated with my employer or any other company or legal entity.
It may contain bugs and may lack handling of certain situations and incidents. It may have been reduced and simplified for clarity to understand
the concepts rather than provide a bullet-proof solution.


## what is podman? 

There are plenty of much better and more comprehensive articles out there which explain who has initiated podman as a docker drop-in replacement and why -
for the purpose of this community contribution, I would like to highlight first the *common ground* that both docker and podman both use container runtime
libraries based on the OCI standard - this provides a huge level of compatibilities between both options.

however, podman addresses and fixed a few by-design issues from docker like

- need of a root-privileged service
- potentially insecure communication between containers by docker.sock
- insecurities in multi-user setups

which is why especially Enterprise Linux distros like RedHat (incl Fedora) and SUSE decided to stop support for docker in favor of podman.

However, podman is available on all major Linux distros (caution it may come in *outdated* versions when installed by the *standard* package manager)
as well as docker (CE = community Edition) can still be installed on Enterprise Linux at your own risk / without vendor support

Frankly speaking many concepts for podman are rather a kind of backport from kubernetes - paired with the simplicity of docker

please refer to  

https://indico.cern.ch/event/757415/contributions/3421994/attachments/1855302/3047064/Podman_Rootless_Containers.pdf

for instance, as one out of many good articles regarding this topic - for more detailed docker/podman comparison and reasoning

## install podman on fedora CoreOS

The easiest way to install podman is to install a host operating system in which it is already included by default - like CoreOS (nowadays also referred as
FCOS = Fedora Core OS). As the name indicates it is now a variant of Fedora to provide a secure minimal Linux designed to run container workloads (with podman by default) -
CoreOS is also used by RedHat Openshift - https://www.redhat.com/en/technologies/cloud-computing/openshift/what-was-coreos

Key features are a minimized Linux with current kernel and fixes and hence low attack surface. After installation most parts are read-only and updates are
applied to a layered filesystem (like in containers itself) - this makes rollback easy in case of issues and ensures integrity of (most of the) code.
And the built-in updater service (zincati) installs updates automatically usually once or twice a month - and can be paused in case of issues.

You will probably miss a graphically user interface or a package manager which allows to install more packages - both are against the basic concepts.
E.g. if a python3 runtime is needed then a container containing python3 is spin up to do the processing - instead of installing python3 into the OS itself.  
Even for debugging in CoreOS itself it uses a special debug container. It is cleaner - and easier to get rid of things which are no longer needed.

There is a very similar alternative to Fedora (IBM RedHat) CoreOS which is "flatcar Linux" by "kinvolk.io" (now a Microsoft(!) company)

 
### CoreOS download 



go to 

https://fedoraproject.org/

and scroll down on that page - Fedora CoreOS should appear in the list of various options and flavors - likely at the very end of the page

click "download now" and pay attention that the version is "stable" and your architecture is likely "x86_64" and "bare metal" - "Live DVD ISO"
is the correct version if you are operating on own hardware - and not in any of the cloud environments.

please remember where exactly this ISO file was downloaded



### CoreOS - prepare login keys

"security first" requires that login by default is key-based - not username/password (which is considered vastly insecure).
There is an option for username/password for the local console only - which is out of scope here

so the first thing to do is create a public/private keypair 

and then add the public key to a so-called ignition file which is used to automatically configure Fedora CoreOS during install.
Ignition file can be used to apply many different configuration settings - but we will only focus here for the login key as the bare minimum.

`ssh-keygen -f coreos`

which creates two files in the current directory

```
coreos
coreos.pub

```

First you may want to copy away the file "coreos" which contains the private key to a safe place.

### CoreOS - prepare ignition file for basic config

To continue in the installation of CoreOS we need to prepare a so-called 'ignition' file (find a version *without* key content in folder 'ignition').
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

The CoreOS installation procedure will need to find the ignition file in the network - on a webserver. This can be any webserver either
in your local network or in the Internet. You may want to check with a browser that this file is delivered from an URL like:

`https://anyserver.org/assets/config.ign`

**TIPP**

If you don't have a webserver you can use github.com for this purpose.

To do this ... create a GitHub account on github.com for free if you do not have one already. 

Then go this community workspace

`https://github.com/ths-community/podman`

and use the 'fork' function to get 1:1 copy - then you can change the ignition file in the 'config' folder and add your public key.
After the file is committed (saved) you will get the full ignition file with URL

`https://raw.githubusercontent.com/YOUNAME/main/ignition/config.ign`

where YOURNAME is the name of your github account.

Note: You may feel uncomfortable to put a public key to a public webserver - but this is what public keys are made for.
However, there is a (very small) risk that if a hacker comes in possession of private(!) keys they may use those to test which public key will fit
and then use this knowledge to make it part of an attack plan. But if a hacker has your private keys already then publicly visible public keys
are your least problem then.

### CoreOS - install

CoreOS is made for *server* use-cases - here is a word of warning:

**CAUTION!!!** - the installation procedure will always grab the full hard disk and overwrite existing content without any further warning.
It is *NOT* made for co-existence with other operating systems like Windows, Linux or MacOS which you know from desktop setups.

There are two easily ways to deal with this "special feature"

* Use a type1 hypervisor like Proxmox, Hyper-V or ESXi - or at least VirtualBox or VMware player/workstation - to provide a virtual machine with separated virtual disks
* Use a seperate hardware PC or notebook which does NOT contain anything (anymore) that is still needed

In a hypervisor / virtual target machine you can configure to use the ISO as virtual DVD / CDROM -

In case of a real hardware (called: bare metal) you need to burn the ISO file onto a DVD (not recommended) or
make a raw copy to an USB device (recommended) - to make such a bootable USB stick use a tool like "rufus" under windows - or in Linux it is

`dd bs=100M if=filename.iso of=/dev/sdX`

where `/dev/sdX` is usually a device like `/dev/sde` - you can insert the USB stick into the machine and call `sudo dmesg` and at the end of the log you 
can easily see which device was last inserted into the system.
 
Next power on your virtual - or real hardware - machine. You should see the "Fedora CoreOS" boot screen 
which automatically boots after a few seconds. Wait for the boot to complete - which is when it allows you to enter commands like `ip a`
to check the network configuration.

### CoreOS - shutdown after install

As printed on the screen the installer to complete the installation requires to enter:

`sudo coreos-installer install /dev/sda --ignition-url https://YOUR.URL/config.ign`

if you use github.com as webserver then it would look like this:

`sudo coreos-installer install /dev/sda --ignition-url https://raw.githubusercontent.com/YOUNAME/main/ignition/config.ign`

after the installation is complete then shutdown the server with 

`init 0`

### CoreOS - first boot after install

remove the USB stick from the server - or remove CD/DVD ISO from the virtual machine config - and start the server again.

This boot should start as before but now shows a 'login' message - do not try to login here, it is useless because we have not configured a password.
Instead, use ssh to login via network - you should also see the IP address of this server on the console - then go to another machine which has the private(!) key
file 'coreos' and login over network via key:

`ssh -i coreos IP-ADDR`

where IP-ADDR is the IP address shown on the CoreOS boot screen e.g. 192.168.55.34

### CoreOS - add persistant volume

This step is not necessary but strongly recommended for disaster reovery - as described above every CoreOS install completely overwrites the
entire boot disk - hence it is recommended to have a second disk to hold the data - which is separated from the boot disk.
This 'data' disk can also be a high-quality USB Disk - only use one which allows a high number of write operations e.g. SanDisk Ultra (Extreme).

On virtual machine just configure a second virtual disk, size should be 30GB

Either way (second disk is a real harddisk, high-quality USB Stick or a virtual disk) assuming this data disk appears as device /dev/sdb
- in case of doubt check disk in output of `sudo dmesg`

**Partitioning**

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

which will host all persistent data of our containers - and ...

**change ownership**

and change ownership to 'core' which is the default unprivileged user in CoreOS - which will also run all containerized workloads in order to
avoid use of 'root'

`sudo chown core /var/disk1/containers`

**SELINUX attribute**

CoreOS has SELINUX enabled by default - we need to run a special command to tell SELINUX that volumes for containers will be created under this path

`chcon -t container_file_t -R /var/disk1/containers/`

this will (hopefully) be inherited to any newly created file and folders - if not, re-apply this command if you see issues in container logfiles,
especially write permissions denied. Background is that 'root' containers will be started in the UID of the regular user `core` but non-root containers
will be mapped to another UID e.g. 568528 which finds then shared folders without sufficient permissions (mainly 'write' is missing)
 
**login setting**

there is one more thing which we need to do in order to allow processes started by regular user 'core' to remain existent after logoff:

`sudo loginctl enable-linger core`

after this setting we will use 'sudo' or 'root' access rarely and most actions will be run under user 'core' even create/start/stop/remove of containers.

**reboot**

in order to test that everything is setup correctly please reboot again now with

`sudo reboot`

after successful reboot CoreOS is ready to use, has a separated data disk for easier disaster recovery and will automatically update at least once a month
for better security - and is ready to run our container workloads like gICS, keycloak or bunkerweb (WAF = web application firewall)

 
## install podman on a different Host OS 

In case you decide not to install CoreOS (or "flatcar Linux") to get a podman runtime you can still install it into any major Operating Systems (OS).
Maybe your current host OS (debian, ubuntu, etc) have already a podman package to install via your standard package manager easily - but note that these packages may contain an older version -
"podman" is actively developed and new features and versions are issued frequently.
Please go to https://podman.io or use your favorite search engine to find recent installation packages.

## a closer look on podman

### where podman is similar to docker

even fundamental concepts are different podman was developed as "drop in" replacement for docker - so as regular user you can use podman in `set podman=docker` mode.

Well known commands like `podman run`, `podman ps`, `podman stop` etc will work exactly like their respective counterparts in docker - also for regular users.
There is no need for such a user to be added to a docker group

### where podman is different

* resources like networks or namespaces are no longer shared on machine level - two regular users can run container workloads using the same names without any interference.
* there is no docker.sock - this may turn out as larger issue if you copy docker scripts or recipes to run under podman. Any access to docker.sock will fail.
There is an options to start a service which provides such a socket for compatibility reasons - usually it is not needed
* docker workloads are started automatically at boot time - for podman we will use systemd in user mode(!) to do the same (see examples in this repo)
* podman can bundle multiple container workloads in a "pod" - a concept well-known in kubernetes but unknown in docker - "pods" are described in YAML files similar
to kubernetes and are very useful in use-cases in which
an application like gICS or keycloak need a database to run - pods have an *internal* network in which any2any port can communicate 
* when a user logs off all containers will stop - use "loginctl enable-linger username" to avoid this and allow containers to continue running

## use-cases - general notes

please get this github repo by

`git clone https://github.com/ths-community/podman.git`

it contains a folder `config` - these contents should go under `~/.config` - please note any file or folder beginning with '.' are hidden by default in git and filesystem.
This is why in github it is named 'config' and must be renamed after cloning:

```
cd podman
# mv config ~/.config
# or (whatever you prefer)
cp -R config ~/.config
```

in this `~/.config` folder you will find the YAML files for various workloads to run as 'pod' - before you start anything check the YAML file for
passwords to be changed from `ChangeMe` to a unique secret one.

Note: podman recently got a new feature to handle 'secrets' similar to kubernetes - which is not yet reflected in these scripts as of this writing.

### use case gICS

the YAML file contains 'all-in-one' including all settings - there is no separate file with environment variables to be considers - it is all included.

It also contains a so-called 'initcontainer' which runs only once. The purpose of this special container is to prepare all files and folders which
gICS expects to find during boot of the container (which is usually done in docker-compose in the original setup). 
Furthermore, it downloads the ZIP File from the ths-greifswald.de website and stores a copy in case of restart to avoid another (unnecessary) download.
In case gICS is already installed it will be moved into a backup-folder with timestamp - and these copies should be deleted every now and then.

**prepare**
Note: The YAML file cannot execute shell commands on the host - so in order to create the necessary folder structure under 

`/var/disk1/containers` please find the commands to prepare the host environment in the first lines as comment in the YAML file

in this case:

```
# prepare:
#  mkdir -p /var/disk1/containers/gics010/data/{initfiles,dbconf,dbdata,applogs,appadins,appcacerts,workdir}
```

copy the `mkdir` line and execute in shell - usually no 'sudo' is required unless explicitly stated

**test run**

After all folders are created on the host we can start creating the pod by

`podman play kube ~/.config/gicsonly.yaml`

then watch the logfiles

`podman logs gics-mysql`

for the database and

`podman logs gics-wildfly`

for the application

If anything goes wrong check the logfile messages to identify the root cause.

*After* the logfiles had been evaluated and all issues fixed - or if everything went just well - you can stop the pod for proper cleanup by

`podman play kube ~/.config/gicsonly.yaml --down`

**autostart at boot**

to start 'gics' automatically at boot time enter the following command


`systemctl --user enable  pod-gics.service --now`

(yes, believe it or not - also users can add systemd services)

the 'magic' behind is a file at

`~/.config/systemd/user/pod-gics.service`

which you can adapt or finetune to your needs - e.g. ensure it will only start *after* the keycloak service had been started successfully if your giCS workload
is configured (depends on) keycloak authentication - instead of GRAS.


```
[Unit]
Description=Podman pod-gics.service

[Service]
Restart=on-failure
RestartSec=30
Type=simple
RemainAfterExit=yes
TimeoutStartSec=30

ExecStartPre=/usr/bin/podman pod rm -f -i gics
ExecStart=/usr/bin/podman play kube /home/core/.config/gicsonly.yaml

ExecStop=/usr/bin/podman pod stop gics
ExecStopPost=/usr/bin/podman pod rm gics

[Install]
WantedBy=default.target
```

### use case keycloak

Again, the YAML file contains 'all-in-one' including all settings - there is no separate file with environment variables to be considers - it is all included.

There is no initcontainer in this keycloak pod yet - you just get a freshly installed keycloak which only contains the 'master' realm.
The Tts realm needs to be created manually following the instructions provided by Treuhandstelle - because keycloak recently moved thru a lot
of major versions with breaking changes and it was rather unsuccessful to restore a setup created by an earlier version. So auto-prepare an environment
for gICS and other services is still on the TO-DO list.

**prepare**

start with creating the necessary folders:

`mkdir -p /var/disk1/containers/keycloak04/data/{dbdata,dbconf}`

keycloak does not share use any persistent folder but fully relies on the database to provide persistence.

Also check the YAML file for passwords and check all settings - especially the start of keycloak: by default YAML will use "-dev-start"
to start a 'non-production' dev instance. To start a 'production' instance comment "-dev-start" and uncomment "start" followed by a hostname (which is checked!).
In this setup you run keycloak most likely behind a reverse proxy or web application firewall so uncomment "-edge" as well for this case - which is
required for keycloak to handle the web traffic and response correctly. 

**test run**

After all folders are created on the host we can start creating the pod by

`podman play kube ~/.config/keycloak04.yaml`

then watch the logfiles

`podman logs keycloak04-db`

for the database and

`podman logs keycloak04-app`

for the application

If anything goes wrong check the logfile messages to identify the root cause.

*After* the logfiles had been evaluated and all issues fixed - or if everything went just well - you can stop the pod for proper cleanup by

`podman play kube ~/.config/keycloak04.yaml --down`

**autostart at boot**

to start 'keycloak' automatically at boot time enter the following command


`systemctl --user enable  pod-keycloak.service --now`

**open topic**

In order to avoid issues caused by gICS and keycloak starting at the same time and gICS throwing errors being unable to connect to keycloak 
the systemd file of gics should be modified to reflect the dependency to keycloak - and allow systemd to boot in the correct order (TO-DO)


### use case bunkerweb (web application firewall - experimental)

Again, the YAML file contains 'all-in-one' including all settings - there is no separate file with environment variables to be considers - it is all included.

bunkerweb is an open-source web application firewall (WAF) maintained by a French cybersecurity company (as a kind of "business card" to attract interested customers).
It combines a reverse proxy with rule based protection rules (main OWASP) - and takes care of the certificate handling - and can also act as static webserver.
Likely there is already a (commercial) WAF in your organisation for internet facing applications - however bunkerweb might still be useful for demonstration 
purposes and development setups under real TLS certificates - it also helps to develop the rules for the *any* WAF as this is usually a time consuming
trial-and-error effort - especially if logfiles always need to be requested from the IT team.

**fixed IP Address**

If you are lucky, you already have an routable Internet IP Address which you could easily forward to the WAF - in case you don't have such an address
you could follow this plan: book a small (the smallest) linux box at any cloud provider at your choice - usually you can book a fixed IP address 
for this box for a few cents more - there is no requirement to this linux box (e.g. any minimal ubuntu should do) it just has to be a minimum amount of
packages installed for a small attack surface - all we need is the kernel - which already has a wireguard device included - and the wireguard tools for setup.
For reference, as of this writing, hetzner cloud should allow to operate such a frontend for a total of less than 5€/month (server class CX11) - which is acceptable for your own
permanent fixed IP address - whereas AWS and Azure likely will charge around 10€/month minimum for the same. 

**wireguard config**
The goal is to forward the (only the expected and intended) traffic from internet thru this tunnel - find sample files in the "wireguard" folder in this repo.
There are plenty of excellent guides how to generate your own keys and setup the tunnel: 'frontend' will be the machine with Internet access while
'backend' is your machine which contains bunkerweb, keycloak and gICS. The sample files also show to map the traffic from port 80/tcp and 443/tcp to
8080/tcp and 8443/tcp for the ports on the container side - to avoid (unsecure) system modifications for privileged port smaller than 1024.

CoreOS already with all necessary wireguard drivers and components - but, again, due to SELINUX constraints, wireguard in CoreOS (and Fedora and RedHat
Enterprise Linux) must be configured the `nmcli` instead of the classic wireguard tools.
 

**prepare**

start with creating the necessary folders:

```
mkdir -p /var/disk1/containers/bunker006/data/{ngxconf,bwdata}
chmod 770 /var/disk1/containers/bunker006/data/*
sudo chown 524388:524388 /var/disk1/containers/bunker006/data/*
chcon -t container_file_t -R /var/disk1/containers/bunker006/data/
```

As now your own domain names will be used (or at least a free service like nip.io which turns any IP address into a DNS name, e.g. IP address 1.11.233.44 into
1.11.233.44.nip.io and 1-11-233-44.nip.io) please review the YAML carefully - and read the documentation at https://docs.bunkerweb.io/

Note: creating "letsencrypt" (letsencrypt.org) certificates for xyz.nip.io is possible - but it requires some patience and ability/willingness to read logfiles.
Reason: there is a max limit of certificates which can be created per domain - which is very often reached for a generic service like 'nip.io'.
However, if this limit is reached and it refuses to generate a new certificate, the message also contains a UTC timestamp by when the counter is reset
(usually every top of an hour, timestamp is UTC). Then retry shortly after. 

**test run**

After all folders are created on the host we can start creating the pod by

`podman play kube ~/.config/bunker006.yaml`

then watch the logfiles

`podman logs bunkerweb-app`

Especially if the write access to the persistent volume shows any error due to insufficient permissions.

If anything goes wrong check the logfile messages to identify the root cause.

*After* the logfiles had been evaluated and all issues fixed - or if everything went just well - you can stop the pod for proper cleanup by

`podman play kube ~/.config/bunker006.yaml --down`

**autostart at boot**

to start 'bunker' automatically at boot time enter the following command


`systemctl --user enable  pod-bunkerweb.service --now`

**open topic**

provide more detailed instructions including static webserver for easier troubleshooting (TO-DO)



