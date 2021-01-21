# Setting up SSH and Remote Mounting

Inspired by a presentation from Demitri Muna.



## Overview

A secure shell (SSH) is a network protocol that provides an easy but secure way to access a remote computer. Many computers, including many Macs and Linux machines, come with SSH pre-installed. In general, Windows machines do not. This document will walk you through configuring SSH on your machine in a way that is convenient, and easy to use. When done you will be able  connect to your configured servers just by typing (with autocomplete!) the following:

```bash
% ssh my_server_nickname
```

For convenience (and added security), we will discuss the use of authentication keys, autocomplete, and remote file mounting. Note that all code snippets in this document will assume you are starting from your home directory.

**Note:** In order to connect to a University of Pittsburgh server over SSH you will need to install and use the University's VPN service. For more information see [technology.pitt.edu](http://technology.pitt.edu/services/pitt-vpn-secure-remote-access). You will also need to have a user account on whatever server you are trying to connect to. Both of requirements should be satisfied before moving forward.

## SSH Config File

The SSH configuration file is used to store settings for an SSH connection initiated by your computer. It allows you to set settings for an individual server, or to set them globally for all connections. To begin, we create the file `~/.ssh/config`. It's very important we create this file to have restricted permissions so that it is only read/writable by the user: 

```bash
% mkdir .ssh
% chmod 700 .ssh
% cd .ssh
% touch config
% chmod 600 config

```

Once you have created this file, open it in your favorite text editor (bbedit anyone!?) and enter in the following information. Note that I'm using information for a made up server called zeus. You will need to change the below example to use the address and login information of your particular server.

```
# Settings for all servers
Host *
     ControlMaster auto
     ControlPath ~/.ssh/connections/%r@%h:%p
     ControlPersist 1m
     ServerAliveInterval 90
     ServerAliveCountMax 10

# Settings for a specific server
Host zeus
    HostName ZEUS.IMAGINARY.UNIVERSITY.EDU
    User mattsmith
    Port 22

```



#### But wait, what do all those arguments mean!?

Since setting up an SSH connection requires some overhead, SSH has the ability to use "multiplexing". This means SSH can send more than one signal over a single connection. The first three arguments configure this ability. The rest are for added convenience.

- The `ControlMaster` argument determines how SSH will handle control connections. Unless you have a specific application in mind, its best to leave this on `auto`. 
- The `ControlPath` specifies where you computer will store information about each connection. This allows the connection to be reused later on. 
- The `controlPersist` argument is how long to keep an SSH connection open after you close your terminal window. Here we have set it to one minute. This means that if you close out your terminal, but then decide you need to reconnect to the server (less than a minute later), you won't have to start a whole new connection to do it. This argument also makes it so that the connection of one terminal window doesn't depend on another.

- By setting the `ServerAliveInterval` argument to 90, our computer will send a message to the remote server every 90 seconds so that we don't get logged for being idle.
- `ServerAliveCountMax` Specifies how many times our computer should try to reconnect to the remote server if we lose our connection.



## Setting up autocomplete

We can setup the `ssh` command to autocomplete the server data stored in `~/.ssh/config`. To do so, create a new file in your home directory called `~/.autocomplete.sh`.

```bash
% touch .autocomplete.sh
```

In this file, paste the following function: 

```bash
# SSH
function _ssh_completion() {
    egrep -o '^Host [a-zA-Z]+' $HOME/.ssh/config | awk '{ print $2 }'
}
complete -W "$(_ssh_completion)" ssh
```

Then In your `~/.bashrc` (linux) or `~/.bash_profile` (macOS) file, add this line:

```bash
source $HOME/.autocomplete.sh 
```



#### Trouble Shooting

If you have completed the above and the autocomplete feature try the following steps:

1. Quit and resetart your command line interface. This will load the changes you made to `.bashrc`  or `.bash_profile`.

2. Make sure you have the `complete` command installed by running 

   ```bash
   brew install bash-completion
   ```

   

## SSH Keys

SSH keys provide an easy, secure way of logging into a server. Authentication keys are generated as a pair of public and private keys. When you connect to a server that has your public key, it will ask you to provide the matching private key. As the name implies, anyone is allowed to have your public key, but your private key should be stored securely.

We will be generating keys using the RSA algorithm with 4096 bits. At present, there is no known way to break an encryption with an  4096-bit key. To keep things organized, we will be saving our keys to a file called `zeus_rsa_4096`.

```bash
% cd .ssh
% ssh-keygen -t rsa -b 4096
% Generating public/private rsa key pair.
Enter file in which to save the key (/Users/mattsmith/.ssh/id_rsa): zeus_rsa_4096
Enter passphrase (empty for no passphrase):
Enter same passphrase again:

```

