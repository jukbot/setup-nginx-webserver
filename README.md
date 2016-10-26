# Setup and Secure CentOS 7 & Redhat 7

## Introduction

The new CentOS 7 server has to be customized before it can be put into use as a production system. In this article, will help you to increase the security and usability of your server and will give you a solid foundation for subsequent actions.

## Prerequisites

A newly activated CentOS 7 server, preferably setup with SSH keys. Log into the server as root with your server-ip-address.

```
ssh -l root xx.xx.xxx.xxx
```
or
```
ssh root@SERVER_IP_ADDRESS
```

Complete the login process by accepting the warning about host authenticity, if it appears, then providing your root authentication (password or private key). If it is your first time logging into the server, with a password, you will also be prompted to change the root password.

The root user is the administrative user in a Linux environment that has very broad privileges. Because of the heightened privileges of the root account, you are actually discouraged from using it on a regular basis. This is because part of the power inherent with the root account is the ability to make very destructive changes, even by accident.

The next step is to set up an alternative user account with a reduced scope of influence for day-to-day work. We'll teach you how to gain increased privileges during the times when you need them.


## Setup and Initial

### Step 1 — Create a New User

For security reasons, it is not advisable to be performing daily computing tasks using the root account. Instead, it is recommended to create a standard user account that will be using sudo to gain administrative privileges. For this tutorial, assume that we're creating a user named joe. To create the user account, type:

```
adduser example
```
Set a password for the new user. You'll be prompted to input and confirm a password.

```
passwd example
```
Add the new user to the wheel group so that it can assume root privileges using sudo.

```
gpasswd -a example wheel
```

Finally, open another terminal on your local machine and use the following command to add your SSH key to the new user's home directory on the remote server. You will be prompted to authenticate before the SSH key is installed.

```
ssh-copy-id example@server-ip-address
```
After the key has been installed, log into the server using the new user account.

```
ssh -l example server-ip-address
```

If the login is successful, you may close the other terminal. From now on, all commands will be preceded with sudo.


### Step 2 — Disallow Root Login and Password Authentication

Since you can now log in as a standard user using SSH keys, a good security practice is to configure SSH so that the root login and password authentication are both disallowed. Both settings have to be configured in the SSH daemon's configuration file. So, open it using nano.

```
sudo nano /etc/ssh/sshd_config
```
Look for the PermitRootLogin line, uncomment it and set the value to no.
```
PermitRootLogin     no
```
Do the same for the PasswordAuthentication line, which should be uncommented already:
```
PasswordAuthentication      no
```
Save and close the file. To apply the new settings, reload SSH.
```
sudo systemctl reload sshd
```

### Step 3 — Configure the Time Zone

By default, the time on the server is given in UTC. It is best to configure it to show the local time zone. To accomplish that, locate the zone file of your country/geographical area in the /usr/share/zoneinfo directory and create a symbolic link from it to the /etc/localtime directory. For example, if you're in the eastern part of the US, you'll create the symbolic link using:
```
sudo ln -sf /usr/share/zoneinfo/US/Eastern /etc/localtime
```
Afterwards, verify that the time is now given in localtime by running the date command. The output should be similar to:

Tue Jun 16 15:35:34 EDT 2015
The EDT in the output confirms that it's localtime.

#### How do I see the current time zone?

Type the date command or the ls command:
```
date
ls -l /etc/localtime
```

To find list of all available time zones, run:

```
timedatectl list-timezones
```
Grep possible Asian timezones 
```
timedatectl list-timezones | grep Asia
```
Step 4: Enable the IPTables Firewall

By default, the active firewall application on a newly activated CentOS 7 server is FirewallD. Though it is a good replacement for IPTables, many security applications still do not have support for it. So if you'll be using any of those applications, like OSSEC HIDS, it's best to disable/uninstall FirewallD.

Let's start by disabling/uninstalling FirewallD:

sudo yum remove -y firewalld
Now, let's install/activate IPTables.

sudo yum install -y iptables-services
sudo systemctl start iptables
Configure IPTables to start automatically at boot time.

sudo systemctl enable iptables
IPTables on CentOS 7 comes with a default set of rules, which you can view with the following command.

sudo iptables -L -n
The output will resemble:

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
You can see that one of those rules allows SSH traffic, so your SSH session is safe.

Because those rules are runtime rules and will be lost on reboot, it's best to save them to a file using:

sudo /usr/libexec/iptables/iptables.init save
That command will save the rules to the /etc/sysconfig/iptables file. You can edit the rules anytime by changing this file with your favorite text editor.

Step 5: Allow Additional Traffic Through the Firewall

Since you'll most likely be going to use your new server to host some websites at some point, you'll have to add new rules to the firewall to allow HTTP and HTTPS traffic. To accomplish that, open the IPTables file:

sudo nano /etc/sysconfig/iptables
Just after or before the SSH rule, add the rules for HTTP (port 80) and HTTPS (port 443) traffic, so that that portion of the file appears as shown in the code block below.

-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
Save and close the file, then reload IPTables.

sudo systemctl reload iptables
With the above step completed, your CentOS 7 server should now be reasonably secure and be ready for use in production.



## Update and Upgrade

To update installed packages
```
sudo yum -y update 
```
To upgrade installed packages
```
sudo yum -y upgrade
```

## Install Common Packages

### Extra Packages for Enterprise Linux (EPEL)

How do I install the extra repositories such as Fedora EPEL repo on a Red Hat Enterprise Linux server version 7.x or CentOS Linux server version 7.x?

You can easily install various packages by configuring a CentOS 7.x or RHEL 7.x system to use Fedora EPEL repos and third party packages. Please note that these packages are not officially supported by either CentOS or Red Hat, but provides many popular packages and apps.

#### How to install RHEL EPEL repository on Centos 7.x or RHEL 7.x

The following instructions assumes that you are running command as root user on a CentOS/RHEL 7.x system and want to use use Fedora Epel repos.

#### Method 1: Install EPEL repositories from Base Linux repository (recommended)

Install Epel repo using the following command:
```
sudo yum -y install epel-release
```
Refresh repo by typing the following commad: 
```
yum repolist
```

#### Method 2: Install EPEL repositories from dl.fedoraproject.org 

The command is as follows to download epel release for CentOS and RHEL 7.x using wget command:
```
cd /tmp
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo rpm -Uvh epel-release-latest-7*.rpm
```
Refresh repo by typing the following commad: 
```
yum repolist
```

#### Search and install package

To list all available packages under a repo called epel
```
sudo yum --disablerepo="*" --enablerepo="epel" list available
```

Example: Search and install htop package from epel repo on a CentOS/RHEL 7.x

## search it ##
```
sudo yum search htop
sudo yum info htop
sudo yum install htop
```

And, there you have it, a larger number of packages to install from EPEL repo on a CentOS and 
Red Hat Enterprise Linux (RHEL) version 7.x.


### Nginx from source (with ALPN support)


### PHP 7 

If you have installed older version you must remove it first by following command.

```
sudo yum remove php-fpm php-cli php-common
```

Install the new PHP 7 packages from IUS. Again, press y and Enter when prompted.

```
sudo yum install php70u-fpm-nginx php70u-cli php70u-mysqlnd
```

### Phalcon Framework

### Build tool

```
sudo yum -y install autoconf automake bind-utils gcc libtool make nmap-netcat ntp pam-devel unzip wget
```

```
sudo yum -y group install "Development Tools" 
```



