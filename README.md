# Setup a Secure Nginx Web Server on CentOS/Redhat 7.x
(Updated June 9, 2018)

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/Centos-logo-light.svg" alt="PHP7"/>
</p>

I've gather informations and created this article for sysadmin who're using CentOS or Enterprise Linux.
Because most of these linux branches have a long term support, f_cking stable and secured (SELinux). 
However, most libraries and applications that preinstalled are obsolete. 
So this article will guide you STEP BY STEP to build a perfect web server with understanding.

**ANNOUCEMENT**
```
CentOS/RedHat 7.4 the openssl package has been updated to upstream version 1.0.2k, which provides a number of enhancements, new features, and bug fixes, including:
- Added support for the Datagram Transport Layer Security TLS (DTLS) protocol version 1.2.
- Added support for the automatic elliptic curve selection for the ECDHE key exchange in TLS.
- Added support for the Application-Layer Protocol Negotiation (ALPN). -- YESSSSSSS!!!
- Added Cryptographic Message Syntax (CMS) support for the following schemes: RSA-PSS, RSA-OAEP, ECDH, and X9.42 DH.
```

**THIS CONFIG BE USE IN ANYWHERE**
- You can use this config in anytypes of environment eg. Nginx on VPS, VCM, any container or Docker. Just optimize to meet your load.

