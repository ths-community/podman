# podman to run gICS

gICS - as all tools provided by "(unabhängige) Treuhandstelle Greifswald" (https://www.ths-greifswald.de/en/researchers-general-public/gics/) comes in a 
containerized format. However - as of this writing - all scripts and documentation refer to docker as runtime - which is OK for workstations
but only a 2nd best choice when it comes to servers, 24/7 operations, small resource footprint and attack service and disaster recovery.

The purpose of this article is to demonstrate how gICS can be operated in a more server-like environment optimized for better security and 
low resource footprint ("costs") for 24/7 operations

## disclaimer

As with all community contributions this is neither a fully-support product nor in any way assiciated with my employer or any other company.
It may contain bugs and may lack handling of certain situations and incidents. It may have been reduced and simplified for clarity to understand
the concepts rather than provide a bullet-proof solution.


## what is podman? 

There are much better and more comprehensive articles out there which explain who has initiated podman as a docker replacement and why -
for the purpose of this community contribution I would only highlight to the common ground that both docker and podman both use container runtime
libraries based on the OCI standard - this provides a huge level of compatibilities between both options.

however, the podman addresses a few by-design issues from docker like

- need of a root-privilegded service
- potentially insecure communication between containers by docker.sock

which is why especially Enterprise Linux distros like RedHat (incl Fedora) and SUSE decided to stop support for docker in favor of podman.

However, podman is available on all major Linux distros (caution it may come in *outdated* versions when installed by the *standard* package manager)
as well as docker (CE = community Edition) can still be installed on Enterprise Linux but without vendor backup when it comes to support

Frankly speaking many concepts for podman are rather a backport from kubernetes paired with the simplicity of docker

please refer to  

https://indico.cern.ch/event/757415/contributions/3421994/attachments/1855302/3047064/Podman_Rootless_Containers.pdf

for instance as one out of many good articles regarding this topic - for more detailed docker/podman comparision and reasoning

## fedora CoreOS

The easiest way to install podman is to install a host operating system in which it is already included by default - like CoreOS (sometimes also referred as
FCOS = Fedora Core OS). As the name indicates it is now a variant of Fedora to provide a secure minimal Linux to run container workloads (with podman by default) -
CoreOS is also used by RedHat Openshift - https://www.redhat.com/en/technologies/cloud-computing/openshift/what-was-coreos

Key features are a minimized Linux with current kernel and fixes and hence low attack surface. After installation most parts are readonly and updates are
applied to a layered filesystem (like in containers itself) - this makes rollback easy in case of issues and ensures integrity of (most of the) code.
And built-in updater install updates automatically usually once or twice a month.

You will probably miss a graphically user interface or a package manager which allows to install more packages - both are against the basic concepts.
E.g. if a python3 runtime is needed then a container containing one is spin up to do the processing - instead of installing it into the OS.  
Even for debugging in CoreOS itself it uses a special debug container. It is cleaner - and easier to get rid of things which are no longer needed.

### CoreOS download 



go to 

https://fedoraproject.org/

and scroll down on that page - Fedora CoreOS should appear in the list open various options and flavours - rather at the end

click "download now" and pay attention that the version is "stable" and your architecture is likely "x86_64" and "bare metal" - "Live DVD ISO"
is the correct version if you are operating on own hardware - and not in any of the cloud environments.

please remember where we downloaded this ISO file



### CoreOS - prepare login

"security first" requires that login by default is key-based - not username/password (which is considered vastly insecure).
There is an option for username/password for the local console only - which is out of scope here

so the first thing to do is create a public/private keypair 

and then add the public key to a so-called ignition file which is used to automatically configure Fedora CoreOS during install.
Ignition file can be used to apply many different configuration setting - but we will only focus here for the login key as the bare minimum.

`ssh-keygen coreos`

which creates two files

```
coreos
coreos.pub

```


then the *public* key must be added to the ignition file (copy the full file content in [ ] at "sshAuthorizedKeys" )

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
          "ssh-rsa AAAAB3NzaC1y...er7Q7/7 coreos"
        ]
      }
    ]
  }
}

```

finally put this file on any webserver (if you don't have one ... you may want to create a github account and put in a new repo)

Note: You may feel uncomfortable to put a public key to a public webserver - but this is why public keys are made for.
However, there is a (very small) risk that if a hacker comes in possession ob private(!) keys they may use those to test which public key will fit
and then use this knowledge to made it part of an attack plan.

### CoreOS - install

CoreOS is made for *server* use-cases - here is a word of warning:

CAUTION - the installtion procedure will always grab the full harddisk and overwrite existing content without any further warning.
It is *NOT* made for co-existence with other operating systems like Windows, Linux oder MacOS which you know from desktop setups.

There are two easily way to deal with this "special feature"

* Use a type1 hypervisor like Proxmox, Hyper-V oder ESXi - or at least VirtualBox or VMware workstation - to provide a virtual machine with seperated virtual disks
* Use a seperate hardware PC or notebook which does NOT contain anything (any more) that is still needed

In a hypervisior / virtual target machine you can configure to use the ISO as virtual CD-ROM -

In case of a real hardware (called: bare metal) you need to burn the ISO file onto a DVD (not recommended) or
make a raw copy to an USB device (recommended) - to make such a bootable USB stick use a tool like "rufus" under windows - or in Linux it is

`dd bs=100M if=filename.iso of=/dev/sdX`

where `/dev/sdX` is usually a device like `/dev/sde` - you can insert the USB stick into the machine and call `sudo dmesg` and at the end of the log you 
can easily see which device was last inserted into the system
 



## install podman on a different Host OS 

your host OS may have already package to install "podman" easily - but note that these packages may contain older version -
"podman" is actively delevoped and new features and versions are issued frequently.
Please go to https://podman.io or use your favorite search engine to find recent installation packages.

