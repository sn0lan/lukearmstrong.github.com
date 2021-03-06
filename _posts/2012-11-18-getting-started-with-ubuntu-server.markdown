---
layout: post
date: 2012-11-18 13:00
title: "Getting started with Ubuntu Server"
description: "Here are the steps I take when setting up a server. Basic configuration, security considerations and firewall rules."
---


This was written using a [Linode](http://www.linode.com/?r=0b86dec2dc8ba417b9d86cb3b44f1131b92c1b26) running Ubuntu 12.04 LTS.


## Logging in

If you are using Linux, or Mac OSX, then login by simply typing this into a terminal.

    ssh root@XXX.XXX.XXX.XXX

_Note: Where you see XXX.XXX.XXX.XXX, use the IP address of your server._

If you are using Windows, you'll have to download something like [Putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/) to login to your server. I'm not going to explain how to setup Putty, it's fairly straight forward, and there are plenty of tutorials out there if you are stuck.


## Update

First thing you should do is update your server.

    apt-get update && apt-get upgrade -y

_Note: If there are updates to the Linux kernel you will need to reboot, but there is no need to do that yet as we will be rebooting at the end of this tutorial._


## Set Hostname

Decide on a name for your server, I'm going to call mine linode.

    echo "linode" > /etc/hostname
    hostname -F /etc/hostname


You also need to add your hostname, and your FQDN, to your hosts file.

`/etc/hosts`

    XXX.XXX.XXX.XXX	linode	linode.example.com

_Note: Where you see `XXX.XXX.XXX.XXX`, use the IP address of your server, and change `example.com`, to your own domain name._

## Set Timezone

    dpkg-reconfigure tzdata
    
    # Current default time zone: 'Europe/London'
    # Local time is now:      Sun Oct  7 19:37:58 BST 2012.
    # Universal Time is now:  Sun Oct  7 18:37:58 UTC 2012.


## Create User

You don't want to be in the habit of using the root account. By default Ubuntu disables the root account, but [Linode](http://www.linode.com/?r=0b86dec2dc8ba417b9d86cb3b44f1131b92c1b26) re-enables it.

Let's create a user account. [This is similar to useradd](http://www.go2linux.org/useradd-vs-adduser).

    adduser luke


### Add to sudoers

    usermod -a -G sudo luke

Now log out, and try logging in using your new user account.


## Local SSH Config

As you have logged out of your server, you could use this opportunity to make life easier when connecting to it.

Setup your SSH Key, this process will recommend you set a password to use your key, do not leave this blank. It may seem odd to replace an SSH password prompt with an SSH Key passphrase prompt, but you will only be asked to unlock your SSH key once per session, and you can use your key to access as many servers/services as you have sent it to.

    ssh-keygen -t rsa -b 4096


An optional step, which I would recommend, is to create an SSH config file.

`~/.ssh/config`

    Host linode
    HostName linode.example.com
    User luke


Now all we need to do is send our SSH key to the server, it'll prompt you for your password.

    ssh-copy-id linode


So now you should be able to connect to your server by running this command, and you won't have to enter your password.

    ssh linode

_Note: You won't have to enter your server password, but the first time you use your SSH Key in a session you will have to enter your passphrase to unlock it._


## Prevent Root from logging in.

Lock the account.

    sudo passwd -l root

    # -l, --lock
    # Lock the password of the named account. This option disables a password by changing it to a value which matches no possible encrypted value (it adds a ´!´ at the beginning of the password).
    #
    # Note that this does not disable the account. The user may still be able to login using another authentication token (e.g. an SSH key). To disable the account, administrators should use usermod --expiredate 1 (this set the account's expire date to Jan 2, 1970).
    #
    # Users with a locked password are not allowed to change their password.


Make the password expire, this prevents anyone from logging in as root using an SSH Key.

    sudo passwd -e root

    # -e, --expire
    # Immediately expire an account's password. This in effect can force a user to change his/her password at the user's next login.


## SSH Settings

These settings prevent root from logging in, and only allow the users you specify to login. It also automatically logs you out if you have been inactive for 5 minutes; If that is annoying, don't use the last 2 configuration lines, or change the last one to be a longer amount of time (in seconds).

`/etc/ssh/sshd_config`

    # Prevent root from logging in
    PermitRootLogin no

    # Only allow these users to log in
    AllowUsers luke

    # Automatically log out a user if they have been inactive for 5 minutes
    ClientAliveCountMax 0
    ClientAliveInterval 300


## Firewall

Now we need to create a file for our Firewall rules, I'm not aware of a recommended place for this, so to make it obvious lets create this file:

`/etc/iptables`

    *filter
    :INPUT ACCEPT [1:52]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    -A INPUT -i lo -j ACCEPT 
    -A INPUT -d 127.0.0.0/8 -i lo -j REJECT --reject-with icmp-port-unreachable 
    -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT 
    -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT 
    -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT 
    -A INPUT -p tcp -s XXX.XXX.XXX.XXX --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
    -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT 
    -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7 
    -A INPUT -j REJECT --reject-with icmp-port-unreachable 
    -A FORWARD -j REJECT --reject-with icmp-port-unreachable 
    -A OUTPUT -j ACCEPT 
    COMMIT


As far as I am aware this config essentially prevents access to all ports apart from 80 (http) and 443 (https). Port 22 (ssh) is restricted to my IP address.

_Note: Change `XXX.XXX.XXX.XXX` to your IP address. You can use this entire line more than once if you want to allow access from more than one IP address._

I have a static IP address so I am able to do this, if you have a dynamic IP address you won't be able to use this rule. You are going to have to figure out another way of securing your SSH port. The simplest way of doing this is to chuck a few quid at your ISP and get a static IP address, if they won't give you one, move to someone who will.

`/etc/iptables` isn't going to be used unless we set it, so we need to set iptables up to enable these rules when the network is connected.

`/etc/network/interfaces`

    # Firewall rules
    pre-up iptables-restore < /etc/iptables


## Hosts Allow/Deny

God only knows what these files do, seemingly nothing if my Deny file is actually being used, but I do this anyway.

`/etc/hosts.allow`

    sshd: XXX.XXX.XXX.XXX


_Note: Change `XXX.XXX.XXX.XXX` to your IP address. You can use this entire line more than once if you want to allow access from more than one IP address._

`/etc/hosts.deny`

    ALL: ALL


## Reboot

It's easiest just to reboot after doing all of this. It means any kernel updates are used, and all the configuration you have setup is used without having to restart a lot of difference services.

    sudo reboot


If you are now unable to login using SSH, it'll probably be to do with your Firewall rules. Luckily, most hosts, including [Linode](http://www.linode.com/?r=0b86dec2dc8ba417b9d86cb3b44f1131b92c1b26), provide another method of accessing the terminal directly.