## Table of Contents  
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Setup and Initial Server](#setup-and-initial-server)
- [Update and Upgrade](#update-and-upgrade)
- [Install Common Packages](#install-common-packages)
  * [1 Yum Utils](#1-yum-utils)
  * [2 Extra Packages for Enterprise Linux (EPEL)](#2-extra-packages-for-enterprise-linux)
  * [3 OpenSSL (with ALPN support)](#3-openssl)
  * [4 Nginx](#4-nginx)
    + [METHOD 1: Prebuilt binary package](#method-1--prebuilt-binary-package)
    + [METHOD 2: Compile from source (If you want to install custom nginx modules)](#method-2--compile-from-source--if-you-want-to-install-custom-nginx-modules-)


## Introduction

The new CentOS 7 server has to be customized before it can be put into use as a production system. In this article, will help you to increase the security and usability of your server with the lastest stable packages and will give you a solid foundation for subsequent actions.

This article **compatible with RedHat Enterprise Linux 7.x**, except some repository links need to change upon vendors and processor architecture of your hardware.


## Prerequisites

A newly activated CentOS 7 server, preferably setup with SSH keys. Log into the server as root with your server-ip-address.

```
ssh -l root xx.xx.xxx.xxx
```
or
```
ssh root@SERVER_IP_ADDRESS
```

To view current ip of server in CentOS
```
# For minimal install
ip addr
```
or
```
ifconfig
```

Complete the login process by accepting the warning about host authenticity, if it appears, then providing your root authentication (password or private key). If it is your first time logging into the server, with a password, you will also be prompted to change the root password.

The root user is the administrative user in a Linux environment that has very broad privileges. Because of the heightened privileges of the root account, you are actually discouraged from using it on a regular basis. This is because part of the power inherent with the root account is the ability to make very destructive changes, even by accident.

The next step is to set up an alternative user account with a reduced scope of influence for day-to-day work. We'll teach you how to gain increased privileges during the times when you need them.


## Setup and Initial Server

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
sudo vi /etc/ssh/sshd_config
```
To exit vim editor press ESC then type :x to save and exit or :q! to ignore and exit 

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

### Step 3 - Configure the Timezones and Network Time Protocol Synchronization

The next step is to adjust the localization settings for your server and configure the Network Time Protocol (NTP) synchronization.

The first step will ensure that your server is operating under the correct time zone. The second step will configure your system to synchronize its system clock to the standard time maintained by a global network of NTP servers. This will help prevent some inconsistent behavior that can arise from out-of-sync clocks.

#### 3.1 Configure Timezones

Our first step is to set our server's timezone. This is a very simple procedure that can be accomplished using the timedatectl command:

First, take a look at the available timezones by typing:

```
sudo timedatectl list-timezones
```
Grep possible Asian timezones 
```
sudo timedatectl list-timezones | grep Asia
```

This will give you a list of the timezones available for your server. When you find the region/timezone setting that is correct for your server, set it by typing:
```
sudo timedatectl set-timezone region/timezone
```

For instance, to set it to United States eastern time, you can type:
```
sudo timedatectl set-timezone America/New_York
```
Your system will be updated to use the selected timezone. You can confirm this by typing:
```
sudo timedatectl
```

#### 3.2 Configure NTP Synchronization

Now that you have your timezone set, we should configure NTP. This will allow your computer to stay in sync with other servers, leading to more predictability in operations that rely on having the correct time.

For NTP synchronization, we will use a service called ntp, which we can install from CentOS's default repositories:
```
sudo yum install ntp
```
Next, you need to start the service for this session. We will also enable the service so that it is automatically started each time the server boots:
```
sudo systemctl start ntpd
sudo systemctl enable ntpd
```
Your server will now automatically correct its system clock to align with the global servers.

#### 3.3 How do I see the current time zone?

Type the date command or the ls command:
```
date
ls -l /etc/localtime
```

### Step 4 - Set Up FirewallD (Optional)

After setting up the bare minimum configuration for a new server, there are some additional steps that are highly recommended in most cases.

#### Prerequisites

In this guide, we will be focusing on configuring some optional but recommended components. This will involve setting our system up with a firewall and a swap file.

#### Configuring a Basic Firewall

Firewalls provide a basic level of security for your server. These applications are responsible for denying traffic to every port on your server with exceptions for ports/services you have approved. CentOS ships with a firewall called firewalld. A tool called firewall-cmd can be used to configure your firewall policies. Our basic strategy will be to lock down everything that we do not have a good reason to keep open.

The firewalld service has the ability to make modifications without dropping current connections, so we can turn it on before creating our exceptions:
```
sudo systemctl start firewalld
```
Now that the service is up and running, we can use the firewall-cmd utility to get and set policy information for the firewall. The firewalld application uses the concept of "zones" to label the trustworthiness of the other hosts on a network. This labelling gives us the ability to assign different rules depending on how much we trust a network.

In this guide, we will only be adjusting the policies for the default zone. When we reload our firewall, this will be the zone applied to our interfaces. We should start by adding exceptions to our firewall for approved services. The most essential of these is SSH, since we need to retain remote administrative access to the server.

You can enable the service by name (eg ssh-daemon) (MUST ENABLE) by typing:
```
sudo firewall-cmd --permanent --add-service=ssh
```
or disable it by typing:
```
sudo firewall-cmd --permanent --remove-service=ssh
```
or enable custom port by typing:
```
sudo firewall-cmd --permanent --add-port=4200/tcp
```

This is the bare minimum needed to retain administrative access to the server. If you plan on running additional services, you need to open the firewall for those as well.

If you plan to run a web server with SSL/TLS enabled, you should allow traffic for https as well:

```
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

```
If you need SMTP email enabled, you can type:
```
sudo firewall-cmd --permanent --add-service=smtp
```
To see any additional services that you can enable by name, type:
```
sudo firewall-cmd --get-services
```
When you are finished, you can see the list of the exceptions that will be implemented by typing:
```
sudo firewall-cmd --permanent --list-all
```
When you are ready to implement the changes, reload the firewall:
```
sudo firewall-cmd --reload
```
If, after testing, everything works as expected, you should make sure the firewall will be started at boot:
```
sudo systemctl enable firewalld
```

Remember that you will have to explicitly open the firewall (with services or ports) for any additional services that you may configure later.


### Step 5 - Enable the IPTables Firewall (Optional for advanced)

By default, the active firewall application on a newly activated CentOS 7 server is FirewallD. Though it is a good replacement for IPTables, many security applications still do not have support for it. So if you'll be using any of those applications, like OSSEC HIDS, it's best to disable/uninstall FirewallD.

5.1 Let's start by disabling/uninstalling FirewallD:
```
sudo yum remove -y firewalld
```

5.2 Now, let's install/activate IPTables.
```
sudo yum install -y iptables-services
sudo systemctl start iptables
```

5.3 Configure IPTables to start automatically at boot time.
```
sudo systemctl enable iptables
```

IPTables on CentOS 7 comes with a default set of rules, which you can view with the following command.
```
sudo iptables -L -n
```
The output will resemble:
```
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
```

You can see that one of those rules allows SSH traffic, so your SSH session is safe.
Because those rules are runtime rules and will be lost on reboot, it's best to save them to a file using:

```
sudo /usr/libexec/iptables/iptables.init save
```

That command will save the rules to the /etc/sysconfig/iptables file. You can edit the rules anytime by changing this file with your favorite text editor.

#### Allow Additional Traffic Through the Firewall

Since you'll most likely be going to use your new server to host some websites at some point, you'll have to add new rules to the firewall to allow HTTP and HTTPS traffic. 

5.4 To accomplish that, open the IPTables file:
```
sudo nano /etc/sysconfig/iptables
```

5.5 Just after or before the SSH rule, add the rules for HTTP (port 80) and HTTPS (port 443) traffic, so that that portion of the file appears as shown in the code block below.
```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
```

5.6 Save and close the file, then reload IPTables.
```
sudo systemctl reload iptables
```

With the above step completed, your server should now be reasonably secure and be ready for use in production.


### Step 6 - Optimized system configuration for maximum concurrency.

In default centos server is not fully optimized to make full use of available hardware. 
This means it might fail under high load. So we need to config the sysctl.conf file for optimization.

1. Open sysctl.conf file
```
sudo vi /etc/sysctl.conf
```

2. Edit file as following configuration
```Ini
# Increase number of incoming connections
net.core.somaxconn = 250000

# Increase the maximum amount of option memory buffers
net.core.optmem_max = 25165824

# Increase Linux auto tuning TCP buffer limits
# min, default, and max number of bytes to use
# set max to at least 4MB, or higher if you use very high BDP paths
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 65536

# Decrease the time default value for tcp_fin_timeout connection
net.ipv4.tcp_fin_timeout = 15

# Number of times SYNACKs for passive TCP connection.
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2

# Decrease the time default value for connections to keep alive
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# Enable timestamps as defined in RFC1323:
net.ipv4.tcp_timestamps = 1

# Allowed local port range
net.ipv4.ip_local_port_range = 2000 65000

# Turn on window scaling which can enlarge the transfer window:
net.ipv4.tcp_window_scaling = 1

# Limit syn backlog to prevent overflow
net.ipv4.tcp_max_syn_backlog = 30000

# Avoid a smurf attack
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Turn on protection for bad icmp error messages
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Turn on syncookies for SYN flood attack protection
net.ipv4.tcp_syncookies = 1

# Protect Against TCP Time-Wait
net.ipv4.tcp_rfc1337 = 1

# Turn on and log spoofed, source routed, and redirect packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# No source routed packets here
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Turn on reverse path filtering
net.ipv4.conf.all.rp_filter = 1

# Make sure no one can alter the routing tables
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# Don't act as a router
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Turn on execshild protection
kernel.randomize_va_space = 2

# Optimization for port usefor LBs Increase system file descriptor limit
fs.file-max = 65535

# Allow for more PIDs (to reduce rollover problems); may break some programs 32768
kernel.pid_max = 65536

# Increase TCP max buffer size setable using setsockopt()
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216

# Increase the tcp-time-wait buckets pool size to prevent simple DOS attacks
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1

# !! Disable ipv6 for security (!! Optional if you're using ipv6 !!)
net.ipv6.conf.all.disable_ipv6= 1
```


3. Reload the systemctl file
```
sysctl -p
```

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

### 1 Yum Utils 

**Yum-utils** is a collection of utilities and plugins extending and supplementing yum in different ways. It included in the base repo (which is enabled by default). However if you install as minimal you have to install it manually by typing. 

```
sudo yum -y install yum-utils
```

Read more about yum-utils 
https://www.if-not-true-then-false.com/2012/delete-remove-old-kernels-on-fedora-centos-red-hat-rhel/
https://www.tecmint.com/linux-yum-package-management-with-yum-utils/

### 2 Extra Packages for Enterprise Linux (EPEL)

How do I install the extra repositories such as Fedora EPEL repo on a Red Hat Enterprise Linux server version 7.x or CentOS Linux server version 7.x?

You can easily install various packages by configuring a CentOS 7.x or RHEL 7.x system to use Fedora EPEL repos and third party packages. Please note that these packages are not officially supported by either CentOS or Red Hat, but provides many popular packages and apps.

#### How to install RHEL EPEL repository on Centos 7.x or RHEL 7.x

The following instructions assumes that you are running command as root user on a CentOS/RHEL 7.x system and want to use use Fedora Epel repos.

#### Install Extra Packages for Enterprise Linux repository configuration

Install epel release for CentOS and RHEL 7.x using yum install
```
sudo yum install epel-release
```
Refresh repo by typing the following command: 
```
yum repolist
```

#### Search and install package

To list all available packages under a repo called epel
```
sudo yum --disablerepo="*" --enablerepo="epel" list available
```

Example: Search and install htop package from epel repo on a CentOS/RHEL 7.x

```
sudo yum search htop
sudo yum info htop
sudo yum install htop
```

And, there you have it, a larger number of packages to install from EPEL repo on a CentOS and 
Red Hat Enterprise Linux (RHEL) version 7.x.


### 3 OpenSSL

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/OpenSSL-Logo.png" alt="OPENSSL"/>
</p>

**OpenSSL** is a software library to be used in applications that need to secure communications against eavesdropping or need to ascertain the identity of the party at the other end. It has found wide use in internet web servers, serving a majority of all web sites.

**Application-Layer Protocol Negotiation (ALPN)** is a Transport Layer Security (TLS) extension for application layer protocol negotiation. ALPN allows the application layer to negotiate which protocol should be performed over a secure connection in a manner which avoids additional round trips and which is independent of the application layer protocols. It is used by HTTP/2.

***Due to TLS False Start was disabled in Google Chrome from version 20 (2012) onward except for websites with the earlier Next Protocol Negotiation (NPN) extension. NPN was replaced with a reworked version, ALPN. On July 11, 2014, ALPN was published as RFC 7301.***

#### Why Has HTTP/2 Stopped Working for Users of the New Version of Google Chrome?

As of May 2016, approximately 8% of the world’s websites are accessible over HTTP/2. These websites all use SSL/TLS, because browsers that support HTTP/2 only upgrade the connection to the new standard if SSL/TLS is also in use. The vast majority of sites – those running NGINX and LiteSpeed – depend on OpenSSL’s implementation of NPN or ALPN to upgrade to HTTP/2.

**OpenSSL added ALPN support on January 2015, in version 1.0.2. Versions 1.0.1 and earlier do not support ALPN.**

Unlike a standalone web server like NGINX, OpenSSL is a core operating system library that is used by many of the packages shipped as part of a modern Linux operating system. To ensure the operating system is stable and reliable, OS distributors do not make major updates to packages such as OpenSSL during the lifetime of each release. They do backport critical OpenSSL patches to their supported versions of OpenSSL to protect their users against OpenSSL vulnerabilities. They do not backport new features, particularly those which change the ABI of essential shared libraries.

<p align="center">
The table summarizes operating system support for ALPN and NPN.
  <a href="https://www.nginx.com/blog/supporting-http2-google-chrome-users/" target="_blank">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/alpn_os_support.PNG" alt="OS that support ALPN"/>
  </a>
</p>

As you can see, only Ubuntu >= 16.04 LTS supports ALPN. This means that if you’re running your website on any other major operating system, the OpenSSL version shipped with the operating system does not support ALPN and Chrome users will be downgraded to HTTP/1.1

So to enable HTTP/2 on ALPN in chrome browser you need to be sure that you have already installed ***OpenSSL that supported ALPN** which is version >= 1.0.2.

According to https://en.wikipedia.org/wiki/OpenSSL#Major_version_releases

**RedHat/CentOS >= 7.4 now supported Application-Layer Protocol Negotiation (ALPN)**

```
In CentOS/RedHat 7.4 the openssl package has been updated to upstream version 1.0.2k, which provides a number of enhancements, new features, and bug fixes, including:
- Added support for the Datagram Transport Layer Security TLS (DTLS) protocol version 1.2.
- Added support for the automatic elliptic curve selection for the ECDHE key exchange in TLS.
- Added support for the Application-Layer Protocol Negotiation (ALPN). 
- Added Cryptographic Message Syntax (CMS) support for the following schemes: RSA-PSS, RSA-OAEP, ECDH, and X9.42 DH.

According from https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/7.4_release_notes/index
```

To update your linux kernel just use command
```
sudo yum upgrade
```

**The following steps describe how to upgrade OpenSSL to the latest version**

Prerequired: To compile openssl from source you need to install compiler and required libraries to build nginx.
```
sudo yum groupinstall 'Development Tools'
sudo yum -y install gcc gcc-c++ make unzip autoconf automake zlib-devel pcre-devel openssl-devel zlib-devel pam-devel
```

To verify the current openssl version by command
```
openssl version
OpenSSL 1.0.1e-fips 11 Feb 2013
```

To view the lastest openssl package from current base repository
```
yum info openssl
```

1). Download the latest version of OpenSSL and generate the config, as follows:
```shell
cd /usr/local/src/
wget https://www.openssl.org/source/openssl-1.1.0-latest.tar.gz
tar -zxf openssl-1.1.0-latest.tar.gz

cd openssl-1.1.0h
./config
```

Note: 
- If you want to install other version you can download it from https://www.openssl.org/source/
- We will use latest openssl version 1.1.0 (Long Term Support Version) from official website.
- Alphabet suffix 'h' of openssl version number depends on release of openssl.

2). Compile the source code, test and then install the package (need root permission)

```
make
make test
make install
```

Recommended: 
- This will take a while depending on your CPU capacity. If your CPU has more than 1 core you can added suffix -j4 for using 4 cores to compile the source code. For example: `make -j4` to use all 4 cores to compile the source code.

3). Move old openssl installed version to the root folder for backup or you can delete it (need root permission)
```
mv /usr/bin/openssl /root/
```

4). Create a symbolic link (need root permission)
```
ln -s /usr/local/bin/openssl /usr/bin/openssl
```

5). Update new lib64 to LD_LIBRARY path then apply the config
```
export LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib64
ldconfig
````

6). Verify the OpenSSL version 
```
openssl version
OpenSSL 1.1.0h  27 Mar 2018
```


### 4 Nginx

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/NGINX_logo.png" alt="NGINX"/>
</p>

Compiling NGINX from the sources provides you with more flexibility: you can add particular NGINX modules or 3rd party modules and apply latest security patches. 

(TL;DR - This section just explain the duty of each modules, can skip to the next section)

#### NGINX Default Modules (No need to config this, it will install as default)

**NGINX consists of modules.**  The set of modules as well as other build options are configured with the ./configure script.

The following list are modules built by Default. So you're not need to configure it out.
However, you can disable it by including it to the configure script with the --without- prefix.

**http_charset_module**	Adds the specified charset to the “Content-Type” response header field, can convert data from one charset to another.

**http_gzip_module**	Compresses responses using the gzip method, helping to reduce the size of transmitted data by half or more.

**http_ssi_module**	Processes SSI (Server Side Includes) commands in responses passing through it.

**http_userid_module**	Sets cookies suitable for client identification.

**http_access_module**	Limits access to certain client addresses.

**http_auth_basic_module**	Limits access to resources by validating the user name and password using the HTTP Basic Authentication protocol.

**http_autoindex_module**	Processes requests ending with the slash character (‘/’) and produces a directory listing.

**http_geo_module**	Creates variables with values depending on the client IP address.

**http_map_module**	Creates variables whose values depend on values of other variables.

**http_split_clients_module**	Creates variables suitable for A/B testing, also known as split testing.

**http_referer_module**	Blocks access to a site for requests with invalid values in the Referer header field.

**http_rewrite_module**	Changes the request URI using regular expressions and return redirects; conditionally selects configurations. Requires the PCRE library.

**http_proxy_module**	Passes requests to another server.

**http_fastcgi_module**	Passes requests to a FastCGI server

**http_uwsgi_module**	Passes requests to a uwsgi server.

**http_scgi_module**	Passes requests to an SCGI server.d

**http_memcached_module**	Obtains responses from a memcached server.

**http_limit_conn_module**	Limits the number of connections per the defined key, in particular, the number of connections from a single IP address.

**http_limit_req_module**	Limits the request processing rate per a defined key, in particular, the processing rate of requests coming from a single IP address.

**http_empty_gif_module**	Emits single-pixel transparent GIF.

**http_browser_module**	Creates variables whose values depend on the value of the “User-Agent” request header field.

**http_upstream_hash_module**	Enables the hash load balancing method.

**http_upstream_ip_hash_module**	Enables the IP hash load balancing method.

**http_upstream_least_conn_module**	Enables the least_conn load balancing method.

**http_upstream_keepalive_module**	Enables keepalive connections.

**http_upstream_zone_module**	Enables the shared memory zone.


#### NGINX Modules Not Built by Default (Select the module what you're using or plan to)

Some NGINX modules are not built by default. You will need to enable them manually by adding to the ./configure command. The mail, stream, geoip, image_filter, perl and xslt modules can be compiled as dynamic. See Dynamic Modules for details.

**--with-threads** Enables NGINX to use thread pools.
See Thread Pools in NGINX Boost Performance 9x! blog post for details.

**--with-file-aio** Enables asynchronous I/O. (Suggest if you're working with I/O)

**--with-ipv6** Enables IPv6 support. (Very Recommended !!)

**--with-http_ssl_module** Provides support for HTTPS. Requires an SSL library such as OpenSSL. (Very Recommended !!)

**--with-http_v2_module** Provides support for HTTP/2. (Very Recommended !!)

**--with-http_realip_module** Changes the client address to the one sent in the specified header field.

**--with-http_addition_module** Adds text before and after a response.

**--with-http_xslt_module or --with-http_xslt_module=dynamic**  Transforms XML responses using one or more XSLT stylesheets. The module requires the Libxml2 and XSLT libraries. The module can also be compiled as dynamic.

**--with-http_image_filter_module or --with-http_image_filter_module=dynamic**  Transforms images in JPEG, GIF, and PNG formats. The module requires the LibGD library. The module can also be compiled as dynamic.

**--with-http_geoip_module or --with-http_geoip_module=dynamic**  Allows creating variables whose values depend on the client IP address. The module uses MaxMind GeoIP databases. The module can also be compiled as dynamic.

**--with-http_sub_module** Modifies a response by replacing one specified string by another.

**--with-http_dav_module** Intended for file management automation via the WebDAV protocol.

**--with-http_flv_module** . Provides pseudo-streaming server-side support for Flash Video (FLV) files.

**--with-mp4_module** Provides pseudo-streaming server-side support for MP4 files.

**--with-http_gunzip_module** Decompresses responses with Content-Encoding: gzip for clients that do not support zip encoding method.

**--with-http_gzip_static_module** Allows sending precompressed files with the *.gz filename extension instead of regular files.

**--with-http_auth_request_module** Implements client authorization based on the result of a subrequest.

**--with-http_random_index_module** Processes requests ending with the slash character (‘/’) and picks a random file in a directory to serve as an index file.

**--with-http_secure_link_module** Used to check authenticity of requested links, protect resources from unauthorized access, and limit link lifetime.

**--with-http_slice_module** Allows splitting a request into subrequests, each subrequest returns a certain range of response. Provides more effective caching of large files.

**--with-http_degradation_module** Allows returning an error when a memory size exceeds the defined value.

**--with-http_stub_status_module** Provides access to basic status information.

**--with-http_perl_module or --with-http_perl_module=dynamic** . Used to implement location and variable handlers in Perl and insert Perl calls into SSI. Requires the PERL library. The module can also be compiled as dynamic.

**--with-mail or --with-mail=dynamic** Enables mail proxy functionality. See the ngx_mail_core_module reference for the list of directives. The module can also be compiled as dynamic.

**--with-mail_ssl_module** Provides support for a mail proxy server to work with the SSL/TLS protocol. Requires an SSL library such as OpenSSL.

**--with-stream or --with-stream=dynamic** Enables the TCP proxy functionality. The module can also be compiled as dynamic.

**--with-stream_ssl_module** Provides support for a stream proxy server to work with the SSL/TLS protocol. Requires an SSL library such as OpenSSL.

**--with-google_perftools_module** Allows using Google Performance tools library.

**--with-cpp_test_module --with-debug** Enables the debugging log.


#### NGINX 3rd Party Modules (Optional module for compile from source method)

<p align="center">
    <img src="https://upload.wikimedia.org/wikipedia/commons/0/0f/Brotli_logo.jpeg" alt="NGINX_BROTLI_COMPRESSION"/>
</p>

**ngx_brotli module** - An open source data compression library introducing by Google, based on a modern variant of the LZ77 algorithm, Huffman coding and 2nd order context modeling. (By default nginx bundle with **gzip** compression library)
 
Brotli performance https://www.opencpu.org/posts/brotli-benchmarks/

Installing nginx brotli module https://github.com/google/ngx_brotli

```
git clone https://github.com/google/ngx_brotli.git
cd /usr/local/src/ngx_brotli 
git submodule update --init
```

-------------------------------------------------------------------------------------------------------------------------

View more NGINX 3rd Party Modules: https://www.nginx.com/resources/wiki/modules/

Read more about extending nginx https://www.nginx.com/resources/wiki/extending/

Read more about dynamic modules https://www.nginx.com/blog/dynamic-modules-nginx-1-9-11/

-------------------------------------------------------------------------------------------------------------------------

### Choosing Between a Stable or a Mainline Version

NGINX Open Source is available in 2 versions:

**The mainline version** This version includes the latest features and bugfixes and is always up-to-date. It is reliable, but it may include some experimental modules, and it may also have some number of new bugs.

**The stable version** This version doesn’t have new features, but includes critical bug fixes that are always backported to the mainline version. The stable version is recommended for production servers.


### Choosing Between a Prebuilt Package and Compiling from Source

Both the NGINX Open Source mainline and stable versions can be installed in two ways:

**Prebuilt binary package.** This is a quick and easy way to install NGINX Open Source. The package includes almost all official NGINX modules and is available for most popular operating systems. See Installing a Prebuilt Package.

**Compile from source.** This way is more flexible: you can add particular modules, including third‑party modules, or apply the latest security patches. See Compiling and Installing from Source for details.

-------------------------------------------------------------------------------------------------------------------------

### METHOD 1: Prebuilt binary package

#### Step 1 Set up the yum repository for RHEL/CentOS 

Creating the file nginx.repo in /etc/yum.repos.d, for example using vi:
```
sudo vi /etc/yum.repos.d/nginx.repo
```

Then edit nginx repository meta by add the following lines to nginx.repo: 
```Ini
[nginx]
name=nginx repo
baseurl=https://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
```
Save the config by ESC -> :x to save and exit the editor

Note: If you're using RedHat just change os name from `centos` to `rhel` 


#### Step 2 Update the repository, install and start NGINX OSS package:
```
sudo yum update
sudo yum install nginx
sudo nginx
```


#### Step 3 Verify the installation and service status:
```
sudo nginx -V
sudo systemctl status nginx
```

-------------------------------------------------------------------------------------------------------------------------

### METHOD 2: Compile from source (If you want to install custom nginx modules)

#### Step 1 - Install Compiler and build tools

To compile nginx from source you need to install compiler and required libraries to build nginx.
```
sudo yum groupinstall 'Development Tools'
sudo yum -y install autoconf automake bind-utils wget curl unzip gcc-c++ pcre-devel zlib-devel libtool make nmap-netcat ntp pam-devel
```

#### Step 2 - Download NGINX Source Code

```
cd /usr/local/src
wget http://nginx.org/download/nginx-1.14.0.tar.gz
tar zxf nginx-1.14.0.tar.gz
cd nginx-1.14.0
```

Mainline version please see https://nginx.org/en/download.html


#### Step 3 - Install NGINX Dependencies

Prior to compiling NGINX from the sources, it is necessary to install its dependencies:

PCRE library (March 3, 2018)
- The PCRE library required by NGINX Core and Rewrite modules and provides support for regular expressions:
```
cd /usr/local/src
wget https://ftp.pcre.org/pub/pcre/pcre-8.42.tar.gz
tar -zxf pcre-8.42.tar.gz
cd pcre-8.42
./configure
make
sudo make install
```

ZLIB library (January 15, 2017)
- The zlib library required by NGINX Gzip module for headers compression:
```
cd /usr/local/src
wget https://zlib.net/zlib-1.2.11.tar.gz
tar -zxf zlib-1.2.11.tar.gz
cd zlib-1.2.11
./configure
make
sudo make install
```

OpenSSL library (lastest or >= 1.0.2)
- The OpenSSL library required by NGINX SSL modules to support the HTTPS protocol:
- Please see the OpenSSL section


#### Step 4 Config source code

**NGINX Compile configure detail (TL;DR - Just explain the duty of each packages, can skip to next section)**

First, we'll set prefix to /etc/nginx for our nginx installation path. 

Next, set sbin path to /usr/sbin/nginx for storing our nginx process.

Next, set modules path to /usr/lib64/nginx/modules for storing our ngnx modules.

Next, set conf path to /etc/nginx/nginx.conf for storing our default global nginx config file.

Next, set error log path to /var/log/nginx/error.log for storing our error log file location.

Next, set http log path to /var/log/nginx/access.log for storing our http log file location.

Next, set pid path to /var/run/nginx.pid to set the name for main process ID file.

Next, set lock path to /var/run/nginx.lock to set the name for Nginx lock file.

Next, set http-client-body-temp path to /var/cache/nginx/client_temp for storing client temporary cache.

Next, set http-proxy-temp path to /var/cache/nginx/proxy_temp for storing temporary proxy.

Next, set http-fastcgi-temp path to /var/cache/nginx/fastcgi_temp for storing fastcgi cache. (commonly used in PHP)

Next, set http-uwsgi-temp path to /var/cache/nginx/uwsgi_temp for storing uwsgi cache.

Next, set http-scgi-temp path to /var/cache/nginx/scgi_temp for storing scgi cache.

Next, set user as nginx.

Next, set group as nginx.

Next, set pcre (required library) path to /usr/local/src/pcre-8.41 that where you downloaded source to.

Next, set zlib (required library) path to /usr/local/src/zlib-1.2.11 that where you downloaded source to.

Next, set openssl (required library) path to /usr/local/src/openssl-1.1.0g that where you downloaded source to.

Next, add ssl http module for enables used SSL/TLS (https) in nginx. (VERY recommeded I love HTTPS !!)

Next, add http_v2 module for enables used http2 in nginx. (Required http_ssl_module !!!)

Next, add http_realip_module to change the client address to the one sent in the specified header field.
Next, add threads, file-aio for enables NGINX to use thread pools, asynchronous I/O.

Next, add addition_module for adds text before and after a response.

Next, add sub_module to modifies a response by replacing one specified string by another.

Next, add dav_module to intend for file management automation via the WebDAV protocol.

Next, add flv_module to provide pseudo-streaming server-side support for Flash Video (FLV) files.

Next, add mp4_module to provide pseudo-streaming server-side support for MP4 files.

Next, add gunzip_module for decompresses responses with Content-Encoding: gzip for clients that do not support zip encoding 
method.

Next, add gzip_static_module for allows sending precompressed files with the *.gz filename extension instead of regular 
files.

Next, add auth_request_module to implement client authorization based on the result of a subrequest.

Next, add random_index_module for processes requests ending with the slash character (‘/’) and picks a random file in a 
directory to serve as an index file.

Next, add secure_link_module for used to check authenticity of requested links, protect resources from unauthorized access, 
and limit link lifetime.

Next, add slice_module for allows splitting a request into subrequests, each subrequest returns a certain range of 
response. Provides more effective caching of large files.

Next, add degradation_module for allows returning an error when a memory size exceeds the defined value.

Next, add stub_status_module to provides access to basic status information.

Next, add mail module to enable mail proxy functionality. (OPTIONAL)

Next, add mail_ssl_module to provide support for a mail proxy server to work with the SSL/TLS protocol. (OPTIONAL) 
(Requires an SSL library such as OpenSSL.)

Next, add stream module to enable the TCP proxy functionality. (OPTIONAL)

Next, add stream_ssl_module to provide support for a stream proxy server to work with the SSL/TLS protocol. (OPTIONAL)
(Requires an SSL library such as OpenSSL.)

NOTE: Someone ask WHERE IS ipv6 module? according to the changes with nginx 1.11.5 (11 Oct 2016) now this configure option was removed and IPv6 support is configured by default automatically. 

```
NOTE!! Please change the library version to your current library path version. For example

--with-pcre=/usr/local/src/pcre-8.xx \
--with-zlib=/usr/local/src/zlib-1.2.xx \
--with-openssl=/usr/local/src/openssl-1.1.0x \
```

4.1 Run the nginx config 
(!! REMOVE THE LAST LINE, IF YOU DON'T WANT BROTLI MODULE, SEE BROTLI MODULE SECTION !!)

```shell
./configure \
--prefix=/etc/nginx \
--sbin-path=/usr/sbin/nginx \
--modules-path=/usr/lib64/nginx/modules \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/run/nginx.lock \
--http-client-body-temp-path=/var/cache/nginx/client_temp \
--http-proxy-temp-path=/var/cache/nginx/proxy_temp \
--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
--http-scgi-temp-path=/var/cache/nginx/scgi_temp \
--user=nginx \
--group=nginx \
--with-pcre-jit \
--with-ld-opt="-lrt" \
--with-pcre=/usr/local/src/pcre-8.42 \
--with-zlib=/usr/local/src/zlib-1.2.11 \
--with-openssl=/usr/local/src/openssl-1.1.0h \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-threads \
--with-file-aio \
--with-http_addition_module \
--with-http_sub_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_auth_request_module \
--with-http_random_index_module \
--with-http_secure_link_module \
--with-http_slice_module \
--with-http_degradation_module \
--with-http_stub_status_module \
--with-mail \
--with-mail_ssl_module \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module \
--add-module=/usr/local/src/ngx_brotli 
```

4.2 Then Compile and install the build:
```
sudo make && sudo make install
```

4.3 (FOR SECURITY) Create nginx user for running nginx service

After the installation process has finished with success, add nginx system user (with /etc/nginx/ as his home directory and with no valid shell), the user that Nginx will run as by issuing the following command.
```
useradd -d /etc/nginx/ -s /sbin/nologin nginx
```

Then, change user in nginx configure file to nginx user
```Ini
vi /etc/nginx/nginx.conf
# Then change the user from nobody to nginx, then save and exit.

user nginx;
```

4.4 (FOR SECURITY) Config the firewall to allow http connnection
```
systemctl enable firewalld.service
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
systemctl restart firewalld.service
```

4.5 Start nginx service by using below command
```
/usr/sbin/nginx -c /etc/nginx/nginx.conf
```

Check nginx process 
```
ps -ef|grep nginx
```

To stop nginx service using below command
```
kill -9 PID-Of-Nginx
```

4.6 (DO ONLY ONCE)* Add nginx as systemd service 

Create a file "nginx.service" in /lib/systemd/system/nginx.service 
```
sudo vi /lib/systemd/system/nginx.service
```

Then copy service detail below into the file and save
```Ini
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
 
Then reload the system files
```
systemctl daemon-reload
```

4.7 (DO ONLY ONCE)* Create a script for automatic start nginx service at system startup
```text
sudo ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx
sudo vi /etc/init.d/nginx then copy script below
```

```bash
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15
# description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

nginx="/usr/sbin/nginx"
prog=$(basename $nginx)

NGINX_CONF_FILE="/etc/nginx/nginx.conf"

[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx

lockfile=/var/lock/subsys/nginx

make_dirs() {
   # make required directories
   user=`$nginx -V 2>&1 | grep "configure arguments:.*--user=" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   if [ -n "$user" ]; then
      if [ -z "`grep $user /etc/passwd`" ]; then
         useradd -M -s /bin/nologin $user
      fi
      options=`$nginx -V 2>&1 | grep 'configure arguments:'`
      for opt in $options; do
          if [ `echo $opt | grep '.*-temp-path'` ]; then
              value=`echo $opt | cut -d "=" -f 2`
              if [ ! -d "$value" ]; then
                  # echo "creating" $value
                  mkdir -p $value && chown -R $user $value
              fi
          fi
       done
    fi
}

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    configtest || return $?
    stop
    sleep 1
    start
}

reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}

force_reload() {
    restart
}

configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}

rh_status() {
    status $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```

4.8 (DO ONLY ONCE)* Set permission to make this script be executable 
```
chmod +x /etc/init.d/nginx
sudo systemctl enable nginx
```

4.9 (DO ONLY ONCE)* To make sure that Nginx starts and stops every time with the Droplet, add it to the default runlevels with the command:
```
sudo chkconfig nginx on
sudo service nginx restart
```

4.10 Finally, to verify nginx version and opensssl that built by type
```
nginx -V
```

4.11 Start nginx 
```
sudo systemctl start nginx
```

If you see this error
```
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (2: No such file or directory)
```

Then manually create the file /var/cache/nginx/client_temp.
```
sudo mkdir /var/cache/nginx
sudo touch /var/cache/nginx/client_temp
```

*DO ONLY ONCE mean if you build newer version of nginx and have done these steps before, you can skip these


#### Step 5 Setup NGINX Web Server 

For using http2 you need to have certificate (SSL) installed

5.1 Go to nginx global file directory

```
cd /etc/nginx/
sudo vi nginx.conf
```

5.2 Config the file as below 

```nginx
user  nginx;
worker_processes auto;
worker_rlimit_nofile 81920;
error_log /var/log/nginx/error.log crit;
pid   /var/run/nginx.pid;

events {
    worker_connections 4096; # 4096 * (2 cores) = max clients handle
    use epoll;
    multi_accept on;
}

http {
    sendfile                      on;
    tcp_nopush                    on;
    tcp_nodelay                   on;
    reset_timedout_connection     on;
    server_tokens                 off;  
    keepalive_requests            100000;
    keepalive_timeout             60s;
    send_timeout                  10s;

    include mime.types;
    default_type application/octet-stream;
    
    ##
    # OpenFile Cache Settings
    ##
    open_file_cache max=1000 inactive=20s; 
    open_file_cache_valid 30s; 
    open_file_cache_min_uses 2;
    open_file_cache_errors off;

    ##
    # Limit request per IP (DDoS prevention)
    ##    
    limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
    limit_req zone=one burst=10 nodelay;
    limit_req_status 403;

    ##
    # Buffer Limit Settings
    ##
    client_header_buffer_size    1k;
    client_body_buffer_size      128k;
    client_max_body_size         10m;
    client_header_timeout        30s;
    client_body_timeout          30s;
    large_client_header_buffers  4 256k;
    
    ##
    # Logging Settings
    ##
    access_log off; # Disable to improve I/O just enable only error

    ##
    # Proxy Cache Settings
    ##
    proxy_cache one;
    proxy_cache_min_uses 3;
    proxy_cache_revalidate on;
    proxy_cache_bypass  $http_cache_control;
    proxy_cache_path /var/nginx/cache levels=1:2 keys_zone=one:10m inactive=60m max_size=512m use_temp_path=off;
    proxy_cache_key "$host$request_uri$cookie_user";
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_valid any 1m;

    ##
    # Brotli Settings (If you not have brotli install, delete this config)
    ##
    #brotli on;
    #brotli_static on;
    #brotli_min_length 1000;
    #brotli_buffers 32 8k;
    #brotli_comp_level 6;
    #brotli_types text/plain text/css text/javascript text/xml font/opentype application/javascript application/x-javascript application/json application/xml application/xml+rss application/ecmascript image/svg+xml;
    
    ##
    # Gzip Settings (If have brotli installed, disable this config)
    ##
    gzip                        on;
    gzip_vary                   on;
    gzip_min_length             10240;
    gzip_proxied                expired no-cache no-store private auth;
    gzip_types                  text/plain text/css text/xml text/javascript application/x-javascript application/xml;
    gzip_disable                msie6;

    ##
    # Virtual Block Host Configs
    ##
    server_names_hash_bucket_size 64;
    include sites-enabled/*;
}
```

5.3 Save and test nginx config then restart nginx service

```
nginx -t
systemctl restart nginx.service
```

5.4 Go to sites-available and create the following file to build a block hosting

```
cd sites-available/
vi <your-domain-name>.conf
```

5.5 Config file as below

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name <your-domain-name> www.<your-domain-name>;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2 reuseport;
    listen [::]:443 ssl http2 reuseport;
    server_name <your-domain-name>;

    root /var/www/<web-folder-name>/html;
    index index.html;
    charset utf-8;

    error_page  404     /404.html;
    error_page  403     /403.html;
    error_page  500 502 503 504 /50x.html;

    location / {
        limit_conn conn_limit_per_ip 10;
        limit_req zone=req_limit_per_ip burst=10 nodelay;
        try_files $uri $uri/ =404;
    }

    # If you using as proxy server (eg: express, nodeJS)
    #location / {
      #proxy_http_version      1.1;
      #proxy_set_header        Upgrade                 $http_upgrade;
      #proxy_set_header        Connection              "upgrade";
      #proxy_set_header        X-Real-IP               $remote_addr;
      #proxy_set_header        X-Forwarded-For         $proxy_add_x_forwarded_for;
      #proxy_set_header        Host                    $host;
      #proxy_connect_timeout   60;
      #proxy_send_timeout      60;
      #proxy_read_timeout      60;
      #proxy_buffering         on;
      #proxy_buffer_size       128k;
      #proxy_buffers           4 256k;
      #proxy_pass              http://localhost:3000;
    }
    
    # Html and data cache timeout (no cache)
    location ~* \.(?:manifest|appcache|html?)$ {
        expires -1;
        access_log off;
    }

    # Feed cache timeout
    location ~* \.(?:rss|atom)$ {
        expires 1h;
        access_log off;
        add_header Cache-Control "public";
    }

    # Static files cache timeout
    location ~* \.(?:jpg|jpeg|gif|png|ico|webp|gz|svg|svgz|eot|woff|woff2|oft|avi|wmv|mp3|mp4|ogg|ogv|webm|xml|txt|json|csv|pdf)$ {
        expires 1d;
        access_log off;
        add_header Cache-Control "public";
    }
     
    # Style, Script files cache timeout
    location ~* \.(?:css|js)$ {
        expires 7d;
        access_log off;
        add_header Cache-Control "public";
    }
     
    # Deny scripts inside writable directories
    location ~* /(img|cache|media|logs|tmp|image|images)/.*.(php|pl|py|jsp|asp|sh|cgi)$ {
        return 403;
    }

    # SSL Certificate config
    ssl_certificate /etc/ssl/<yourweb-ssl-folder>/cert.crt;
    ssl_certificate_key /etc/ssl/<yourweb-ssl-folder>/privkey.key;
    ssl_trusted_certificate /etc/ssl/<yourweb-ssl-folder>/trustchain.crt;
    ssl_session_timeout 60m;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 4096 bits
    # to generate your dhparam.pem file, run $ openssl dhparam -out /etc/nginx/ssl/<your-domain-name>/dhparam.pem 4096
    ssl_dhparam  /etc/ssl/<yourweb-ssl-folder>/dhparam.pem;

    # SSL Key exchanges
    ssl_protocols TLSv1.2 TLSv1.3; #!! TLS 1.3 Requires nginx >= 1.13.0 !!
    ssl_ecdh_curve secp384r1;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;

    # OCSP Stapling - fetch OCSP records from URL in ssl_certificate and cache them for faster handshake
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # DNS Resolver - to lookup your upstream domain name URL
    resolver 1.1.1.1 8.8.8.8 ipv6=off;
    resolver_timeout 10s;

    # Security Header
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";
    add_header Referrer-Policy no-referrer-when-downgrade;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-XSS-Protection "1; mode=block";
    add_header Content-Security-Policy upgrade-insecure-requests;
    # Please add you own resource domain to this custom CSP!! and delete the line upper.
    # add_header Content-Security-Policy "default-src 'none'; frame-ancestors 'none'; base-uri 'none'; child-src 'none'; object-src 'none'; form-action 'self'; script-src 'self' https://storage.googleapis.com https://www.googletagmanager.com https://www.google-analytics.com https://apis.google.com https://cdnjs.cloudflare.com; img-src 'self' https: data: *.googleusercontent.com; style-src 'self' unsafe-inline https:; font-src 'self' https: data:; frame-src 'self' https:; connect-src 'self' https:; worker-src 'self';";
}

    # Public Key Pinning Extension for HTTP (HPKP) - OPTIONAL
    # to generate use one of these and put base64 text in pin-sha256:  
    # $ openssl rsa -in my-website.key -outform der -pubout | openssl dgst -sha256 -binary | base64
    # $ openssl req -in my-website.csr -pubkey -noout | openssl rsa -pubin -outform der | openssl dgst -sha256 -binary | base64
    add_header Public-Key-Pins 'pin-sha256="<base64+primary==>"; pin-sha256="<base64+backup==>"; max-age=31536000; includeSubDomains'; 
```

!!! In order to generate base64 for backup key, you need to generate a backup certificates set from CA. !!!
LEARN MORE: about HPKP: https://developer.mozilla.org/en-US/docs/Web/Security/Public_Key_Pinning

LEARN MORE: about configuration: https://mozilla.github.io/server-side-tls/ssl-config-generator/

!!! For website that include Facebook Sharing (FB.UI) using X-Frame-Options can refused to display. 
Solution: is set target="_top" on the link or window.top.location=<FBAppNameSpaces>
    
!!! For website that include Youtube iFrame using X-Frame-Options can refused to display too.
Solution: try replace https://www.youtube.com/watch?v= with https://www.youtube.com/embed/

!!! If you're using Facebook Sharing or Embedded frame eg: youtube, vimeo, facebook login button or codepen you need to set X-Frame-Options to SAMEORIGIN otherwise it will refuse to load content from external iframe.

!!! Learn more about Socket Sharding in NGINX Release 1.9.1 (reuseport)
https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/

TIP: Resolver in Nginx is a load-balancer that resolves an upstream domain name asynchronously. It chooses one IP from its buffer according to round-robin for each request. Its buffer has the latest IPs of the backend domain name. At every interval (one second by default), it resolves the domain name. If it fails to resolve the domain name, the buffer retains the last successfully resolved IPs.


5.6 Enable config file by symbolic them
```
cd /etc/nginx/sites-enabled/
ls -s /etc/nginx/sites-available/<your-domain-name>.conf /etc/nginx/sites-enabled/
```

5.7 Save and test nginx config then restart nginx service
```
nginx -t
systemctl restart nginx.service
```

5.8 Test your website SSL

https://observatory.mozilla.org/
https://www.ssllabs.com/ssltest/

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/setup-nginx-webserver/21a211ec/observation_mozilla.png" alt="MOZILLA_OBSERVVATORY_result"/>
</p>

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/setup-webserver-centos7/12a2c363/ssllab_result_.png" alt="SSLLAB_result"/>
</p>

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/setup-webserver-centos7/12a2c363/handshake_result.png" alt="handshake_Result"/>
</p>

Read more about 7 Tips for Faster HTTP/2 Performance
https://www.nginx.com/blog/7-tips-for-faster-http2-performance/


#### Step 6: Config SELinux to allow NGINX process (FOR ENHANCED SECURITY)

<p align="center">
    <img src="https://github.com/jukbot/setup-webserver-centos7/blob/master/selinux_diagram.png" alt="SELinux_diagram"/>
</p>
 
 
#### Introduction

Security Enhanced Linux or SELinux is an advanced access control mechanism built into most modern Linux distributions. 
Many system administrators find SELinux a somewhat uncharted territory. The topic can seem daunting and at times quite confusing. 

However, a properly configured SELinux system can greatly reduce security risks, and knowing a bit about it can help you troubleshoot access-related error messages. In this section we will guide the basic config of SELinux for Nginx process
We will also see a few practical instances of putting SELinux in action.

#### SELinux Modes
First off, a quick overview of the three different SELinux modes. SELinux can be in enforcing, permissive, or disabled mode.

| Policy Mode | Detail | 
|---------|---------------|
| Enforcing | This is the default. In enforcing mode, if something happens on the system that is against the defined policy, the action will be both blocked and logged. |
| Permissive | This mode will not actually block or deny anything from happening, however it will log anything that would have normally been blocked in enforcing mode. It’s a good mode to use if you perhaps want to test a Linux system that has never used SELinux and you want to get an idea of any problems you may have. No system reboot is needed when swapping between permissive and enforcing modes. |
| Disabled | Disabled is completely turned off, nothing is logged at all. In order to swap to the disabled mode, a system reboot will be required. Additionally if you are switching from disabled mode to either permissive or enforcing modes a system reboot will also be required. |

#### View current SELinux Status 

CentOS/RHEL use SELinux in enforcing mode by default, there are a few ways that we can check and confirm this. 

Run command sestatus to view current SELinux mode

```shell
$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28
````

As shown above both of these show that we are currently in enforcing mode.

Note: SELinux is incredibly valuable as part of an overall Linux system security strategy, and we recommend leaving it enabled in enforcing mode in production environments where possible. If a particular application or package does not work properly with SELinux customized allowances can be made which is the preferred option compared to simply disabling the whole thing.

#### Let's config our policy

In this section, we will be running the commands as the root user unless otherwise stated. If you don't have access to the root account and use another account with sudo privileges, you need to precede the commands with the sudo keyword.

6.1 First, we'll install selinux manager
```
sudo yum install policycoreutils-python
```

By default, web server will not have permision to connect to socket directly, in this case if we want to use it as reverse proxy but SELinux not allow to use socket. 

Most people will disable SELinux policy to fix this issue. In the worst case if other application that not Nginx has been hacked from backdoor the SELinux will keep log and process report. Which you can view using audit2allow as follow: 

6.2 Export audit of nginx process and export as policy package (.pp) file
```
sudo cat /var/log/audit/audit.log | grep nginx | grep denied | audit2allow -m nginxlocalconf > nginxlocalconf.pp
```

6.3 Then install our exported policy package by command
```
semodule -i nginxlocalconf.pp
```

Done, after installed this will append to default system policy in SELinux. Which allow nginx to access the socket safely.

Reference: 
- https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/
- https://gist.github.com/plentz/6737338
- https://www.digitalocean.com/community/tutorial_series/an-introduction-to-selinux-on-centos-7
- https://www.rootusers.com/how-to-enable-or-disable-selinux-in-centos-rhel-7/
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/7.4_release_notes/index
- https://www.nginx.com/resources/wiki/start/topics/examples/SSL-Offloader/
