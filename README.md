Apache Archiva Cluster with Cassandra
======================================

# About

This project tries to set up an Apache Archiva cluster with 3 nodes. Things should keep running if one (or maybe 2) servers are down.

I am not looking for a Kubernetes solution where a single host is re-created automatically if it is down.

The project is still "work in progress". It is not working yet. Looks like I'm still missing some points...

# The system

The test system uses LXD for virtualization. 3 VMs are created, each one with 3GB of RAM. Real VMs are used instead of Docker to make the migration to real machines as painless as possible.

The 3 VMs are running Ubuntu 22.04. They are managed with Ansible.

Installed on each node is...

* Cassandra
* Archiva
* (to be done) a common filesystem, maybe GlusterFS?

The 3 Cassandra nodes form a cluster for the repositories metadata content storage ( https://archiva.apache.org/docs/2.2.8/adminguide/repositories-content-storage.html ).

The user database will be done with an external LDAP server. This is not part of my test as I carelessly assume that it will work without a problem.

With information from here ( https://cwiki.apache.org/confluence/display/ARCHIVA/High+Availability+Archiva ) I assume there should be a common FS for all nodes. I'm planning to use GlusterFS for this.

# Host setup

This chapter describes the necessary steps to prepare your host system for running the 3 VMs. Please note that each VM has 3GB of RAM, so your host should have 9GB + working space available.

I'm using Ubuntu 20.04 with 16GB of RAM as my host. All instructions are only tested on this system.

## Install LXD

Run in a shell:
```
sudo snap install lxd
```

At the time of writing this installs LXD 5.3 .

Do a quick-and-dirty local LXD initialization:
```
huhn@asteroid:~$ sudo lxd init
[sudo] password for huhn: 
Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Name of the storage backend to use (btrfs, dir, lvm, zfs, ceph) [default=zfs]: dir
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: 
What should the new bridge be called? [default=lxdbr0]: 
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: none
Would you like the LXD server to be available over the network? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: no
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: 
```

Here are the differences to the default configuration:
* The storage pool was set to ```dir```.
* IPv6 address was set to ```none```.
* Automatic update of images is set to ```no```.

*Please note*: these settings result in a "quick-and-dirty" test system. They are definitely not suitable for a production system.

Add the current user to the ```lxd``` group:
```
huhn@asteroid:~$ sudo addgroup huhn lxd
```
*Please note*: you have to re-login to make these changes effective.

## Prepare automatic SSH configuration

The script ```lxd_test/create``` prepares an SSH key for each VM and provides a configuration in ```~/.ssh/config.d/```.

The configuration...
* Provides a short host ID ("powermooh02", "powermooh03" and "powermooh04").
* Sets the IP of the VM.
* Sets username and SSH key.
* Disables strict host key checking and known hosts for the VMs.

Create the folder for the auto-generated SSH configuration:
```
huhn@asteroid:~$ mkdir -p ~/.ssh/config.d/
```

Include the auto-generated SSH configuration by inserting the following line at the top of ```~/.ssh/config```:
```
Include config.d/*
```
*Please note*: The line must be the first one in ```~/.ssh/config``` or it will be ignored.

## Choose IPs

In the chapter "Install LXD" a new bridge was created with the command ```sudo lxd init```. Check the following line for the name of the bridge:
```
What should the new bridge be called? [default=lxdbr0]:
```
Here the bridge has the name ```lxdbr0```.

Now check the IP range of the bridge:
```
huhn@asteroid:~$ lxc network show lxdbr0
config:
  ipv4.address: 10.4.34.1/24
  ipv4.nat: "true"
  ipv6.address: none
description: ""
name: lxdbr0
type: bridge
used_by:
- /1.0/profiles/default
managed: true
status: Created
locations:
- none
```

Here the bridge has the network ```10.4.34.x/24``` assigned with ```10.4.34.1``` reserved for the bridge.

Choose 3 IPs in the network for the 3 VMs. In my example I chose...

* ```10.4.34.2```
* ```10.4.34.3```
* ```10.4.34.4```

Assign the IPs to the 3 VMs in the file ```lxd_test/create```:
```
# This is a list of all hostnames with their IP.
atInstances=(
  'powermooh02 10.4.34.2'
  'powermooh03 10.4.34.3'
  'powermooh04 10.4.34.4'
)
```

## Install Ansible

Install Ansible as described here: https://launchpad.net/~ansible/+archive/ubuntu/ansible

Here are the necessary commands:

```
sudo add-apt-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible
```


# Create the VMs

Change to the ```lxd_test``` folder and run the command ```./create```. It creates a new SSH key for the "powermooh" user on the VMs,
creates and starts the 3 VMs and creates a SSH configuration to connect to all VMs.

After the ```./create``` command completed successfully, it is possible to connect to all 3 VMs:

```
huhn@asteroid:~$ ssh powermooh02
Warning: Permanently added '10.4.34.2' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 22.04 LTS (GNU/Linux 5.15.0-1012-kvm x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


14 updates can be applied immediately.
14 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


Last login: Mon Jul 18 13:12:44 2022 from 10.4.34.1
powermooh@powermooh02:~$ 
```

*Please note*: Despite the warning ("Permanently added '10.4.34.2' (ECDSA) to the list of known hosts.") nothing is added to the list of known hosts. This is disabled in ```~/.ssh/config.d/powermooh```.

# Run the Ansible playbook

Run the playbook with the following command:

```
ansible-playbook artifact-manager.yml
```

*Please note*: The "play recap" mentions 1 skipped step for ```powermooh03``` and ```powermooh04```:
```
PLAY RECAP **********************************************************************************************************************
powermooh02                : ok=40   changed=36   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
powermooh03                : ok=39   changed=35   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
powermooh04                : ok=39   changed=35   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
```

This is the task "Create the keyspace.". It creates the Cassandra keyspace. As Cassandra is set up as a cluster this has to be done only on one of the VMs. For the other 2 VMs the task is skipped.

# And now?

At the moment Archiva does not start yet. Take a look at ```/var/lib/archiva/logs/```, it complains with the Exception "InvalidRequestException(why:unconfigured table metadatafacet)".