This will generate and write a private and public key to the files `zeus_rsa_4096` and `zeus_rsa_4096.pub` . If you entered a passphrase, you will need to type that password every time you use the ssh keys. Since a 4096 RSA key is unbreakable by brute force, we skip that here.



To  setup authentication on a server using SSH keys, you need to do two things. First you will need to put the public key on the server you wish to connect to. On the remote server, create the file `~/.ssh/authorized_keys` and copy the public key from ``zeus_rsa_4096.pub` onto a single line. This file also require that we set the proper permissions.

```bash
% mkdir .ssh
% chmod 700 .ssh
% cd .ssh
% touch authorized_keys
% chmod 700 authorized_keys
```



Next you need to add the private key information to your SSH config file as follows:

```
Host zeus
     HostName ZEUS.IMAGINARY.UNIVERSITY.EDU
     User mattsmith
     IdentityFile ~/.ssh/zeus_rsa_4096
     IdentitiesOnly yes
```

The `IdentityFile` argument tells SSH where to look for your SSH key. If this value is not set, SSH will try every private key it can find. This is clearly a security issue, since you don't want to be sending all of your **private** keys to every server you meet. The  `IdentitiesOnly` argument here tells SSH to skip asking for a password.



## Remote Mounting

Interacting with a server over the command line is great, but sometimes its useful to explore the file system graphically. Furthermore, mounting a remote file system to your own allows you to access that file system using locally available tools and software that may not be built to work with SSH. We will be setting up the ability to remote mount a file system using SSHFS.



### For Mac

In the following example, we setup SSHFS to mount remote file systems to the local directory `/Volumes/`. This means if you were to mount the server `zeus`, it would be located at `/Volumes/zeus/`.



First install FUSE for macOS & SSHFS from [http://osxfuse.github.io](http://osxfuse.github.io). **Make sure to have the  “compatibility layer” option checked.** Then add the following functions to your `.bash_profile` file:

```bash
source $HOME/.autocomplete.sh
remotemount () {
    umount ~/Volumes/$1 >/dev/null 2>&1
    if ! [ -d ~/Volumes/$1 ]
    then
        mkdir ~/Volumes/$1
    fi
    sshfs -o volname=$1 -o local $1:$2 ~/Volumes/$1
}

uremotemount () { umount ~/Volumes/$1; }
```

You can now use the commands `remotemount zeus` and `uremotemout zeus` to mount and unmount the zeus server we configured earlier, and it works with autocomplete! By default the `remotemout` command will mount the remote server from your user directory on that machine. You can change this by running

```bash
remotemout zeus \some\other\dir
```



We can setup the `remotemount` and `uremotemount` commands to autocomplete just like the `ssh` command. To do so, we edit the `~/.autocomplete.sh` file so that it looks like the following

```bash
# SSH
function _ssh_completion() {
    egrep -o '^Host [a-zA-Z]+' $HOME/.ssh/config | awk '{ print $2 }'
}
complete -W "$(_ssh_completion)" ssh
complete -W "$(_ssh_completion)" remotemount
complete -W "$(_ssh_completion)" uremotemount
```



### For Linux

First you will need to install fuse and sshfs.

```bash
% sudo apt-get install fuse
% sudp 
% sudo apt-get install sshfs
% sudo adduser <your username> fuse 
```

For the install to complete, log out and then log back in. Then create the directory where you want to mount your remote connections. Here we use the directory `/mnt/sshfs`.

```bash
% sudo mkdir -p /mnt/sshfs
% sudo chown <your username>:fuse /mnt/sshfs 
```

If your account name on the remote server `ZEUS.IMAGINARY.UNIVERSITY.EDU` is `mattsmith`, you can mount your home directory on Zeus to your local home directory using

```
% sudo sshfs mattsmith@ZEUS.IMAGINARY.UNIVERSITY.EDU: /mnt/sshfs/zeus
```

To mount a different remote directory, run

```bash
% sudo sshfs mattsmith@ZEUS.IMAGINARY.UNIVERSITY.EDU:/some/other/dir /mnt/sshfs/zeus
```

To unmount the remote drive use the `fusermount` command.

```bash
% sudo fusermount -u /mnt/sshfs/zeus  
```

You can also create aliases to make this simpler for commonly used hosts.

```
# zeus
alias mount_astro="sudo sshfs mattsmith@ZEUS.IMAGINARY.UNIVERSITY.EDU: /mnt/sshfs/zeus"
alias unmount_astro="sudo fusermount -u /mnt/sshfs/zeus  "
```


