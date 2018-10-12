# luks-net

Unlock a LUKS volume automatically at boot time, where the LUKS key is
stored on a remote server.  This is really useful for headless server
machines.

# Threat Model

The idea is have a headless server machine with an encrypted root file
system (LUKS) that can only be unlocked if the machine is connected to
a network that provides the correct key.

Imagine having a headless server in your garage and you want the
server to be useless if it were to be stolen and disconnected from
your home network.

# How it Works

The system requires two machines, a key server (holds the LUKS key)
and the client machine with the encrypted LUKS volume.

At boot time (in the initramfs) the client machine uses `scp` to copy
the LUKS key from the key server.  Next, the client machine unlocks
the LUKS volume with the key and the normal client boot continues.

This package is intended for a small number of client machines and is
not really appropriate for large scale deployment.

For scale, consider using network bound disk encryption projects like
[clevis and
tang](https://blog.cloudpassage.com/2017/12/21/network-bound-disk-encryption-red-hat-linux-7/).

# Assumptions

* The primary assumption is that the key server machine is secure.

* It is very handy to have the `dropbear-initramfs` package installed
  for remote initramfs debugging.

# Install and Setup Instructions

This repo contains the source for a Debian package that is installed
on the client machine.

The configuration steps for the key server are not automated at this
time and must be manually applied.

## Build the Debian Package

```
build-host:$ cd luks-net
build-host:$ dpkg-buildpackage -uc -us
build-host:$ ls -l ../*.deb
-rw-r--r-- 1 curt denizen 3192 Oct 12 14:07 ../luks-net_1.0_all.deb
```

## Install the Package on the Client

```
client:$ sudo dpkg -i luks-net_1.0_all.deb
```

`Note:` The client is not complete configured yet.

There is also a configuration file, `/etc/luks-net/config`, that we
will cover in the following sections.

## Create a ssh Key Pair

The system uses ssh to communicate between the client and key server.
Create a special purpose ssh key pair for this service:

```
client:$ sudo ssh-keygen -t rsa -b 2048 -f /etc/luks-net/ssh.key
```

The system expects the private key file to have the name
`/etc/luks-net/ssh.key`.

We will install the public key (ssh.key.pub) on the key server in the
next step.

## Create User on Key Server

Create a user on the key server to service the key request.

```
key-server:$ sudo adduser luks-net
```

The default user name is `luks-net`, but you can specify whatever you
want in the luks-net configuration file on the client.

## Store Public ssh Key in luks-net User Account

Add the public part of the ssh key to the created user's
`authorized_keys` file.  On the key server:

```
key-server:$ sudo cp ssh.key.pub /home/luks-net/.ssh/authorized_keys
```

## Store LUKS Key in luks-net User Account

In this example the LUKS key is a one line text file containing a
simple passphrase.  The default location for this file is under the
luks-net user's home directory on the key server:

```
key-server:$ sudo cp luks-passphrase.txt /home/luks-net/data/<hostname>
```

The path to the LUKS key on the key server is configurable in the
luks-net conf file on the client.

`Note:` The LUKS key is quite sensitive.  Take all appropriate
measures to make sure the key file is read-only for the luks-net user
and that the key server is secure.

## Configure the Client

Edit the luks-net configuration file, setting the parameters as
necessary.  After updating this file or any file in `/etc/luks-net`,
the initramfs for the running kernel must be rebuilt.

Here is the config file with defaults:
```
#
# Configuration options for the luks-net boot scripts.  You must run
# update-initramfs(8) to effect changes to this file.  Likewise for
# other files under the '/etc/luks-net' directory).

# luks-net requires the user to create a public/private ssh key pair
# and put the private key in /etc/luks-net/ssh.key with file
# permissions 0600.

# Remote server holding key
#
# LUKS_NET_REMOTE_SERVER=key-server

# Remote user holding key
#
# LUKS_NET_REMOTE_USER=luks-net

# Path to key on remote server
#
# LUKS_NET_REMOTE_KEY_PATH=data/<hostname>

# Additional command line options to pass to scp(1)
#
# LUKS_NET_SCP_OPTIONS=""
```

After making any changes, update the initramfs:

```
client:$ sudo update-initramfs -u
```

You also need to update the initramfs if you change the ssh key.

# Debuging

Be sure to install and configure `dropbear-initramfs`.  This is super
handy for headless debug.

Things to look at and try:

1. Add `debug` to the kernel command line options.  Among other things
   this instructs the initramfs to log a bunch of info in
   /run/initramfs/ that you can look at after the system has fully
   booted.

1. Add some debug prints and `set -x` to initramfs scripts, in
   particular `/usr/share/initramfs-tools/scripts/luks-net`.
