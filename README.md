# Setup a Perfect CentOS 7 Guide

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/Centos-logo-light.svg" alt="PHP7"/>
</p>

Please read: This article **compatibility with RedHat Enterprise Linux 7**, except some repository links need to change upon vendors and processor architecture of your hardware.

## Introduction

The new CentOS 7 server has to be customized before it can be put into use as a production system. In this article, will help you to increase the security and usability of your server with the lastest stable packages and will give you a solid foundation for subsequent actions.

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

### Step 3 — Configure the Timezones and Network Time Protocol Synchronization

The next step is to adjust the localization settings for your server and configure the Network Time Protocol (NTP) synchronization.

The first step will ensure that your server is operating under the correct time zone. The second step will configure your system to synchronize its system clock to the standard time maintained by a global network of NTP servers. This will help prevent some inconsistent behavior that can arise from out-of-sync clocks.

#### Configure Timezones

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

#### Configure NTP Synchronization

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

#### How do I see the current time zone?

Type the date command or the ls command:
```
date
ls -l /etc/localtime
```

### Step 4 - Enable the IPTables Firewall (Recommended)

By default, the active firewall application on a newly activated CentOS 7 server is FirewallD. Though it is a good replacement for IPTables, many security applications still do not have support for it. So if you'll be using any of those applications, like OSSEC HIDS, it's best to disable/uninstall FirewallD.

Let's start by disabling/uninstalling FirewallD:
```
sudo yum remove -y firewalld
```
Now, let's install/activate IPTables.

```
sudo yum install -y iptables-services
sudo systemctl start iptables
```

Configure IPTables to start automatically at boot time.
```
sudo systemctl enable iptables
```
IPTables on CentOS 7 comes with a default set of rules, which you can view with the following command.
```
sudo iptables -L -n
```
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
```
sudo /usr/libexec/iptables/iptables.init save
```
That command will save the rules to the /etc/sysconfig/iptables file. You can edit the rules anytime by changing this file with your favorite text editor.

### Step 5 - Allow Additional Traffic Through the Firewall

Since you'll most likely be going to use your new server to host some websites at some point, you'll have to add new rules to the firewall to allow HTTP and HTTPS traffic. To accomplish that, open the IPTables file:

```
sudo nano /etc/sysconfig/iptables
```

Just after or before the SSH rule, add the rules for HTTP (port 80) and HTTPS (port 443) traffic, so that that portion of the file appears as shown in the code block below.
```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
```
Save and close the file, then reload IPTables.
```
sudo systemctl reload iptables
```
With the above step completed, your CentOS 7 server should now be reasonably secure and be ready for use in production.


### Step 6 - Optimized system configuration for maximum concurrancy.

In default centos server is not fully optimized to make full use of available hardware. 
This means it might fail under high load. So we need to config the sysctl.conf file for optimization.

1. Open sysctl.conf file
```
sudo vi /etc/sysctl.conf
```

2. Edit file as following configuration
```
Last login: Sun Dec 25 14:13:33 on console
Juks-MacBook-Pro:~ juk$ \! . 2to3 2to3- 2to3-2.7 2to32.6 \: AppleFileServer AssetCacheLocatorUtil BootCacheControl BuildStrings CpMac DeRez DevToolsSecurity DirectoryService FileStatsAgent GetFileInfo IOAccelMemory KernelEventAgent MergePef MvMac NetBootClientStatus PasswordService ResMerger Rez RezDet RezWack SetFile SplitForks UnRezWack WirelessRadioManagerd \[ \[\[ ]] a2p a2p5.16 a2p5.18 ab ac accept accton addftinfo afclip afconvert afhash afida afinfo afmtodit afplay afscexpand agentxtrap agvtool alias amt apachectl apfs_hfs_convert applesingle appletviewer apply appsleepd apropos apt apxs ar arch arp as asa asctl aslmanager asr assetutil at atos atq atrm atsutil audit auditd auditreduce autodiskmount automator automount auval auvaltool avbdeviced avbdiagnose avbutil avconvert avmetareadwrite awk banner base64 basename bash bashbug batch bc bg biff bind binhex binhex.pl binhex5.18.pl bioutil bison bitesize.d bless blued bluetoothaudiod bnepd brctl break brew bsdtar bspatch btmmdiagnose builtin bunzip2 bzcat bzcmp bzdiff bzegrep bzfgrep bzgrep bzip2 bzip2recover bzless bzmore c++ c++filt c2ph c2ph5.16 c2ph5.18 c89 c99 c_rehash caffeinate cal calendar caller cancel cap_mkdb captoinfo case cat cc cd certtool cfprefsd chat checkgid checknr chflags chfn chgrp chmod chown chpass chroot chsh cksum clang clang++ clear cmp cmpdylib codesign codesign_allocate col colcrt colldef colrm column comm command compgen complete compress config_data config_data5.16 config_data5.18 continue coreaudiod corelist corelist5.16 corelist5.18 cp cpan cpan2dist cpan2dist5.16 cpan2dist5.18 cpan5.16 cpan5.18 cpanp cpanp-run-perl cpanp-run-perl5.16 cpanp-run-perl5.18 cpanp5.16 cpanp5.18 cpio cpp cpu_profiler.d cpuwalk.d crc32 crc325.16 crc325.18 creatbyproc.d createhomedir crlrefresh cron crontab csdiagnose csgather csh csplit csreq csrutil ctags ctf_insert cu cups-config cupsaccept cupsaddsmb cupsctl cupsdisable cupsenable cupsfilter cupsreject cupstestdsc cupstestppd curl curl-config cut cvadmin cvaffinity cvcp cvdb cvdbset cvfsck cvfsdb cvfsid cvgather cvlabel cvmkdir cvmkfile cvmkfs cvupdatefs cvversions dappprof dapptrace darwinup date db_archive db_checkpoint db_codegen db_deadlock db_dump db_hotbackup db_load db_printlog db_recover db_stat db_upgrade db_verify dbicadmin dbicadmin5.16 dbicadmin5.18 dbilogstrip dbilogstrip5.16 dbilogstrip5.18 dbiprof dbiprof5.16 dbiprof5.18 dbiproxy dbiproxy5.16 dbiproxy5.18 dbmmanage dc dd ddns-confgen debinhex.pl debinhex5.18.pl declare defaults defragcli desdp dev_mkdb df diff diff3 diffstat dig dirname dirs diskhits disklabel disktool diskutil disown dispqlen.d distnoted ditto dmesg dnctl dns-sd dnsextd do domainname done dot_clean drutil dscacheutil dscl dsconfigad dsconfigldap dseditgroup dsenableroot dserr dsexport dsimport dsmemberutil dsymutil dtrace dtruss du dwarfdump dynamic_pager easy_install easy_install-2.6 easy_install-2.7 echo ed edquota egrep elif else emacs emacs-undumped emacsclient emond enable enc2xs enc2xs5.16 enc2xs5.18 encode_keychange env eqn erb errinfo esac eslint etags eval ex exec execsnoop exit expand expect export expr extcheck eyapp eyapp5.16 eyapp5.18 false fc fcgistarter fddist fdesetup fdisk fg fgrep fi fibreconfig file filebyproc.d filecoordinationd filtercalltree find find2perl find2perl5.16 find2perl5.18 findrule findrule5.16 findrule5.18 finger firmwarepasswd fixproc flex flex++ fmt fold fontrestore footprint for format-sql format-sql5.16 format-sql5.18 from fs_usage fsck fsck_apfs fsck_cs fsck_exfat fsck_hfs fsck_msdos fsck_udf fstyp fstyp_hfs fstyp_msdos fstyp_ntfs fstyp_udf ftp function funzip fuser fwkdp fwkpfv g++ gatherheaderdoc gcc gcore gcov gdiffmk gem gen_bridge_metadata gencat genstrings getconf getopt getopts git git-credential-osxkeychain git-cvsserver git-lfs git-receive-pack git-shell git-upload-archive git-upload-pack github gitk gm4 gnuattach gnuclient gnudoit gnumake gnuserv gperf gpt graphicssession grep grn grodvi groff groffer grog grolbp grolj4 grops grotty groups gssd gunzip gzcat gzexe gzip h2ph h2ph5.16 h2ph5.18 h2xs h2xs5.16 h2xs5.18 halt hash hdid hdik hdiutil hdxml2manxml head headerdoc2html heap heap32 help hexdump hidutil history hiutil host hostinfo hostname hotspot.d hpftodit htcacheclean htdbm htdigest htmltree htmltree5.16 htmltree5.18 htop htpasswd httpd httpd-wrapper httpdstat.d httxt2dbm ibtool iconutil iconv ictool id idle idle2.6 idle2.7 idlj if ifconfig imptrace in indent indxbib info infocmp infokey infotocap install install-info install_name_tool installer instmodsh instmodsh5.16 instmodsh5.18 instruments ioalloccount ioclasscount iofile.d iofileb.d iopattern iopending ioreg iosnoop iostat iotop ip2cc ip2cc5.16 ip2cc5.18 ipconfig ipcount ipcount5.16 ipcount5.18 ipcrm ipcs ippfind ipptool iprofiler iptab iptab5.16 iptab5.18 irb jar jarsigner java javac javadoc javah javap javapackager javaws jcmd jconsole jcontrol jdb jdeps jhat jhsdb jimage jinfo jjs jmap jmc jobs join jot jps jrunscript jsadebugd jshell json_pp json_pp5.16 json_pp5.18 jstack jstat jstatd jvisualvm kadmin kadmin.local kcc kdcsetup kdestroy kextcache kextfind kextlibs kextload kextstat kextunload kextutil(arg: 1) \! . 2to3 2to3- 2to3-2.7 2to32.6 \: AppleFileServer AssetCacheLocatorUtil BootCacheControl BuildStrings CpMac DeRez DevToolsSecurity DirectoryService FileStatsAgent GetFileInfo IOAccelMemory KernelEventAgent MergePef MvMac NetBootClientStatus PasswordService ResMerger Rez RezDet RezWack SetFile SplitForks UnRezWack WirelessRadioManagerd \[ \[\[ ]] a2p a2p5.16 a2p5.18 ab ac accept accton addftinfo afclip afconvert afhash afida afinfo afmtodit afplay afscexpand agentxtrap agvtool alias amt apachectl apfs_hfs_convert applesingle appletviewer apply appsleepd apropos apt apxs ar arch arp as asa asctl aslmanager asr assetutil at atos atq atrm atsutil audit auditd auditreduce autodiskmount automator automount auval auvaltool avbdeviced avbdiagnose avbutil avconvert avmetareadwrite awk banner base64 basename bash bashbug batch bc bg biff bind binhex binhex.pl binhex5.18.pl bioutil bison bitesize.d bless blued bluetoothaudiod bnepd brctl break brew bsdtar bspatch btmmdiagnose builtin bunzip2 bzcat bzcmp bzdiff bzegrep bzfgrep bzgrep bzip2 bzip2recover bzless bzmore c++ c++filt c2ph c2ph5.16 c2ph5.18 c89 c99 c_rehash caffeinate cal calendar caller cancel cap_mkdb captoinfo case cat cc cd certtool cfprefsd chat checkgid checknr chflags chfn chgrp chmod chown chpass chroot chsh cksum clang clang++ cledr cmp cmpdylib codesign codesign_allocate col colcrt colldef colrm column comm command compgen complete compress config_data config_data5.16 config_data5.18 continue coreaudiod corelist corelist5.16 corelist5.18 cp cpan cpan2dist cpan2dist5.16 cpan2dist5.18 cpan5.16 cpan5.18 cpanp cpanp-run-perl cpanp-run-perl5.16 cpanp-run-perl5.18 cpanp5.16 cpanp5.18 cpio cpp cpu_profiler.d cpuwalk.d crc32 crc325.16 crc325.18 creatbyproc.d createhomedir crlrefresh cron crontab csdiagnose csgather csh csplit csreq csrutil ctags ctf_insert cu cups-config cupsaccept cupsaddsmb cupsctl cupsdisable cupsenable cupsfilter cupsreject cupstestdsc cupstestppd curl curl-config cut cvadmin cvaffinity cvcp cvdb cvdbset cvfsck cvfsdb cvfsiJuks-MacBook-Pro:~ juk$ \! . 2to3 2to3- 2to3-2.7 2to32.6 \: AppleFileServer AssetCacheLocatorUtil BootCacheControl BuildStrings CpMac DeRez DevToolsSecurity DirectoryService FileStatsAgent GetFileInfo IOAccelMemory KernelEventAgent MergePef MvMac NetBootClientStatus PasswordService ResMerger Rez RezDet RezWack SetFile SplitForks UnRezWack WirelessRadioManagerd \[ \[\[ ]] a2p a2p5.16 a2p5.18 ab ac accept accton addftinfo afclip afconvert afhash afida afinfo afmtodit afplay afscexpand agentxtrap agvtool alias amt apachectl apfs_hfs_convert applesingle appletviewer apply appsleepd apropos apt apxs ar arch arp as asa asctl aslmanager asr assetutil at atos atq atrm atsutil audit auditd auditreduce autodiskmount automator automount auval auvaltool avbdeviced avbdiagnose avbutil avconvert avmetareadwrite awk banner base64 basename bash bashbug batch bc bg biff bind binhex binhex.pl binhex5.18.pl bioutil bison bitesize.d bless blued bluetoothaudiod bnepd brctl break brew bsdtar bspatch btmmdiagnose builtin bunzip2 bzcat bzcmp bzdiff bzegrep bzfgrep bzgrep bzip2 bzip2recover bzless bzmore c++ c++filt c2ph c2ph5.16 c2ph5.18 c89 c99 c_rehash caffeinate cal calendar caller cancel cap_mkdb captoinfo case cat cc cd certtool cfprefsd chat checkgid checknr chflags chfn chgrp chmod chown chpass chroot chsh cksum cl.ng clang++ clear cmp cmpdylib codesign codesign_allocate col colcrt colldef colrm column comm command compgen complete compress config_data config_data5.16 config_data5.18 continue coreaudiod corelist corelist5.16 corelist5.18 cp cpan cpan2dist cpan2dist5.16 cpan2dist5.18 cpan5.16 cpan5.18 cpanp cpanp-run-perl cpanp-run-perl5.16 cpanp-run-perl5.18 cpanp5.16 cpanp5.18 cpio cpp cpu_profiler.d cpuwalk.d crc32 crc325.16 crc325.18 creatbyproc.d createhomedir crlrefresh cron crontab csdiagnose csgather csh csplit csreq csrutil ctags ctf_insert cu cups-config cupsaccept cupsaddsmb cupsctl cupsdisable cupsenable cupsfilter cupsreject cupstestdsc cupstestppd curl curl-config cut cvadmin cvaffinity cvcp cvdb cvdbset cvfsck cvfsdb cvfsid cvgather cvlabel cvmkdir cvmkfile cvmkfs cvupdatefs cvversions dappprof dapptrace darwinup date db_archive db_checkpoint db_codegen db_deadlock db_dump db_hotbackup db_load db_printlog db_recover db_stat db_upgrade db_verify dbicadmin dbicadmin5.16 dbicadmin5.18 dbilogstrip dbilogstrip5.16 dbilogstrip5.18 dbiprof dbiprof5.16 dbiprof5.18 dbiproxy dbiproxy5.16 dbiproxy5.18 dbmmanage dc dd ddns-confgen debinhex.pl debinhex5.18.pl declare defaults defragcli desdp dev_mkdb df diff diff3 diffstat dig dirname dirs diskhits disklabel disktool diskutil disown dispqlen.d distnoted ditto dmesg dnctl dns-sd dnsextd do domainname done dot_clean drutil dscacheutil dscl dsconfigad dsconfigldap dseditgroup dsenableroot dserr dsexport dsimport dsmemberutil dsymutil dtrace dtruss du dwarfdump dynamic_pager easy_install easy_install-2.6 easy_install-2.7 echo ed edquota egrep elif else emacs emacs-undumped emacsclient emond enable enc2xs enc2xs5.16 enc2xs5.18 encode_keychange env eqn erb errinfo esac eslint etags eval ex exec execsnoop exit expand expect export expr extcheck eyapp eyapp5.16 eyapp5.18 false fc fcgistarter fddist fdesetup fdisk fg fgrep fi fibreconfig file filebyproc.d filecoordinationd filtercalltree find find2perl find2perl5.16 find2perl5.18 findrule findrule5.16 findrule5.18 fingerpasrmwarepasswd fixproc flex flex++ fmt fold fontrestore footprint for format-sql format-sql5.16 format-sql5.18 from fs_usage fsck fsck_apfs fsck_cs fsck_exfat fsck_hfs fsck_msdos fsck_udf fstyp fstyp_hfs fstyp_msdos fstyp_ntfs fstyp_udf ftp function funzip fuser fwkdp fwkpfv g++ gatherheaderdoc gcc gcore gcov gdiffmk gem gen_bridge_metadata gencat genstringso etconf getopt getopts git git-credential-osxkeychain git-cvsserver git-lfs git-receive-pack git-shell git-upload-archive git-upload-pack github gitk gm4 gnuattach gnuclient gnudoit gnumake gnuserv gperf gpt graphicssession grep grn grodvi groff groffer grog grolbp grolj4 grops grotty groups gssd gunzip gzcat gzexe gzip h2ph h2ph5.16 h2ph5.18 h2xs h2xs5.16 h2xs5.18 halt hash hdid hdik hdiutil hdxml2manxml head headerdoc2html heap heap32 help hexdump hidutil history hiutil host hostinfo hostname hotspot.d hpftodit htcacheclean htdbm htdigest htmltree htmltree5.16 htmltree5.18 htop htpasswd httpd httpd-wrapper httpdstat.d httxt2dbm ibtool iconutil iconv ictool id idle idle2.6 idle2.7 idlj if ifconfig imptrace indandent indxbib info infocmp infokey infotocap install install-info install_name_tool installer instmodsh instmodsh5.16 instmodsh5.18 instruments ioalloccount ioclasscount iofile.d iofileb.d iopattern iopending ioreg iosnoop iostat iotop ip2cc ip2cc5.16 ip2cc5.18 ipconfig ipcount ipcount5.16 ipcount5.18 ipcrm ipcs ippfind ipptool iprofiler iptab iptab5.16 iptab5.18 irb jar jarsigner java javac javadoc javah javap javapackager javaws jcmd jconsole jcontrol jdb jdeps jhat jhsdb jimage jinfo jjs jmap jmc jobs join jot jps jrunscript jsadebugd jshell json_pp json_pp5.16 json_pp5.18 jstack jstat jstptd jvisualvm kadmin kadmin.local kcc kdcsetup kdestroy kextcache kextfind kextlibs kextload kextstat kextunload kextutil keytool kgetcred kill kill.d killall kinit klist kpasswd krb5-config krbservicesetup ksh kswitch ktutil lam languagesetup last lastcomm lastwords latency launchctl launchd layerutil ld ldapadd ldapcompare ldapdelete ldapexop ldapmodify ldapmodrdn ldappasswd ldapsearch ldapurl ldapwhoami leaks leaks32 leave less lessecho let lex libnetcfg libnetcfg5.16 libnetcfg5.18 libtool link lipo lkbib lldb llvm-g++ llvm-gcc ln loads.d local locale localedef localemanager locate lockstat log logger login logname logout logresolve look lookbib lorder lp lpadmin lpc lpinfo lpmove lpoptions lpq lpr lprm lpstat ls lsappinfo lsbom lskq lsm lsmp lsof lsvfs lwp-download lwp-download5.16 lwp-download5.18 lwp-dump lwp-dump5.16 lwp-dump5.18 lwp-mirror lwp-mirror5.16 lwp-mirror5.18 lwp-request lwp-request5.16 lwp-request5.18 m4 mDNSResponder mDNSResponderHelper macbinary macerror macerror5.16 macerror5.18 machine mail mailq mailx make makeinfo malloc_history malloc_history32 man manpath mcxquery mcxrefresh md5 mddiagnose mdfind mdimport mdimport32 mdls mdutil memory_pressure mesg mib2c mib2c-update mig mkbom mkdep mkdir mkextunpack mkfifo mkfile mklocale mknod mkpassdb mktemp mmroff mnthome moo-outdated moo-outdated5.18 moose-outdated moose-outdated5.16 moose-outdated5.18 more mount mount_acfs mount_afp mount_apfs mount_cd9660 mount_cddafs mount_devfs mount_exfat mount_fdesc mount_ftp mount_hfs mount_msdos mount_nfs mount_ntfs mount_smbfs mount_udf mount_webdav mp2bug mpioutil msgs mtree mv nano nasm native2ascii nbdst nc ncal ncctl ncdestroy ncinit nclist ncu ncurses5.4-config ndisasm ndp neqn net-server net-server5.16 net-server5.18 net-snmp-cert net-snmp-config net-snmp-create-v3-user netbiosd netstat nettop networksetup newaliases newfs_apfs newfs_exfat newfs_hfs newfs_msdos newfs_udf newgrp newproc.d newsyslog nfs4mapid nfsd nfsiod nfsstat nice nl nlcontrol nm nmedit node nohup nologin notifyd notifyutil npm npm-check-updates nroff nscurl nslookup nsupdate ntp-keygen ntpd ntpdate ntpdc ntpq ntptrace nvram objdump ocspd od odutil open opendiff opensnoop openssl orbd osacompile osadecompile osalang osascript otool pack200 package-stash-conflicts package-stash-conflicts5.16 package-stash-conflicts5.18 pagesize pagestuff par.pl par5.16.pl par5.18.pl parl parl5.16 parl5.18 parldyn parldyn5.16 parldyn5.18 passwd paste patch pathchk pathopens.d pax pbcopy pbpaste pcap-config pcsctest pdisk periodic perl perl5.16 perl5.18 perlbug perlbug5.16 perlbug5.18 perldoc perldoc5.16 perldoc5.18 perlivp perlivp5.16 perlivp5.18 perlthanks perlthanks5.16 perlthanks5.18 pfbtops pfctl pgrep phar phar.phar php php-config php-fpm phpize pic pico piconv piconv5.16 piconv5.18 pictd pidpersec.d ping ping6 pkgbuild pkgutil pkill pl pl2pm pl2pm5.16 pl2pm5.18 plink plockstat pluginkit plutil pmset pod2html pod2html5.16 pod2html5.18 pod2latex pod2latex5.16 pod2latex5.18 pod2man pod2man5.16 pod2man5.18 pod2readme pod2readme5.16 pod2readme5.18 pod2text pod2text5.16 pod2text5.18 pod2usage pod2usage5.16 pod2usage5.18 podchecker podchecker5.16 podchecker5.18 podselect podselect5.16 podselect5.18 policytool popd post-grohtml postalias postcat postconf postdrop postfix postkick postlock postlog postmap postmulti postqueue postsuper power_report.sh powermetrics pp pp5.16 pp5.18 ppdc ppdhtml ppdi ppdmerge ppdpo pppd pr praudit pre-grohtml priclass.d pridist.d printenv printf procsystime productbuild productsign profiles prove prove5.16 prove5.18 ps pscp psed psed5.16 psed5.18 psftp pstopdf pstruct pstruct5.16 pstruct5.18 ptar ptar5.16 ptar5.18 ptardiff ptardiff5.16 ptardiff5.18 ptargrep ptargrep5.16 ptargrep5.18 pubsub purge pushd puttygen pwd pwd_mkdb pwhich pwhich5.16 pwhich5.18 pwpolicy pydoc pydoc2.6 pydoc2.7 python python-config python2.6 python2.6-config python2.7 python2.7-config pythonw pythonw2.6 pythonw2.7 qc2movie qlmanage qtdefaults qtmodernizer quota quotacheck quotaoff quotaon racoon raidutil rails rake ranlib rarpd rdoc read readlink readonly rebase reboot redo_prebinding refer reject renice repairHomePermissions repquota reset resolveLinks return rev ri rm rmdir rmic rmid rmiregistry rotatelogs route rpc.lockd rpc.statd rpcbind rpcgen rpcinfo rs rsync rtadvd rtsol rtsold ruby runocc.d rvictl rview rvim rwbypid.d rwbytype.d rwsnoop s2p s2p5.16 s2p5.18 sa safaridriver sample sampleproc sandbox-exec say sc_auth sc_usage scandeps.pl scandeps5.16.pl scandeps5.18.pl schemagen scp screen screencapture scriptpnote                                           orce zgrep zic zip zipcloak zipdetails zipdetails5.16 zipdetails5.18 zip
Juks-MacBook-Pro:~ juk$ 
Juks-MacBook-Pro:~ juk$ 
Juks-MacBook-Pro:~ juk$ npm install -g polymer-cli
npm WARN deprecated minimatch@2.0.10: Please update to minimatch 3.0.2 or higher to avoid a RegExp DoS issue
npm WARN deprecated minimatch@0.2.14: Please update to minimatch 3.0.2 or higher to avoid a RegExp DoS issue
npm WARN deprecated graceful-fs@1.2.3: graceful-fs v3.0.0 and before will fail on node releases >= v7.0. Please update to graceful-fs@^4.0.0 as soon as possible. Use 'npm ls graceful-fs' to find it in the tree.
npm WARN deprecated node-uuid@1.4.7: use uuid module instead
npm WARN checkPermissions Missing write access to /usr/local/lib/node_modules
/usr/local/lib
`-- polymer-cli@0.17.0 
  +-- @types/chalk@0.4.31 
  +-- @types/del@2.2.31 
  | `-- @types/glob@5.0.30 
  +-- @types/findup-sync@0.3.29 
  | `-- @types/minimatch@2.0.29 
  +-- @types/fs-extra@0.0.34 
  +-- @types/gulp-if@0.0.30 
  +-- @types/html-minifier@1.1.30 
  | +-- @types/clean-css@3.4.30 
  | `-- @types/relateurl@0.2.28 
  +-- @types/inquirer@0.0.30 
  | +-- @types/rx@2.5.34 
  | `-- @types/through@0.0.28 
  +-- @types/merge-stream@1.0.28 
  +-- @types/node@6.0.53 
  +-- @types/parse5@2.2.34 
  +-- @types/request@0.0.32 
  | `-- @types/form-data@0.0.33 
  +-- @types/temp@0.8.29 
  +-- @types/uglify-js@2.6.28 
  | `-- @types/source-map@0.5.0 
  +-- @types/vinyl@1.2.30 
  +-- @types/vinyl-fs@0.0.28 
  | `-- @types/glob-stream@3.1.30 
  +-- @types/yeoman-generator@0.0.30 
  +-- chalk@1.1.3 
  | +-- ansi-styles@2.2.1 
  | +-- escape-string-regexp@1.0.5 
  | +-- has-ansi@2.0.0 
  | | `-- ansi-regex@2.0.0 
  | +-- strip-ansi@3.0.1 
  | `-- supports-color@2.0.0 
  +-- command-line-args@3.0.5 
  | +-- array-back@1.0.4 
  | +-- feature-detect-es6@1.3.1 
  | +-- find-replace@1.0.2 
  | | `-- test-value@2.1.0 
  | `-- typical@2.6.0 
  +-- command-line-commands@1.0.4 
  +-- command-line-usage@3.0.8 
  | +-- ansi-escape-sequences@3.0.0 
  | `-- table-layout@0.3.0 
  |   +-- core-js@2.4.1 
  |   +-- deep-extend@0.4.1 
  |   `-- wordwrapjs@2.0.0 
  |     `-- reduce-flatten@1.0.1 
  +-- css-slam@1.2.1 
  | +-- dom5@1.3.6 
  | | +-- @types/node@4.0.30 
  | | +-- @types/parse5@0.0.31 
  | | | `-- @types/node@6.0.53 
  | | `-- parse5@1.5.1 
  | `-- shady-css-parser@0.0.8 
  +-- del@2.2.2 
  | +-- globby@5.0.0 
  | | +-- array-union@1.0.2 
  | | | `-- array-uniq@1.0.3 
  | | `-- arrify@1.0.1 
  | +-- is-path-cwd@1.0.0 
  | +-- is-path-in-cwd@1.0.0 
  | | `-- is-path-inside@1.0.0 
  | |   `-- path-is-inside@1.0.2 
  | +-- object-assign@4.1.0 
  | +-- pify@2.3.0 
  | +-- pinkie-promise@2.0.1 
  | | `-- pinkie@2.0.4 
  | `-- rimraf@2.5.4 
  +-- dom5@2.0.1 
  | +-- @types/clone@0.1.30 
  | +-- @types/node@4.0.30 
  | `-- clone@1.0.2 
  +-- findup-sync@0.4.3 
  | +-- detect-file@0.1.0 
  | | `-- fs-exists-sync@0.1.0 
  | +-- is-glob@2.0.1 
  | | `-- is-extglob@1.0.0 
  | +-- micromatch@2.3.11 
  | | +-- arr-diff@2.0.0 
  | | | `-- arr-flatten@1.0.1 
  | | +-- array-unique@0.2.1 
  | | +-- braces@1.8.5 
  | | | +-- expand-range@1.8.2 
  | | | | `-- fill-range@2.2.3 
  | | | |   +-- is-number@2.1.0 
  | | | |   +-- isobject@2.1.0 
  | | | |   +-- randomatic@1.1.6 
  | | | |   `-- repeat-string@1.6.1 
  | | | +-- preserve@0.2.0 
  | | | `-- repeat-element@1.1.2 
  | | +-- expand-brackets@0.1.5 
  | | | `-- is-posix-bracket@0.1.1 
  | | +-- extglob@0.3.2 
  | | +-- filename-regex@2.0.0 
  | | +-- kind-of@3.1.0 
  | | | `-- is-buffer@1.1.4 
  | | +-- normalize-path@2.0.1 
  | | +-- object.omit@2.0.1 
  | | | +-- for-own@0.1.4 
  | | | | `-- for-in@0.1.6 
  | | | `-- is-extendable@0.1.1 
  | | +-- parse-glob@3.0.4 
  | | | +-- glob-base@0.3.0 
  | | | | `-- glob-parent@2.0.0 
  | | | `-- is-dotfile@1.0.2 
  | | `-- regex-cache@0.4.3 
  | |   +-- is-equal-shallow@0.1.3 
  | |   `-- is-primitive@2.0.0 
  | `-- resolve-dir@0.1.1 
  |   +-- expand-tilde@1.2.2 
  |   `-- global-modules@0.2.3 
  |     +-- global-prefix@0.1.5 
  |     | +-- homedir-polyfill@1.0.1 
  |     | | `-- parse-passwd@1.0.0 
  |     | `-- ini@1.3.4 
  |     `-- is-windows@0.2.0 
  +-- fs-extra@1.0.0 
  | +-- graceful-fs@4.1.11 
  | +-- jsonfile@2.4.0 
  | `-- klaw@1.3.1 
  +-- github@6.1.0 
  | +-- follow-redirects@0.0.7 
  | | `-- stream-consume@0.1.0 
  | +-- https-proxy-agent@1.0.0 
  | | `-- agent-base@2.0.1 
  | `-- mime@1.3.4 
  +-- gulp-if@2.0.2 
  | +-- gulp-match@1.0.3 
  | | `-- minimatch@3.0.3 
  | |   `-- brace-expansion@1.1.6 
  | |     +-- balanced-match@0.4.2 
  | |     `-- concat-map@0.0.1 
  | +-- ternary-stream@2.0.1 
  | | `-- fork-stream@0.0.4 
  | `-- through2@2.0.3 
  |   `-- xtend@4.0.1 
  +-- gunzip-maybe@1.3.1 
  | +-- browserify-zlib@0.1.4 
  | | `-- pako@0.2.9 
  | +-- is-deflate@1.0.0 
  | +-- is-gzip@1.0.0 
  | +-- peek-stream@1.1.1 
  | | `-- through2@0.5.1 
  | |   +-- readable-stream@1.0.34 
  | |   | `-- isarray@0.0.1 
  | |   `-- xtend@3.0.0 
  | +-- pumpify@1.3.5 
  | | `-- inherits@2.0.3 
  | `-- through2@0.4.2 
  |   +-- readable-stream@1.0.34 
  |   | `-- isarray@0.0.1 
  |   `-- xtend@2.1.2 
  |     `-- object-keys@0.4.0 
  +-- html-minifier@3.2.3 
  | +-- camel-case@3.0.0 
  | | +-- no-case@2.3.0 
  | | | `-- lower-case@1.1.3 
  | | `-- upper-case@1.1.3 
  | +-- clean-css@3.4.23 
  | | +-- commander@2.8.1 
  | | `-- source-map@0.4.4 
  | |   `-- amdefine@1.0.1 
  | +-- commander@2.9.0 
  | | `-- graceful-readlink@1.0.1 
  | +-- he@1.1.0 
  | +-- ncname@1.0.0 
  | | `-- xml-char-classes@1.0.0 
  | +-- param-case@2.1.0 
  | `-- relateurl@0.2.7 
  +-- inquirer@1.2.3 
  | +-- ansi-escapes@1.4.0 
  | +-- cli-cursor@1.0.2 
  | | `-- restore-cursor@1.0.1 
  | |   +-- exit-hook@1.1.1 
  | |   `-- onetime@1.1.0 
  | +-- cli-width@2.1.0 
  | +-- external-editor@1.1.1 
  | | +-- spawn-sync@1.0.15 
  | | | `-- os-shim@0.1.3 
  | | `-- tmp@0.0.29 
  | +-- figures@1.7.0 
  | +-- lodash@4.17.3 
  | +-- mute-stream@0.0.6 
  | +-- run-async@2.3.0 
  | | `-- is-promise@2.1.0 
  | +-- rx@4.1.0 
  | +-- string-width@1.0.2 
  | | +-- code-point-at@1.1.0 
  | | `-- is-fullwidth-code-point@1.0.0 
  | `-- through@2.3.8 
  +-- merge-stream@1.0.1 
  | `-- readable-stream@2.2.2 
  |   +-- buffer-shims@1.0.0 
  |   +-- core-util-is@1.0.2 
  |   +-- isarray@1.0.0 
  |   +-- process-nextick-args@1.0.7 
  |   +-- string_decoder@0.10.31 
  |   `-- util-deprecate@1.0.2 
  +-- parse5@2.2.3 
  +-- plylog@0.4.0 
  | `-- winston@2.3.0 
  |   +-- async@1.0.0 
  |   +-- colors@1.0.3 
  |   +-- cycle@1.0.3 
  |   +-- eyes@0.1.8 
  |   `-- stack-trace@0.0.9 
  +-- polylint@2.10.1 
  | +-- colors@1.1.2 
  | +-- command-line-args@2.1.6 
  | | `-- command-line-usage@2.0.5 
  | |   +-- ansi-escape-sequences@2.2.2 
  | |   | `-- collect-all@0.2.1 
  | |   |   `-- stream-via@0.1.1 
  | |   +-- column-layout@2.1.4 
  | |   | +-- ansi-escape-sequences@2.2.2 
  | |   | +-- collect-json@1.0.8 
  | |   | | +-- collect-all@1.0.2 
  | |   | | +-- stream-connect@1.0.2 
  | |   | | `-- stream-via@1.0.3 
  | |   | +-- command-line-args@2.1.6 
  | |   | | `-- command-line-usage@2.0.5 
  | |   | +-- object-tools@2.0.6 
  | |   | | +-- object-get@2.1.0 
  | |   | | `-- test-value@1.1.0 
  | |   | `-- wordwrapjs@1.2.1 
  | |   `-- wordwrapjs@1.2.1 
  | +-- dom5@1.3.6 
  | | +-- @types/node@4.0.30 
  | | +-- @types/parse5@0.0.31 
  | | | `-- @types/node@6.0.53 
  | | `-- parse5@1.5.1 
  | +-- es6-set@0.1.4 
  | | +-- d@0.1.1 
  | | +-- es5-ext@0.10.12 
  | | +-- es6-iterator@2.0.0 
  | | +-- es6-symbol@3.1.0 
  | | `-- event-emitter@0.3.4 
  | +-- hydrolysis@1.24.1 
  | | +-- babel-polyfill@6.20.0 
  | | | +-- babel-runtime@6.20.0 
  | | | `-- regenerator-runtime@0.10.1 
  | | +-- doctrine@0.7.2 
  | | | +-- esutils@1.1.6 
  | | | `-- isarray@0.0.1 
  | | +-- dom5@1.3.6 
  | | | +-- @types/node@4.0.30 
  | | | +-- @types/parse5@0.0.31 
  | | | | `-- @types/node@6.0.53 
  | | | `-- parse5@1.5.1 
  | | +-- escodegen@1.8.1 
  | | | +-- esprima@2.7.3 
  | | | +-- estraverse@1.9.3 
  | | | +-- esutils@2.0.2 
  | | | +-- optionator@0.8.2 
  | | | | +-- deep-is@0.1.3 
  | | | | +-- fast-levenshtein@2.0.5 
  | | | | +-- levn@0.3.0 
  | | | | +-- prelude-ls@1.1.2 
  | | | | +-- type-check@0.3.2 
  | | | | `-- wordwrap@1.0.0 
  | | | `-- source-map@0.2.0 
  | | +-- espree@3.3.2 
  | | | +-- acorn@4.0.4 
  | | | `-- acorn-jsx@3.0.1 
  | | |   `-- acorn@3.3.0 
  | | `-- estraverse@3.1.0 
  | +-- optimist@0.6.1 
  | | +-- minimist@0.0.10 
  | | `-- wordwrap@0.0.2 
  | `-- path-is-absolute@1.0.1 
  +-- polymer-build@0.4.1 
  | +-- dom5@1.3.6 
  | | +-- @types/node@4.0.30 
  | | +-- @types/parse5@0.0.31 
  | | | `-- @types/node@6.0.53 
  | | `-- parse5@1.5.1 
  | +-- fs-extra@0.30.0 
  | +-- gulp@3.9.1 
  | | +-- archy@1.0.0 
  | | +-- deprecated@0.0.1 
  | | +-- gulp-util@3.0.7 
  | | | +-- array-differ@1.0.0 
  | | | +-- beeper@1.1.1 
  | | | +-- fancy-log@1.2.0 
  | | | | `-- time-stamp@1.0.1 
  | | | +-- gulplog@1.0.0 
  | | | | `-- glogg@1.0.0 
  | | | +-- has-gulplog@0.1.0 
  | | | | `-- sparkles@1.0.0 
  | | | +-- lodash._reescape@3.0.0 
  | | | +-- lodash._reevaluate@3.0.0 
  | | | +-- lodash._reinterpolate@3.0.0 
  | | | +-- lodash.template@3.6.2 
  | | | | +-- lodash._basecopy@3.0.1 
  | | | | +-- lodash._basetostring@3.0.1 
  | | | | +-- lodash._basevalues@3.0.0 
  | | | | +-- lodash.escape@3.2.0 
  | | | | | `-- lodash._root@3.0.1 
  | | | | +-- lodash.keys@3.1.2 
  | | | | | +-- lodash._getnative@3.9.1 
  | | | | | +-- lodash.isarguments@3.1.0 
  | | | | | `-- lodash.isarray@3.0.4 
  | | | | +-- lodash.restparam@3.6.1 
  | | | | `-- lodash.templatesettings@3.1.1 
  | | | +-- minimist@1.2.0 
  | | | +-- multipipe@0.1.2 
  | | | | `-- duplexer2@0.0.2 
  | | | |   `-- readable-stream@1.1.14 
  | | | |     `-- isarray@0.0.1 
  | | | +-- object-assign@3.0.0 
  | | | `-- vinyl@0.5.3 
  | | +-- interpret@1.0.1 
  | | +-- liftoff@2.3.0 
  | | | +-- fined@1.0.2 
  | | | | +-- lodash.assignwith@4.2.0 
  | | | | +-- lodash.isempty@4.4.0 
  | | | | +-- lodash.pick@4.4.0 
  | | | | `-- parse-filepath@1.0.1 
  | | | |   +-- is-absolute@0.2.6 
  | | | |   | `-- is-relative@0.2.1 
  | | | |   |   `-- is-unc-path@0.1.2 
  | | | |   |     `-- unc-path-regex@0.1.2 
  | | | |   +-- map-cache@0.2.2 
  | | | |   `-- path-root@0.1.1 
  | | | |     `-- path-root-regex@0.1.2 
  | | | +-- flagged-respawn@0.3.2 
  | | | +-- lodash.isplainobject@4.0.6 
  | | | +-- lodash.isstring@4.0.1 
  | | | `-- lodash.mapvalues@4.6.0 
  | | +-- minimist@1.2.0 
  | | +-- orchestrator@0.3.8 
  | | | +-- end-of-stream@0.1.5 
  | | | | `-- once@1.3.3 
  | | | `-- sequencify@0.0.7 
  | | +-- pretty-hrtime@1.0.3 
  | | +-- semver@4.3.6 
  | | +-- tildify@1.2.0 
  | | +-- v8flags@2.0.11 
  | | | `-- user-home@1.1.1 
  | | `-- vinyl-fs@0.3.14 
  | |   +-- defaults@1.0.3 
  | |   +-- glob-stream@3.1.18 
  | |   | +-- glob@4.5.3 
  | |   | +-- glob2base@0.0.12 
  | |   | | `-- find-index@0.1.1 
  | |   | +-- minimatch@2.0.10 
  | |   | +-- ordered-read-streams@0.1.0 
  | |   | +-- through2@0.6.5 
  | |   | | `-- readable-stream@1.0.34 
  | |   | |   `-- isarray@0.0.1 
  | |   | `-- unique-stream@1.0.0 
  | |   +-- glob-watcher@0.0.6 
  | |   | `-- gaze@0.5.2 
  | |   |   `-- globule@0.1.0 
  | |   |     +-- glob@3.1.21 
  | |   |     | +-- graceful-fs@1.2.3 
  | |   |     | `-- inherits@1.0.2 
  | |   |     +-- lodash@1.0.2 
  | |   |     `-- minimatch@0.2.14 
  | |   |       +-- lru-cache@2.7.3 
  | |   |       `-- sigmund@1.0.1 
  | |   +-- graceful-fs@3.0.11 
  | |   | `-- natives@1.1.0 
  | |   +-- strip-bom@1.0.0 
  | |   +-- through2@0.6.5 
  | |   | `-- readable-stream@1.0.34 
  | |   |   `-- isarray@0.0.1 
  | |   `-- vinyl@0.4.6 
  | |     `-- clone@0.2.0 
  | +-- minimatch-all@1.1.0 
  | +-- multipipe@0.3.1 
  | | `-- duplexer2@0.1.4 
  | +-- sw-precache@3.2.0 
  | | +-- dom-urls@1.1.0 
  | | | `-- urijs@1.18.4 
  | | +-- es6-promise@3.3.1 
  | | +-- glob@6.0.4 
  | | +-- lodash.defaults@4.2.0 
  | | +-- lodash.template@4.4.0 
  | | | `-- lodash.templatesettings@4.1.0 
  | | +-- meow@3.7.0 
  | | | +-- camelcase-keys@2.1.0 
  | | | | `-- camelcase@2.1.1 
  | | | +-- loud-rejection@1.6.0 
  | | | | +-- currently-unhandled@0.4.1 
  | | | | | `-- array-find-index@1.0.2 
  | | | | `-- signal-exit@3.0.2 
  | | | +-- map-obj@1.0.1 
  | | | +-- minimist@1.2.0 
  | | | +-- normalize-package-data@2.3.5 
  | | | | +-- hosted-git-info@2.1.5 
  | | | | +-- is-builtin-module@1.0.0 
  | | | | | `-- builtin-modules@1.1.1 
  | | | | `-- validate-npm-package-license@3.0.1 
  | | | |   +-- spdx-correct@1.0.2 
  | | | |   | `-- spdx-license-ids@1.2.2 
  | | | |   `-- spdx-expression-parse@1.0.4 
  | | | +-- redent@1.0.0 
  | | | | +-- indent-string@2.1.0 
  | | | | `-- strip-indent@1.0.1 
  | | | `-- trim-newlines@1.0.0 
  | | `-- sw-toolbox@3.4.0 
  | |   +-- path-to-regexp@1.7.0 
  | |   | `-- isarray@0.0.1 
  | |   `-- serviceworker-cache-polyfill@4.0.0 
  | `-- vulcanize@1.15.2 
  |   +-- dom5@1.3.6 
  |   | +-- @types/node@4.0.30 
  |   | +-- @types/parse5@0.0.31 
  |   | | `-- @types/node@6.0.53 
  |   | `-- parse5@1.5.1 
  |   +-- es6-promise@2.3.0 
  |   `-- path-posix@1.0.0 
  +-- polyserve@0.13.0 
  | +-- @types/express@4.0.34 
  | | `-- @types/express-serve-static-core@4.0.39 
  | +-- @types/mime@0.0.29 
  | +-- @types/node@4.0.30 
  | +-- @types/opn@3.0.28 
  | +-- @types/serve-static@1.7.31 
  | +-- express@4.14.0 
  | | +-- accepts@1.3.3 
  | | | `-- negotiator@0.6.1 
  | | +-- array-flatten@1.1.1 
  | | +-- content-disposition@0.5.1 
  | | +-- content-type@1.0.2 
  | | +-- cookie@0.3.1 
  | | +-- cookie-signature@1.0.6 
  | | +-- debug@2.2.0 
  | | | `-- ms@0.7.1 
  | | +-- depd@1.1.0 
  | | +-- encodeurl@1.0.1 
  | | +-- escape-html@1.0.3 
  | | +-- etag@1.7.0 
  | | +-- finalhandler@0.5.0 
  | | | +-- debug@2.2.0 
  | | | | `-- ms@0.7.1 
  | | | `-- unpipe@1.0.0 
  | | +-- fresh@0.3.0 
  | | +-- merge-descriptors@1.0.1 
  | | +-- methods@1.1.2 
  | | +-- on-finished@2.3.0 
  | | | `-- ee-first@1.1.1 
  | | +-- parseurl@1.3.1 
  | | +-- path-to-regexp@0.1.7 
  | | +-- proxy-addr@1.1.2 
  | | | +-- forwarded@0.1.0 
  | | | `-- ipaddr.js@1.1.1 
  | | +-- qs@6.2.0 
  | | +-- range-parser@1.2.0 
  | | +-- send@0.14.1 
  | | | +-- destroy@1.0.4 
  | | | `-- statuses@1.3.1 
  | | +-- serve-static@1.11.1 
  | | | `-- send@0.14.1 
  | | |   +-- debug@2.2.0 
  | | |   `-- ms@0.7.1 
  | | +-- type-is@1.6.14 
  | | | `-- media-typer@0.3.0 
  | | +-- utils-merge@1.0.0 
  | | `-- vary@1.1.0 
  | +-- find-port@1.0.1 
  | +-- opn@3.0.3 
  | `-- send@0.10.1 
  |   +-- debug@2.1.3 
  |   | `-- ms@0.7.0 
  |   +-- depd@1.0.1 
  |   +-- destroy@1.0.3 
  |   +-- escape-html@1.0.1 
  |   +-- etag@1.5.1 
  |   | `-- crc@3.2.1 
  |   +-- fresh@0.2.4 
  |   +-- mime@1.2.11 
  |   +-- ms@0.6.2 
  |   +-- on-finished@2.1.1 
  |   | `-- ee-first@1.1.0 
  |   `-- range-parser@1.0.3 
  +-- request@2.79.0 
  | +-- aws-sign2@0.6.0 
  | +-- aws4@1.5.0 
  | +-- caseless@0.11.0 
  | +-- combined-stream@1.0.5 
  | | `-- delayed-stream@1.0.0 
  | +-- extend@3.0.0 
  | +-- forever-agent@0.6.1 
  | +-- form-data@2.1.2 
  | | `-- asynckit@0.4.0 
  | +-- har-validator@2.0.6 
  | | `-- is-my-json-valid@2.15.0 
  | |   +-- generate-function@2.0.0 
  | |   +-- generate-object-property@1.2.0 
  | |   | `-- is-property@1.0.2 
  | |   `-- jsonpointer@4.0.1 
  | +-- hawk@3.1.3 
  | | +-- boom@2.10.1 
  | | +-- cryptiles@2.0.5 
  | | +-- hoek@2.16.3 
  | | `-- sntp@1.0.9 
  | +-- http-signature@1.1.1 
  | | +-- assert-plus@0.2.0 
  | | +-- jsprim@1.3.1 
  | | | +-- extsprintf@1.0.2 
  | | | +-- json-schema@0.2.3 
  | | | `-- verror@1.3.6 
  | | `-- sshpk@1.10.1 
  | |   +-- asn1@0.2.3 
  | |   +-- assert-plus@1.0.0 
  | |   +-- bcrypt-pbkdf@1.0.0 
  | |   +-- dashdash@1.14.1 
  | |   | `-- assert-plus@1.0.0 
  | |   +-- ecc-jsbn@0.1.1 
  | |   +-- getpass@0.1.6 
  | |   | `-- assert-plus@1.0.0 
  | |   +-- jodid25519@1.0.2 
  | |   +-- jsbn@0.1.0 
  | |   `-- tweetnacl@0.14.5 
  | +-- is-typedarray@1.0.0 
  | +-- isstream@0.1.2 
  | +-- json-stringify-safe@5.0.1 
  | +-- mime-types@2.1.13 
  | | `-- mime-db@1.25.0 
  | +-- oauth-sign@0.8.2 
  | +-- qs@6.3.0 
  | +-- stringstream@0.0.5 
  | +-- tough-cookie@2.3.2 
  | | `-- punycode@1.4.1 
  | +-- tunnel-agent@0.4.3 
  | `-- uuid@3.0.1 
  +-- resolve@1.2.0 
  +-- tar-fs@1.15.0 
  | +-- mkdirp@0.5.1 
  | | `-- minimist@0.0.8 
  | +-- pump@1.0.2 
  | | +-- end-of-stream@1.1.0 
  | | | `-- once@1.3.3 
  | | `-- once@1.4.0 
  | |   `-- wrappy@1.0.2 
  | `-- tar-stream@1.5.2 
  |   +-- bl@1.2.0 
  |   `-- end-of-stream@1.0.0 
  |     `-- once@1.3.3 
  +-- temp@0.8.3 
  | +-- os-tmpdir@1.0.2 
  | `-- rimraf@2.2.8 
  +-- uglify-js@2.7.5 
  | +-- async@0.2.10 
  | +-- source-map@0.5.6 
  | +-- uglify-to-browserify@1.0.2 
  | `-- yargs@3.10.0 
  |   +-- camelcase@1.2.1 
  |   +-- cliui@2.1.0 
  |   | +-- center-align@0.1.3 
  |   | | +-- align-text@0.1.4 
  |   | | | `-- longest@1.0.1 
  |   | | `-- lazy-cache@1.0.4 
  |   | `-- right-align@0.1.3 
  |   +-- decamelize@1.2.0 
  |   `-- window-size@0.1.0 
  +-- update-notifier@1.0.3 
  | +-- boxen@0.6.0 
  | | +-- ansi-align@1.1.0 
  | | +-- camelcase@2.1.1 
  | | +-- cli-boxes@1.0.0 
  | | +-- filled-array@1.1.0 
  | | +-- repeating@2.0.1 
  | | | `-- is-finite@1.0.2 
  | | `-- widest-line@1.0.0 
  | +-- configstore@2.1.0 
  | | +-- dot-prop@3.0.0 
  | | | `-- is-obj@1.0.1 
  | | +-- osenv@0.1.4 
  | | +-- uuid@2.0.3 
  | | `-- write-file-atomic@1.2.0 
  | |   +-- imurmurhash@0.1.4 
  | |   `-- slide@1.1.6 
  | +-- is-npm@1.0.0 
  | +-- latest-version@2.0.0 
  | | `-- package-json@2.4.0 
  | |   +-- registry-auth-token@3.1.0 
  | |   | `-- rc@1.1.6 
  | |   |   +-- minimist@1.2.0 
  | |   |   `-- strip-json-comments@1.0.4 
  | |   +-- registry-url@3.1.0 
  | |   `-- semver@5.3.0 
  | +-- lazy-req@1.1.0 
  | +-- semver-diff@2.1.0 
  | | `-- semver@5.0.3 
  | `-- xdg-basedir@2.0.0 
  |   `-- os-homedir@1.0.2 
  +-- vinyl@1.2.0 
  | +-- clone-stats@0.0.1 
  | `-- replace-ext@0.0.1 
  +-- vinyl-fs@2.4.4 
  | +-- duplexify@3.5.0 
  | | `-- stream-shift@1.0.0 
  | +-- glob-stream@5.3.5 
  | | +-- glob@5.0.15 
  | | +-- glob-parent@3.1.0 
  | | | +-- is-glob@3.1.0 
  | | | | `-- is-extglob@2.1.1 
  | | | `-- path-dirname@1.0.2 
  | | +-- ordered-read-streams@0.3.0 
  | | | `-- is-stream@1.1.0 
  | | +-- through2@0.6.5 
  | | | `-- readable-stream@1.0.34 
  | | |   `-- isarray@0.0.1 
  | | +-- to-absolute-glob@0.1.1 
  | | | `-- extend-shallow@2.0.1 
  | | `-- unique-stream@2.2.1 
  | |   `-- json-stable-stringify@1.0.1 
  | |     `-- jsonify@0.0.0 
  | +-- gulp-sourcemaps@1.6.0 
  | | `-- convert-source-map@1.3.0 
  | +-- is-valid-glob@0.3.0 
  | +-- lazystream@1.0.0 
  | +-- lodash.isequal@4.4.0 
  | +-- strip-bom@2.0.0 
  | | `-- is-utf8@0.2.1 
  | +-- strip-bom-stream@1.0.0 
  | | `-- first-chunk-stream@1.0.0 
  | +-- through2-filter@2.0.0 
  | `-- vali-date@1.0.0 
  +-- web-component-tester@5.0.0 
  | +-- accessibility-developer-tools@2.11.0 
  | +-- async@1.5.2 
  | +-- body-parser@1.15.2 
  | | +-- bytes@2.4.0 
  | | +-- debug@2.2.0 
  | | | `-- ms@0.7.1 
  | | +-- http-errors@1.5.1 
  | | | `-- setprototypeof@1.0.2 
  | | +-- iconv-lite@0.4.13 
  | | `-- raw-body@2.1.7 
  | +-- chai@3.5.0 
  | | +-- assertion-error@1.0.2 
  | | +-- deep-eql@0.1.3 
  | | | `-- type-detect@0.1.1 
  | | `-- type-detect@1.0.0 
  | +-- cleankill@1.0.3 
  | +-- findup-sync@0.2.1 
  | | `-- glob@4.3.5 
  | +-- glob@5.0.15 
  | | +-- inflight@1.0.6 
  | | `-- minimatch@2.0.10 
  | +-- lodash@3.10.1 
  | +-- mocha@3.2.0 
  | | +-- browser-stdout@1.3.0 
  | | +-- debug@2.2.0 
  | | | `-- ms@0.7.1 
  | | +-- diff@1.4.0 
  | | +-- glob@7.0.5 
  | | +-- growl@1.9.2 
  | | +-- json3@3.3.2 
  | | +-- lodash.create@3.1.1 
  | | | +-- lodash._baseassign@3.2.0 
  | | | +-- lodash._basecreate@3.0.3 
  | | | `-- lodash._isiterateecall@3.0.9 
  | | `-- supports-color@3.1.2 
  | |   `-- has-flag@1.0.0 
  | +-- multer@1.2.1 
  | | +-- append-field@0.1.0 
  | | +-- busboy@0.2.13 
  | | | +-- dicer@0.2.5 
  | | | | +-- readable-stream@1.1.14 
  | | | | | `-- isarray@0.0.1 
  | | | | `-- streamsearch@0.1.2 
  | | | `-- readable-stream@1.1.14 
  | | |   `-- isarray@0.0.1 
  | | +-- concat-stream@1.6.0 
  | | | `-- typedarray@0.0.6 
  | | `-- object-assign@3.0.0 
  | +-- nomnom@1.8.1 
  | | +-- chalk@0.4.0 
  | | | +-- ansi-styles@1.0.0 
  | | | +-- has-color@0.1.7 
  | | | `-- strip-ansi@0.1.1 
  | | `-- underscore@1.6.0 
  | +-- promisify-node@0.4.0 
  | | `-- nodegit-promise@4.0.0 
  | |   `-- asap@2.0.5 
  | +-- send@0.11.1 
  | | +-- debug@2.1.3 
  | | +-- depd@1.0.1 
  | | +-- destroy@1.0.3 
  | | +-- escape-html@1.0.1 
  | | +-- etag@1.5.1 
  | | +-- fresh@0.2.4 
  | | +-- mime@1.2.11 
  | | +-- ms@0.7.0 
  | | +-- on-finished@2.2.1 
  | | | `-- ee-first@1.1.0 
  | | `-- range-parser@1.0.3 
  | +-- serve-waterfall@1.1.1 
  | | `-- send@0.11.1 
  | |   +-- debug@2.1.3 
  | |   +-- depd@1.0.1 
  | |   +-- destroy@1.0.3 
  | |   +-- escape-html@1.0.1 
  | |   +-- etag@1.5.1 
  | |   +-- fresh@0.2.4 
  | |   +-- mime@1.2.11 
  | |   +-- ms@0.7.0 
  | |   +-- on-finished@2.2.1 
  | |   | `-- ee-first@1.1.0 
  | |   `-- range-parser@1.0.3 
  | +-- server-destroy@1.0.1 
  | +-- sinon@1.17.6 
  | | +-- formatio@1.1.1 
  | | +-- lolex@1.3.2 
  | | +-- samsam@1.1.2 
  | | `-- util@0.10.3 
  | |   `-- inherits@2.0.1 
  | +-- sinon-chai@2.8.0 
  | +-- socket.io@1.7.2 
  | | +-- debug@2.3.3 
  | | +-- engine.io@1.8.2 
  | | | +-- base64id@1.0.0 
  | | | +-- debug@2.3.3 
  | | | +-- engine.io-parser@1.3.2 
  | | | | +-- after@0.8.2 
  | | | | +-- arraybuffer.slice@0.0.6 
  | | | | +-- base64-arraybuffer@0.1.5 
  | | | | +-- blob@0.0.4 
  | | | | `-- wtf-8@1.0.0 
  | | | `-- ws@1.1.1 
  | | |   +-- options@0.0.6 
  | | |   `-- ultron@1.0.2 
  | | +-- has-binary@0.1.7 
  | | | `-- isarray@0.0.1 
  | | +-- socket.io-adapter@0.5.0 
  | | | `-- debug@2.3.3 
  | | +-- socket.io-client@1.7.2 
  | | | +-- backo2@1.0.2 
  | | | +-- component-bind@1.0.0 
  | | | +-- component-emitter@1.2.1 
  | | | +-- debug@2.3.3 
  | | | +-- engine.io-client@1.8.2 
  | | | | +-- component-emitter@1.2.1 
  | | | | +-- component-inherit@0.0.3 
  | | | | +-- debug@2.3.3 
  | | | | +-- has-cors@1.1.0 
  | | | | +-- parsejson@0.0.3 
  | | | | +-- parseqs@0.0.5 
  | | | | +-- xmlhttprequest-ssl@1.5.3 
  | | | | `-- yeast@0.1.2 
  | | | +-- indexof@0.0.1 
  | | | +-- object-component@0.0.3 
  | | | +-- parseuri@0.0.5 
  | | | | `-- better-assert@1.0.2 
  | | | |   `-- callsite@1.0.0 
  | | | `-- to-array@0.1.4 
  | | `-- socket.io-parser@2.3.1 
  | |   +-- component-emitter@1.1.2 
  | |   +-- debug@2.2.0 
  | |   | `-- ms@0.7.1 
  | |   `-- isarray@0.0.1 
  | +-- stacky@1.3.1 
  | | `-- lodash@3.10.1 
  | +-- test-fixture@1.1.1  (git://github.com/polymerelements/test-fixture.git#18144479b7e364cef7936672b56b4512c7e8e1d0)
  | +-- update-notifier@0.6.3 
  | | `-- boxen@0.3.1 
  | +-- wct-local@2.0.13 
  | | +-- @types/freeport@1.0.20 
  | | +-- @types/which@1.0.28 
  | | +-- freeport@1.0.5 
  | | +-- launchpad@0.5.4 
  | | | +-- async@2.1.4 
  | | | +-- browserstack@1.5.0 
  | | | +-- plist@2.0.1 
  | | | | +-- base64-js@1.1.2 
  | | | | +-- xmlbuilder@8.2.2 
  | | | | `-- xmldom@0.1.27 
  | | | +-- restify@4.3.0 
  | | | | +-- assert-plus@0.1.5 
  | | | | +-- backoff@2.5.0 
  | | | | | `-- precond@0.2.3 
  | | | | +-- bunyan@1.8.5 
  | | | | | +-- dtrace-provider@0.8.0 
  | | | | | +-- moment@2.17.1 
  | | | | | +-- mv@2.1.1 
  | | | | | | +-- ncp@2.0.0 
  | | | | | | `-- rimraf@2.4.5 
  | | | | | |   `-- glob@6.0.4 
  | | | | | `-- safe-json-stringify@1.0.3 
  | | | | +-- csv@0.4.6 
  | | | | | +-- csv-generate@0.0.6 
  | | | | | +-- csv-parse@1.1.7 
  | | | | | +-- csv-stringify@0.0.8 
  | | | | | `-- stream-transform@0.1.1 
  | | | | +-- dtrace-provider@0.6.0 
  | | | | | `-- nan@2.5.0 
  | | | | +-- escape-regexp-component@1.0.2 
  | | | | +-- formidable@1.0.17 
  | | | | +-- http-signature@0.11.0 
  | | | | | `-- asn1@0.1.11 
  | | | | +-- keep-alive-agent@0.0.1 
  | | | | +-- lru-cache@4.0.2 
  | | | | +-- node-uuid@1.4.7 
  | | | | +-- qs@6.3.0 
  | | | | +-- semver@4.3.6 
  | | | | +-- spdy@3.4.4 
  | | | | | +-- handle-thing@1.2.5 
  | | | | | +-- http-deceiver@1.2.7 
  | | | | | +-- select-hose@2.0.0 
  | | | | | `-- spdy-transport@2.0.18 
  | | | | |   +-- hpack.js@2.1.6 
  | | | | |   +-- obuf@1.1.1 
  | | | | |   `-- wbuf@1.7.2 
  | | | | |     `-- minimalistic-assert@1.0.0 
  | | | | +-- vasync@1.6.3 
  | | | | | `-- verror@1.6.0 
  | | | | |   `-- extsprintf@1.2.0 
  | | | | `-- verror@1.9.0 
  | | | |   +-- assert-plus@1.0.0 
  | | | |   `-- extsprintf@1.3.0 
  | | | `-- underscore@1.8.3 
  | | +-- selenium-standalone@5.9.0 
  | | | +-- async@1.2.1 
  | | | +-- commander@2.6.0 
  | | | +-- lodash@3.9.3 
  | | | +-- minimist@1.1.0 
  | | | +-- mkdirp@0.5.0 
  | | | | `-- minimist@0.0.8 
  | | | +-- progress@1.1.8 
  | | | +-- request@2.51.0 
  | | | | +-- aws-sign2@0.5.0 
  | | | | +-- bl@0.9.5 
  | | | | | `-- readable-stream@1.0.34 
  | | | | |   `-- isarray@0.0.1 
  | | | | +-- caseless@0.8.0 
  | | | | +-- combined-stream@0.0.7 
  | | | | | `-- delayed-stream@0.0.5 
  | | | | +-- forever-agent@0.5.2 
  | | | | +-- form-data@0.2.0 
  | | | | | +-- async@0.9.2 
  | | | | | `-- mime-types@2.0.14 
  | | | | |   `-- mime-db@1.12.0 
  | | | | +-- hawk@1.1.1 
  | | | | | +-- boom@0.4.2 
  | | | | | +-- cryptiles@0.2.2 
  | | | | | +-- hoek@0.9.1 
  | | | | | `-- sntp@0.2.4 
  | | | | +-- http-signature@0.10.1 
  | | | | | +-- asn1@0.1.11 
  | | | | | `-- assert-plus@0.1.5 
  | | | | +-- mime-types@1.0.2 
  | | | | +-- node-uuid@1.4.7 
  | | | | +-- oauth-sign@0.5.0 
  | | | | `-- qs@2.3.3 
  | | | +-- urijs@1.16.1 
  | | | +-- which@1.1.1 
  | | | | `-- is-absolute@0.1.7 
  | | | |   `-- is-relative@0.1.3 
  | | | `-- yauzl@2.7.0 
  | | |   `-- fd-slicer@1.0.1 
  | | |     `-- pend@1.2.0 
  | | `-- which@1.2.12 
  | |   `-- isexe@1.1.2 
  | +-- wct-sauce@1.8.6 
  | | +-- lodash@3.10.1 
  | | +-- sauce-connect-launcher@1.1.1 
  | | | +-- adm-zip@0.4.7 
  | | | `-- async@2.1.4 
  | | `-- uuid@2.0.3 
  | `-- wd@0.3.12 
  |   +-- archiver@0.14.4 
  |   | +-- async@0.9.2 
  |   | +-- buffer-crc32@0.2.13 
  |   | +-- glob@4.3.5 
  |   | | `-- minimatch@2.0.10 
  |   | +-- lazystream@0.1.0 
  |   | +-- lodash@3.2.0 
  |   | +-- readable-stream@1.0.34 
  |   | | `-- isarray@0.0.1 
  |   | +-- tar-stream@1.1.5 
  |   | | `-- bl@0.9.5 
  |   | `-- zip-stream@0.5.2 
  |   |   +-- compress-commons@0.2.9 
  |   |   | +-- crc32-stream@0.3.4 
  |   |   | | `-- readable-stream@1.0.34 
  |   |   | |   `-- isarray@0.0.1 
  |   |   | +-- node-int64@0.3.3 
  |   |   | `-- readable-stream@1.0.34 
  |   |   |   `-- isarray@0.0.1 
  |   |   +-- lodash@3.2.0 
  |   |   `-- readable-stream@1.0.34 
  |   |     `-- isarray@0.0.1 
  |   +-- async@1.0.0 
  |   +-- lodash@3.9.3 
  |   +-- q@1.4.1 
  |   +-- request@2.55.0 
  |   | +-- aws-sign2@0.5.0 
  |   | +-- bl@0.9.5 
  |   | | `-- readable-stream@1.0.34 
  |   | |   `-- isarray@0.0.1 
  |   | +-- caseless@0.9.0 
  |   | +-- combined-stream@0.0.7 
  |   | | `-- delayed-stream@0.0.5 
  |   | +-- form-data@0.2.0 
  |   | | `-- async@0.9.2 
  |   | +-- har-validator@1.8.0 
  |   | | `-- bluebird@2.11.0 
  |   | +-- hawk@2.3.1 
  |   | +-- http-signature@0.10.1 
  |   | | +-- asn1@0.1.11 
  |   | | +-- assert-plus@0.1.5 
  |   | | `-- ctype@0.5.3 
  |   | +-- mime-types@2.0.14 
  |   | | `-- mime-db@1.12.0 
  |   | +-- node-uuid@1.4.7 
  |   | +-- oauth-sign@0.6.0 
  |   | `-- qs@2.4.2 
  |   `-- vargs@0.1.0 
  +-- yeoman-environment@1.6.6 
  | +-- debug@2.5.1 
  | | `-- ms@0.7.2 
  | +-- diff@2.2.3 
  | +-- globby@4.1.0 
  | | `-- glob@6.0.4 
  | +-- grouped-queue@0.3.3 
  | +-- log-symbols@1.0.2 
  | +-- mem-fs@1.1.3 
  | | `-- vinyl-file@2.0.0 
  | |   `-- strip-bom-stream@2.0.0 
  | |     `-- first-chunk-stream@2.0.0 
  | +-- text-table@0.2.0 
  | `-- untildify@2.1.0 
  `-- yeoman-generator@0.23.4 
    +-- async@1.5.2 
    +-- class-extend@0.1.2 
    | `-- object-assign@2.1.1 
    +-- cli-table@0.3.1 
    +-- cross-spawn@3.0.1 
    | `-- lru-cache@4.0.2 
    |   +-- pseudomap@1.0.2 
    |   `-- yallist@2.0.0 
    +-- dargs@4.1.0 
    | `-- number-is-nan@1.0.1 
    +-- dateformat@1.0.12 
    | `-- get-stdin@4.0.1 
    +-- detect-conflict@1.0.1 
    +-- download@4.4.3 
    | +-- caw@1.2.0 
    | | +-- get-proxy@1.1.0 
    | | `-- object-assign@3.0.0 
    | +-- each-async@1.1.1 
    | | `-- set-immediate-shim@1.0.1 
    | +-- filenamify@1.2.1 
    | | +-- filename-reserved-regex@1.0.0 
    | | +-- strip-outer@1.0.0 
    | | `-- trim-repeated@1.0.0 
    | +-- got@5.7.1 
    | | +-- create-error-class@3.0.2 
    | | | `-- capture-stack-trace@1.0.0 
    | | +-- duplexer2@0.1.4 
    | | +-- is-redirect@1.0.0 
    | | +-- is-retry-allowed@1.1.0 
    | | +-- lowercase-keys@1.0.0 
    | | +-- node-status-codes@1.0.0 
    | | +-- parse-json@2.2.0 
    | | | `-- error-ex@1.3.0 
    | | |   `-- is-arrayish@0.2.1 
    | | +-- timed-out@3.1.0 
    | | +-- unzip-response@1.0.2 
    | | `-- url-parse-lax@1.0.0 
    | |   `-- prepend-http@1.0.4 
    | +-- gulp-decompress@1.2.0 
    | | +-- archive-type@3.2.0 
    | | | `-- file-type@3.9.0 
    | | `-- decompress@3.0.0 
    | |   +-- buffer-to-vinyl@1.1.0 
    | |   | `-- uuid@2.0.3 
    | |   +-- decompress-tar@3.1.0 
    | |   | +-- is-tar@1.0.0 
    | |   | +-- object-assign@2.1.1 
    | |   | +-- strip-dirs@1.1.1 
    | |   | | +-- is-absolute@0.1.7 
    | |   | | | `-- is-relative@0.1.3 
    | |   | | +-- is-natural-number@2.1.1 
    | |   | | +-- minimist@1.2.0 
    | |   | | `-- sum-up@1.0.3 
    | |   | +-- through2@0.6.5 
    | |   | | `-- readable-stream@1.0.34 
    | |   | |   `-- isarray@0.0.1 
    | |   | `-- vinyl@0.4.6 
    | |   |   `-- clone@0.2.0 
    | |   +-- decompress-tarbz2@3.1.0 
    | |   | +-- is-bzip2@1.0.0 
    | |   | +-- object-assign@2.1.1 
    | |   | +-- seek-bzip@1.0.5 
    | |   | | `-- commander@2.8.1 
    | |   | +-- through2@0.6.5 
    | |   | | `-- readable-stream@1.0.34 
    | |   | |   `-- isarray@0.0.1 
    | |   | `-- vinyl@0.4.6 
    | |   |   `-- clone@0.2.0 
    | |   +-- decompress-targz@3.1.0 
    | |   | +-- object-assign@2.1.1 
    | |   | +-- through2@0.6.5 
    | |   | | `-- readable-stream@1.0.34 
    | |   | |   `-- isarray@0.0.1 
    | |   | `-- vinyl@0.4.6 
    | |   |   `-- clone@0.2.0 
    | |   +-- decompress-unzip@3.4.0 
    | |   | +-- is-zip@1.0.0 
    | |   | `-- stat-mode@0.2.2 
    | |   `-- vinyl-assign@1.2.1 
    | +-- gulp-rename@1.2.2 
    | +-- is-url@1.2.2 
    | +-- read-all-stream@3.1.0 
    | +-- stream-combiner2@1.1.1 
    | | `-- duplexer2@0.1.4 
    | `-- ware@1.3.0 
    |   `-- wrap-fn@0.1.5 
    |     `-- co@3.1.0 
    +-- error@7.0.2 
    | `-- string-template@0.2.1 
    +-- find-up@1.1.2 
    +-- github-username@2.1.0 
    | `-- gh-got@2.4.0 
    +-- glob@7.1.1 
    | `-- fs.realpath@1.0.0 
    +-- gruntfile-editor@1.2.1 
    | `-- ast-query@2.0.0 
    |   +-- escodegen-wallaby@1.6.8 
    |   | +-- estraverse@1.9.3 
    |   | +-- esutils@2.0.2 
    |   | `-- source-map@0.2.0 
    |   `-- traverse@0.6.6 
    +-- html-wiring@1.2.0 
    | +-- cheerio@0.19.0 
    | | +-- css-select@1.0.0 
    | | | +-- boolbase@1.0.0 
    | | | +-- css-what@1.0.0 
    | | | +-- domutils@1.4.3 
    | | | `-- nth-check@1.0.1 
    | | +-- dom-serializer@0.1.0 
    | | | `-- domelementtype@1.1.3 
    | | +-- entities@1.1.1 
    | | +-- htmlparser2@3.8.3 
    | | | +-- domelementtype@1.3.0 
    | | | +-- domhandler@2.3.0 
    | | | +-- domutils@1.5.1 
    | | | +-- entities@1.0.0 
    | | | `-- readable-stream@1.1.14 
    | | |   `-- isarray@0.0.1 
    | | `-- lodash@3.10.1 
    | `-- detect-newline@1.0.3 
    |   `-- minimist@1.2.0 
    +-- istextorbinary@1.0.2 
    | +-- binaryextensions@1.0.1 
    | `-- textextensions@1.0.2 
    +-- mem-fs-editor@2.3.0 
    | +-- commondir@1.0.1 
    | +-- ejs@2.5.5 
    | +-- globby@4.1.0 
    | | `-- glob@6.0.4 
    | `-- multimatch@2.1.0 
    +-- nopt@3.0.6 
    | `-- abbrev@1.0.9 
    +-- path-exists@2.1.0 
    +-- pretty-bytes@3.0.1 
    +-- read-chunk@1.0.1 
    +-- read-pkg-up@1.0.1 
    | `-- read-pkg@1.1.0 
    |   +-- load-json-file@1.1.0 
    |   `-- path-type@1.1.0 
    +-- shelljs@0.7.5 
    | `-- rechoir@0.6.2 
    +-- underscore.string@3.0.3 
    +-- user-home@2.0.0 
    +-- yeoman-assert@2.2.2 
    | `-- lodash@3.10.1 
    +-- yeoman-test@1.6.0 
    | `-- yeoman-generator@0.24.1 
    |   +-- async@2.1.4 
    |   +-- cross-spawn@4.0.2 
    |   | `-- lru-cache@4.0.2 
    |   +-- istextorbinary@2.1.0 
    |   | `-- editions@1.3.3 
    |   `-- user-home@2.0.0 
    `-- yeoman-welcome@1.0.1 

npm ERR! Darwin 16.3.0
npm ERR! argv "/usr/local/bin/node" "/usr/local/bin/npm" "install" "-g" "polymer-cli"
npm ERR! node v7.2.1
npm ERR! npm  v4.0.5
npm ERR! path /usr/local/lib/node_modules
npm ERR! code EACCES
npm ERR! errno -13
npm ERR! syscall access

npm ERR! Error: EACCES: permission denied, access '/usr/local/lib/node_modules'
npm ERR!  { Error: EACCES: permission denied, access '/usr/local/lib/node_modules'
npm ERR!   errno: -13,
npm ERR!   code: 'EACCES',
npm ERR!   syscall: 'access',
npm ERR!   path: '/usr/local/lib/node_modules' }
npm ERR! 
npm ERR! Please try running this command again as root/Administrator.

npm ERR! Please include the following file with any support request:
npm ERR!     /Users/juk/npm-debug.log
Juks-MacBook-Pro:~ juk$ sudo npm install -g polymer-cli
Password:
npm WARN deprecated minimatch@2.0.10: Please update to minimatch 3.0.2 or higher to avoid a RegExp DoS issue
npm WARN deprecated minimatch@0.2.14: Please update to minimatch 3.0.2 or higher to avoid a RegExp DoS issue
npm WARN deprecated graceful-fs@1.2.3: graceful-fs v3.0.0 and before will fail on node releases >= v7.0. Please update to graceful-fs@^4.0.0 as soon as possible. Use 'npm ls graceful-fs' to find it in the tree.
npm WARN deprecated node-uuid@1.4.7: use uuid module instead
/usr/local/bin/polymer -> /usr/local/lib/node_modules/polymer-cli/bin/polymer.js

> dtrace-provider@0.8.0 install /usr/local/lib/node_modules/polymer-cli/node_modules/bunyan/node_modules/dtrace-provider
> node scripts/install.js


> dtrace-provider@0.6.0 install /usr/local/lib/node_modules/polymer-cli/node_modules/dtrace-provider
> node scripts/install.js


> spawn-sync@1.0.15 postinstall /usr/local/lib/node_modules/polymer-cli/node_modules/spawn-sync
> node postinstall


> sauce-connect-launcher@1.1.1 postinstall /usr/local/lib/node_modules/polymer-cli/node_modules/sauce-connect-launcher
> node scripts/install.js || nodejs scripts/install.js


> wct-sauce@1.8.6 postinstall /usr/local/lib/node_modules/polymer-cli/node_modules/wct-sauce
> node scripts/postinstall.js

Prefetching the Sauce Connect binary.
Missing Sauce Connect local proxy, downloading dependency
This will only happen once.
Downloading 5.53MB
Archive checksum verified.
Unzipping /usr/local/lib/node_modules/polymer-cli/node_modules/sauce-connect-launcher/sc/sc-4.4.2-osx.zip
Removing /usr/local/lib/node_modules/polymer-cli/node_modules/sauce-connect-launcher/sc/sc-4.4.2-osx.zip
Sauce Connect downloaded correctly

> wct-local@2.0.13 postinstall /usr/local/lib/node_modules/polymer-cli/node_modules/wct-local
> node scripts/postinstall.js

----------
selenium-standalone installation starting
----------

---
selenium install:
from: https://selenium-release.storage.googleapis.com/3.0/selenium-server-standalone-3.0.1.jar
to: /usr/local/lib/node_modules/polymer-cli/node_modules/selenium-standalone/.selenium/selenium-server/3.0.1-server.jar
---
chrome install:
from: https://chromedriver.storage.googleapis.com/2.25/chromedriver_mac64.zip
to: /usr/local/lib/node_modules/polymer-cli/node_modules/selenium-standalone/.selenium/chromedriver/2.25-x64-chromedriver
---
firefox install:
from: https://github.com/mozilla/geckodriver/releases/download/v0.11.1/geckodriver-v0.11.1-macos.tar.gz
to: /usr/local/lib/node_modules/polymer-cli/node_modules/selenium-standalone/.selenium/geckodriver/0.11.1-x64-geckodriver


-----
selenium-standalone installation finished
-----
/usr/local/lib
`-- polymer-cli@0.17.0 
  +-- @types/chalk@0.4.31 
  +-- @types/del@2.2.31 
  | `-- @types/glob@5.0.30 
  +-- @types/findup-sync@0.3.29 
  | `-- @types/minimatch@2.0.29 
  +-- @types/fs-extra@0.0.34 
  +-- @types/gulp-if@0.0.30 
  +-- @types/html-minifier@1.1.30 
  | +-- @types/clean-css@3.4.30 
  | `-- @types/relateurl@0.2.28 
  +-- @types/inquirer@0.0.30 
  | +-- @types/rx@2.5.34 
  | `-- @types/through@0.0.28 
  +-- @types/merge-stream@1.0.28 
  +-- @types/node@6.0.53 
  +-- @types/parse5@2.2.34 
  +-- @types/request@0.0.32 
  | `-- @types/form-data@0.0.33 
  +-- @types/temp@0.8.29 
  +-- @types/uglify-js@2.6.28 
  | `-- @types/source-map@0.5.0 
  +-- @types/vinyl@1.2.30 
  +-- @types/vinyl-fs@0.0.28 
  | `-- @types/glob-stream@3.1.30 
  +-- @types/yeoman-generator@0.0.30 
  +-- chalk@1.1.3 
  | +-- ansi-styles@2.2.1 
  | +-- escape-string-regexp@1.0.5 
  | +-- has-ansi@2.0.0 
  | | `-- ansi-regex@2.0.0 
  | +-- strip-ansi@3.0.1 
  | `-- supports-color@2.0.0 
  +-- command-line-args@3.0.5 
  | +-- array-back@1.0.4 
  | +-- feature-detect-es6@1.3.1 
  | +-- find-replace@1.0.2 
  | | `-- test-value@2.1.0 
  | `-- typical@2.6.0 
  +-- command-line-commands@1.0.4 
  +-- command-line-usage@3.0.8 
  | +-- ansi-escape-sequences@3.0.0 
  | `-- table-layout@0.3.0 
  |   +-- core-js@2.4.1 
  |   +-- deep-extend@0.4.1 
  |   `-- wordwrapjs@2.0.0 
  |     `-- reduce-flatten@1.0.1 
  +-- css-slam@1.2.1 
  | +-- dom5@1.3.6 
  | | +-- @types/node@4.0.30 
  | | +-- @types/parse5@0.0.31 
  | | | `-- @types/node@6.0.53 
  | | `-- parse5@1.5.1 
  | `-- shady-css-parser@0.0.8 
  +-- del@2.2.2 
  | +-- globby@5.0.0 
  | | +-- array-union@1.0.2 
  | | | `-- array-uniq@1.0.3 
  | | `-- arrify@1.0.1 
  | +-- is-path-cwd@1.0.0 
  | +-- is-path-in-cwd@1.0.0 
  | | `-- is-path-inside@1.0.0 
  | |   `-- path-is-inside@1.0.2 
  | +-- object-assign@4.1.0 
  | +-- pify@2.3.0 
  | +-- pinkie-promise@2.0.1 
  | | `-- pinkie@2.0.4 
  | `-- rimraf@2.5.4 
  +-- dom5@2.0.1 
  | +-- @types/clone@0.1.30 
  | +-- @types/node@4.0.30 
  | `-- clone@1.0.2 
  +-- findup-sync@0.4.3 
  | +-- detect-file@0.1.0 
  | | `-- fs-exists-sync@0.1.0 
  | +-- is-glob@2.0.1 
  | | `-- is-extglob@1.0.0 
  | +-- micromatch@2.3.11 
  | | +-- arr-diff@2.0.0 
  | | | `-- arr-flatten@1.0.1 
  | | +-- array-unique@0.2.1 
  | | +-- braces@1.8.5 
  | | | +-- expand-range@1.8.2 
  | | | | `-- fill-range@2.2.3 
  | | | |   +-- is-number@2.1.0 
  | | | |   +-- isobject@2.1.0 
  | | | |   +-- randomatic@1.1.6 
  | | | |   `-- repeat-string@1.6.1 
  | | | +-- preserve@0.2.0 
  | | | `-- repeat-element@1.1.2 
  | | +-- expand-brackets@0.1.5 
  | | | `-- is-posix-bracket@0.1.1 
  | | +-- extglob@0.3.2 
  | | +-- filename-regex@2.0.0 
  | | +-- kind-of@3.1.0 
  | | | `-- is-buffer@1.1.4 
  | | +-- normalize-path@2.0.1 
  | | +-- object.omit@2.0.1 
  | | | +-- for-own@0.1.4 
  | | | | `-- for-in@0.1.6 
  | | | `-- is-extendable@0.1.1 
  | | +-- parse-glob@3.0.4 
  | | | +-- glob-base@0.3.0 
  | | | | `-- glob-parent@2.0.0 
  | | | `-- is-dotfile@1.0.2 
  | | `-- regex-cache@0.4.3 
  | |   +-- is-equal-shallow@0.1.3 
  | |   `-- is-primitive@2.0.0 
  | `-- resolve-dir@0.1.1 
  |   +-- expand-tilde@1.2.2 
  |   `-- global-modules@0.2.3 
  |     +-- global-prefix@0.1.5 
  |     | +-- homedir-polyfill@1.0.1 
  |     | | `-- parse-passwd@1.0.0 
  |     | `-- ini@1.3.4 
  |     `-- is-windows@0.2.0 
  +-- fs-extra@1.0.0 
  | +-- graceful-fs@4.1.11 
  | +-- jsonfile@2.4.0 
  | `-- klaw@1.3.1 
  +-- github@6.1.0 
  | +-- follow-redirects@0.0.7 
  | | `-- stream-consume@0.1.0 
  | +-- https-proxy-agent@1.0.0 
  | | `-- agent-base@2.0.1 
  | `-- mime@1.3.4 
  +-- gulp-if@2.0.2 
  | +-- gulp-match@1.0.3 
  | | `-- minimatch@3.0.3 
  | |   `-- brace-expansion@1.1.6 
  | |     +-- balanced-match@0.4.2 
  | |     `-- concat-map@0.0.1 
  | +-- ternary-stream@2.0.1 
  | | `-- fork-stream@0.0.4 
  | `-- through2@2.0.3 
  |   `-- xtend@4.0.1 
  +-- gunzip-maybe@1.3.1 
  | +-- browserify-zlib@0.1.4 
  | | `-- pako@0.2.9 
  | +-- is-deflate@1.0.0 
  | +-- is-gzip@1.0.0 
  | +-- peek-stream@1.1.1 
  | | `-- through2@0.5.1 
  | |   +-- readable-stream@1.0.34 
  | |   | `-- isarray@0.0.1 
  | |   `-- xtend@3.0.0 
  | +-- pumpify@1.3.5 
  | | `-- inherits@2.0.3 
  | `-- through2@0.4.2 
  |   +-- readable-stream@1.0.34 
  |   | `-- isarray@0.0.1 
  |   `-- xtend@2.1.2 
  |     `-- object-keys@0.4.0 
  +-- html-minifier@3.2.3 
  | +-- camel-case@3.0.0 
  | | +-- no-case@2.3.0 
  | | | `-- lower-case@1.1.3 
  | | `-- upper-case@1.1.3 
  | +-- clean-css@3.4.23 
  | | +-- commander@2.8.1 
  | | `-- source-map@0.4.4 
  | |   `-- amdefine@1.0.1 
  | +-- commander@2.9.0 
  | | `-- graceful-readlink@1.0.1 
  | +-- he@1.1.0 
  | +-- ncname@1.0.0 
  | | `-- xml-char-classes@1.0.0 
  | +-- param-case@2.1.0 
  | `-- relateurl@0.2.7 
  +-- inquirer@1.2.3 
  | +-- ansi-escapes@1.4.0 
  | +-- cli-cursor@1.0.2 
  | | `-- restore-cursor@1.0.1 
  | |   +-- exit-hook@1.1.1 
  | |   `-- onetime@1.1.0 
  | +-- cli-width@2.1.0 
  | +-- external-editor@1.1.1 
  | | +-- spawn-sync@1.0.15 
  | | | `-- os-shim@0.1.3 
  | | `-- tmp@0.0.29 
  | +-- figures@1.7.0 
  | +-- lodash@4.17.3 
  | +-- mute-stream@0.0.6 
  | +-- run-async@2.3.0 
  | | `-- is-promise@2.1.0 
  | +-- rx@4.1.0 
  | +-- string-width@1.0.2 
  | | +-- code-point-at@1.1.0 
  | | `-- is-fullwidth-code-point@1.0.0 
  | `-- through@2.3.8 
  +-- merge-stream@1.0.1 
  | `-- readable-stream@2.2.2 
  |   +-- buffer-shims@1.0.0 
  |   +-- core-util-is@1.0.2 
  |   +-- isarray@1.0.0 
  |   +-- process-nextick-args@1.0.7 
  |   +-- string_decoder@0.10.31 
  |   `-- util-deprecate@1.0.2 
  +-- parse5@2.2.3 
  +-- plylog@0.4.0 
  | `-- winston@2.3.0 
  |   +-- async@1.0.0 
  |   +-- colors@1.0.3 
  |   +-- cycle@1.0.3 
  |   +-- eyes@0.1.8 
  |   `-- stack-trace@0.0.9 
  +-- polylint@2.10.1 
  | +-- colors@1.1.2 
  | +-- command-line-args@2.1.6 
  | | `-- command-line-usage@2.0.5 
  | |   +-- ansi-escape-sequences@2.2.2 
  | |   | `-- collect-all@0.2.1 
  | |   |   `-- stream-via@0.1.1 
  | |   +-- column-layout@2.1.4 
  | |   | +-- ansi-escape-sequences@2.2.2 
  | |   | +-- collect-json@1.0.8 
  | |   | | +-- collect-all@1.0.2 
  | |   | | +-- stream-connect@1.0.2 
  | |   | | `-- stream-via@1.0.3 
  | |   | +-- command-line-args@2.1.6 
  | |   | | `-- command-line-usage@2.0.5 
  | |   | +-- object-tools@2.0.6 
  | |   | | +-- object-get@2.1.0 
  | |   | | `-- test-value@1.1.0 
  | |   | `-- wordwrapjs@1.2.1 
  | |   `-- wordwrapjs@1.2.1 
  | +-- dom5@1.3.6 
  | | +-- @types/node@4.0.30 
  | | +-- @types/parse5@0.0.31 
  | | | `-- @types/node@6.0.53 
  | | `-- parse5@1.5.1 
  | +-- es6-set@0.1.4 
  | | +-- d@0.1.1 
  | | +-- es5-ext@0.10.12 
  | | +-- es6-iterator@2.0.0 
  | | +-- es6-symbol@3.1.0 
  | | `-- event-emitter@0.3.4 
  | +-- hydrolysis@1.24.1 
  | | +-- babel-polyfill@6.20.0 
  | | | +-- babel-runtime@6.20.0 
  | | | `-- regenerator-runtime@0.10.1 
  | | +-- doctrine@0.7.2 
  | | | +-- esutils@1.1.6 
  | | | `-- isarray@0.0.1 
  | | +-- dom5@1.3.6 
  | | | +-- @types/node@4.0.30 
  | | | +-- @types/parse5@0.0.31 
  | | | | `-- @types/node@6.0.53 
  | | | `-- parse5@1.5.1 
  | | +-- escodegen@1.8.1 
  | | | +-- esprima@2.7.3 
  | | | +-- estraverse@1.9.3 
  | | | +-- esutils@2.0.2 
  | | | +-- optionator@0.8.2 
  | | | | +-- deep-is@0.1.3 
  | | | | +-- fast-levenshtein@2.0.5 
  | | | | +-- levn@0.3.0 
  | | | | +-- prelude-ls@1.1.2 
  | | | | +-- type-check@0.3.2 
  | | | | `-- wordwrap@1.0.0 
  | | | `-- source-map@0.2.0 
  | | +-- espree@3.3.2 
  | | | +-- acorn@4.0.4 
  | | | `-- acorn-jsx@3.0.1 
  | | |   `-- acorn@3.3.0 
  | | `-- estraverse@3.1.0 
  | +-- optimist@0.6.1 
  | | +-- minimist@0.0.10 
  | | `-- wordwrap@0.0.2 
  | `-- path-is-absolute@1.0.1 
  +-- polymer-build@0.4.1 
  | +-- dom5@1.3.6 
  | | +-- @types/node@4.0.30 
  | | +-- @types/parse5@0.0.31 
  | | | `-- @types/node@6.0.53 
  | | `-- parse5@1.5.1 
  | +-- fs-extra@0.30.0 
  | +-- gulp@3.9.1 
  | | +-- archy@1.0.0 
  | | +-- deprecated@0.0.1 
  | | +-- gulp-util@3.0.7 
  | | | +-- array-differ@1.0.0 
  | | | +-- beeper@1.1.1 
  | | | +-- fancy-log@1.2.0 
  | | | | `-- time-stamp@1.0.1 
  | | | +-- gulplog@1.0.0 
  | | | | `-- glogg@1.0.0 
  | | | +-- has-gulplog@0.1.0 
  | | | | `-- sparkles@1.0.0 
  | | | +-- lodash._reescape@3.0.0 
  | | | +-- lodash._reevaluate@3.0.0 
  | | | +-- lodash._reinterpolate@3.0.0 
  | | | +-- lodash.template@3.6.2 
  | | | | +-- lodash._basecopy@3.0.1 
  | | | | +-- lodash._basetostring@3.0.1 
  | | | | +-- lodash._basevalues@3.0.0 
  | | | | +-- lodash.escape@3.2.0 
  | | | | | `-- lodash._root@3.0.1 
  | | | | +-- lodash.keys@3.1.2 
  | | | | | +-- lodash._getnative@3.9.1 
  | | | | | +-- lodash.isarguments@3.1.0 
  | | | | | `-- lodash.isarray@3.0.4 
  | | | | +-- lodash.restparam@3.6.1 
  | | | | `-- lodash.templatesettings@3.1.1 
  | | | +-- minimist@1.2.0 
  | | | +-- multipipe@0.1.2 
  | | | | `-- duplexer2@0.0.2 
  | | | |   `-- readable-stream@1.1.14 
  | | | |     `-- isarray@0.0.1 
  | | | +-- object-assign@3.0.0 
  | | | `-- vinyl@0.5.3 
  | | +-- interpret@1.0.1 
  | | +-- liftoff@2.3.0 
  | | | +-- fined@1.0.2 
  | | | | +-- lodash.assignwith@4.2.0 
  | | | | +-- lodash.isempty@4.4.0 
  | | | | +-- lodash.pick@4.4.0 
  | | | | `-- parse-filepath@1.0.1 
  | | | |   +-- is-absolute@0.2.6 
  | | | |   | `-- is-relative@0.2.1 
  | | | |   |   `-- is-unc-path@0.1.2 
  | | | |   |     `-- unc-path-regex@0.1.2 
  | | | |   +-- map-cache@0.2.2 
  | | | |   `-- path-root@0.1.1 
  | | | |     `-- path-root-regex@0.1.2 
  | | | +-- flagged-respawn@0.3.2 
  | | | +-- lodash.isplainobject@4.0.6 
  | | | +-- lodash.isstring@4.0.1 
  | | | `-- lodash.mapvalues@4.6.0 
  | | +-- minimist@1.2.0 
  | | +-- orchestrator@0.3.8 
  | | | +-- end-of-stream@0.1.5 
  | | | | `-- once@1.3.3 
  | | | `-- sequencify@0.0.7 
  | | +-- pretty-hrtime@1.0.3 
  | | +-- semver@4.3.6 
  | | +-- tildify@1.2.0 
  | | +-- v8flags@2.0.11 
  | | | `-- user-home@1.1.1 
  | | `-- vinyl-fs@0.3.14 
  | |   +-- defaults@1.0.3 
  | |   +-- glob-stream@3.1.18 
  | |   | +-- glob@4.5.3 
  | |   | +-- glob2base@0.0.12 
  | |   | | `-- find-index@0.1.1 
  | |   | +-- minimatch@2.0.10 
  | |   | +-- ordered-read-streams@0.1.0 
  | |   | +-- through2@0.6.5 
  | |   | | `-- readable-stream@1.0.34 
  | |   | |   `-- isarray@0.0.1 
  | |   | `-- unique-stream@1.0.0 
  | |   +-- glob-watcher@0.0.6 
  | |   | `-- gaze@0.5.2 
  | |   |   `-- globule@0.1.0 
  | |   |     +-- glob@3.1.21 
  | |   |     | +-- graceful-fs@1.2.3 
  | |   |     | `-- inherits@1.0.2 
  | |   |     +-- lodash@1.0.2 
  | |   |     `-- minimatch@0.2.14 
  | |   |       +-- lru-cache@2.7.3 
  | |   |       `-- sigmund@1.0.1 
  | |   +-- graceful-fs@3.0.11 
  | |   | `-- natives@1.1.0 
  | |   +-- strip-bom@1.0.0 
  | |   +-- through2@0.6.5 
  | |   | `-- readable-stream@1.0.34 
  | |   |   `-- isarray@0.0.1 
  | |   `-- vinyl@0.4.6 
  | |     `-- clone@0.2.0 
  | +-- minimatch-all@1.1.0 
  | +-- multipipe@0.3.1 
  | | `-- duplexer2@0.1.4 
  | +-- sw-precache@3.2.0 
  | | +-- dom-urls@1.1.0 
  | | | `-- urijs@1.18.4 
  | | +-- es6-promise@3.3.1 
  | | +-- glob@6.0.4 
  | | +-- lodash.defaults@4.2.0 
  | | +-- lodash.template@4.4.0 
  | | | `-- lodash.templatesettings@4.1.0 
  | | +-- meow@3.7.0 
  | | | +-- camelcase-keys@2.1.0 
  | | | | `-- camelcase@2.1.1 
  | | | +-- loud-rejection@1.6.0 
  | | | | +-- currently-unhandled@0.4.1 
  | | | | | `-- array-find-index@1.0.2 
  | | | | `-- signal-exit@3.0.2 
  | | | +-- map-obj@1.0.1 
  | | | +-- minimist@1.2.0 
  | | | +-- normalize-package-data@2.3.5 
  | | | | +-- hosted-git-info@2.1.5 
  | | | | +-- is-builtin-module@1.0.0 
  | | | | | `-- builtin-modules@1.1.1 
  | | | | `-- validate-npm-package-license@3.0.1 
  | | | |   +-- spdx-correct@1.0.2 
  | | | |   | `-- spdx-license-ids@1.2.2 
  | | | |   `-- spdx-expression-parse@1.0.4 
  | | | +-- redent@1.0.0 
  | | | | +-- indent-string@2.1.0 
  | | | | `-- strip-indent@1.0.1 
  | | | `-- trim-newlines@1.0.0 
  | | `-- sw-toolbox@3.4.0 
  | |   +-- path-to-regexp@1.7.0 
  | |   | `-- isarray@0.0.1 
  | |   `-- serviceworker-cache-polyfill@4.0.0 
  | `-- vulcanize@1.15.2 
  |   +-- dom5@1.3.6 
  |   | +-- @types/node@4.0.30 
  |   | +-- @types/parse5@0.0.31 
  |   | | `-- @types/node@6.0.53 
  |   | `-- parse5@1.5.1 
  |   +-- es6-promise@2.3.0 
  |   `-- path-posix@1.0.0 
  +-- polyserve@0.13.0 
  | +-- @types/express@4.0.34 
  | | `-- @types/express-serve-static-core@4.0.39 
  | +-- @types/mime@0.0.29 
  | +-- @types/node@4.0.30 
  | +-- @types/opn@3.0.28 
  | +-- @types/serve-static@1.7.31 
  | +-- express@4.14.0 
  | | +-- accepts@1.3.3 
  | | | `-- negotiator@0.6.1 
  | | +-- array-flatten@1.1.1 
  | | +-- content-disposition@0.5.1 
  | | +-- content-type@1.0.2 
  | | +-- cookie@0.3.1 
  | | +-- cookie-signature@1.0.6 
  | | +-- debug@2.2.0 
  | | | `-- ms@0.7.1 
  | | +-- depd@1.1.0 
  | | +-- encodeurl@1.0.1 
  | | +-- escape-html@1.0.3 
  | | +-- etag@1.7.0 
  | | +-- finalhandler@0.5.0 
  | | | +-- debug@2.2.0 
  | | | | `-- ms@0.7.1 
  | | | `-- unpipe@1.0.0 
  | | +-- fresh@0.3.0 
  | | +-- merge-descriptors@1.0.1 
  | | +-- methods@1.1.2 
  | | +-- on-finished@2.3.0 
  | | | `-- ee-first@1.1.1 
  | | +-- parseurl@1.3.1 
  | | +-- path-to-regexp@0.1.7 
  | | +-- proxy-addr@1.1.2 
  | | | +-- forwarded@0.1.0 
  | | | `-- ipaddr.js@1.1.1 
  | | +-- qs@6.2.0 
  | | +-- range-parser@1.2.0 
  | | +-- send@0.14.1 
  | | | +-- destroy@1.0.4 
  | | | `-- statuses@1.3.1 
  | | +-- serve-static@1.11.1 
  | | | `-- send@0.14.1 
  | | |   +-- debug@2.2.0 
  | | |   `-- ms@0.7.1 
  | | +-- type-is@1.6.14 
  | | | `-- media-typer@0.3.0 
  | | +-- utils-merge@1.0.0 
  | | `-- vary@1.1.0 
  | +-- find-port@1.0.1 
  | +-- opn@3.0.3 
  | `-- send@0.10.1 
  |   +-- debug@2.1.3 
  |   | `-- ms@0.7.0 
  |   +-- depd@1.0.1 
  |   +-- destroy@1.0.3 
  |   +-- escape-html@1.0.1 
  |   +-- etag@1.5.1 
  |   | `-- crc@3.2.1 
  |   +-- fresh@0.2.4 
  |   +-- mime@1.2.11 
  |   +-- ms@0.6.2 
  |   +-- on-finished@2.1.1 
  |   | `-- ee-first@1.1.0 
  |   `-- range-parser@1.0.3 
  +-- request@2.79.0 
  | +-- aws-sign2@0.6.0 
  | +-- aws4@1.5.0 
  | +-- caseless@0.11.0 
  | +-- combined-stream@1.0.5 
  | | `-- delayed-stream@1.0.0 
  | +-- extend@3.0.0 
  | +-- forever-agent@0.6.1 
  | +-- form-data@2.1.2 
  | | `-- asynckit@0.4.0 
  | +-- har-validator@2.0.6 
  | | `-- is-my-json-valid@2.15.0 
  | |   +-- generate-function@2.0.0 
  | |   +-- generate-object-property@1.2.0 
  | |   | `-- is-property@1.0.2 
  | |   `-- jsonpointer@4.0.1 
  | +-- hawk@3.1.3 
  | | +-- boom@2.10.1 
  | | +-- cryptiles@2.0.5 
  | | +-- hoek@2.16.3 
  | | `-- sntp@1.0.9 
  | +-- http-signature@1.1.1 
  | | +-- assert-plus@0.2.0 
  | | +-- jsprim@1.3.1 
  | | | +-- extsprintf@1.0.2 
  | | | +-- json-schema@0.2.3 
  | | | `-- verror@1.3.6 
  | | `-- sshpk@1.10.1 
  | |   +-- asn1@0.2.3 
  | |   +-- assert-plus@1.0.0 
  | |   +-- bcrypt-pbkdf@1.0.0 
  | |   +-- dashdash@1.14.1 
  | |   | `-- assert-plus@1.0.0 
  | |   +-- ecc-jsbn@0.1.1 
  | |   +-- getpass@0.1.6 
  | |   | `-- assert-plus@1.0.0 
  | |   +-- jodid25519@1.0.2 
  | |   +-- jsbn@0.1.0 
  | |   `-- tweetnacl@0.14.5 
  | +-- is-typedarray@1.0.0 
  | +-- isstream@0.1.2 
  | +-- json-stringify-safe@5.0.1 
  | +-- mime-types@2.1.13 
  | | `-- mime-db@1.25.0 
  | +-- oauth-sign@0.8.2 
  | +-- qs@6.3.0 
  | +-- stringstream@0.0.5 
  | +-- tough-cookie@2.3.2 
  | | `-- punycode@1.4.1 
  | +-- tunnel-agent@0.4.3 
  | `-- uuid@3.0.1 
  +-- resolve@1.2.0 
  +-- tar-fs@1.15.0 
  | +-- mkdirp@0.5.1 
  | | `-- minimist@0.0.8 
  | +-- pump@1.0.2 
  | | +-- end-of-stream@1.1.0 
  | | | `-- once@1.3.3 
  | | `-- once@1.4.0 
  | |   `-- wrappy@1.0.2 
  | `-- tar-stream@1.5.2 
  |   +-- bl@1.2.0 
  |   `-- end-of-stream@1.0.0 
  |     `-- once@1.3.3 
  +-- temp@0.8.3 
  | +-- os-tmpdir@1.0.2 
  | `-- rimraf@2.2.8 
  +-- uglify-js@2.7.5 
  | +-- async@0.2.10 
  | +-- source-map@0.5.6 
  | +-- uglify-to-browserify@1.0.2 
  | `-- yargs@3.10.0 
  |   +-- camelcase@1.2.1 
  |   +-- cliui@2.1.0 
  |   | +-- center-align@0.1.3 
  |   | | +-- align-text@0.1.4 
  |   | | | `-- longest@1.0.1 
  |   | | `-- lazy-cache@1.0.4 
  |   | `-- right-align@0.1.3 
  |   +-- decamelize@1.2.0 
  |   `-- window-size@0.1.0 
  +-- update-notifier@1.0.3 
  | +-- boxen@0.6.0 
  | | +-- ansi-align@1.1.0 
  | | +-- camelcase@2.1.1 
  | | +-- cli-boxes@1.0.0 
  | | +-- filled-array@1.1.0 
  | | +-- repeating@2.0.1 
  | | | `-- is-finite@1.0.2 
  | | `-- widest-line@1.0.0 
  | +-- configstore@2.1.0 
  | | +-- dot-prop@3.0.0 
  | | | `-- is-obj@1.0.1 
  | | +-- osenv@0.1.4 
  | | +-- uuid@2.0.3 
  | | `-- write-file-atomic@1.2.0 
  | |   +-- imurmurhash@0.1.4 
  | |   `-- slide@1.1.6 
  | +-- is-npm@1.0.0 
  | +-- latest-version@2.0.0 
  | | `-- package-json@2.4.0 
  | |   +-- registry-auth-token@3.1.0 
  | |   | `-- rc@1.1.6 
  | |   |   +-- minimist@1.2.0 
  | |   |   `-- strip-json-comments@1.0.4 
  | |   +-- registry-url@3.1.0 
  | |   `-- semver@5.3.0 
  | +-- lazy-req@1.1.0 
  | +-- semver-diff@2.1.0 
  | | `-- semver@5.0.3 
  | `-- xdg-basedir@2.0.0 
  |   `-- os-homedir@1.0.2 
  +-- vinyl@1.2.0 
  | +-- clone-stats@0.0.1 
  | `-- replace-ext@0.0.1 
  +-- vinyl-fs@2.4.4 
  | +-- duplexify@3.5.0 
  | | `-- stream-shift@1.0.0 
  | +-- glob-stream@5.3.5 
  | | +-- glob@5.0.15 
  | | +-- glob-parent@3.1.0 
  | | | +-- is-glob@3.1.0 
  | | | | `-- is-extglob@2.1.1 
  | | | `-- path-dirname@1.0.2 
  | | +-- ordered-read-streams@0.3.0 
  | | | `-- is-stream@1.1.0 
  | | +-- through2@0.6.5 
  | | | `-- readable-stream@1.0.34 
  | | |   `-- isarray@0.0.1 
  | | +-- to-absolute-glob@0.1.1 
  | | | `-- extend-shallow@2.0.1 
  | | `-- unique-stream@2.2.1 
  | |   `-- json-stable-stringify@1.0.1 
  | |     `-- jsonify@0.0.0 
  | +-- gulp-sourcemaps@1.6.0 
  | | `-- convert-source-map@1.3.0 
  | +-- is-valid-glob@0.3.0 
  | +-- lazystream@1.0.0 
  | +-- lodash.isequal@4.4.0 
  | +-- strip-bom@2.0.0 
  | | `-- is-utf8@0.2.1 
  | +-- strip-bom-stream@1.0.0 
  | | `-- first-chunk-stream@1.0.0 
  | +-- through2-filter@2.0.0 
  | `-- vali-date@1.0.0 
  +-- web-component-tester@5.0.0 
  | +-- accessibility-developer-tools@2.11.0 
  | +-- async@1.5.2 
  | +-- body-parser@1.15.2 
  | | +-- bytes@2.4.0 
  | | +-- debug@2.2.0 
  | | | `-- ms@0.7.1 
  | | +-- http-errors@1.5.1 
  | | | `-- setprototypeof@1.0.2 
  | | +-- iconv-lite@0.4.13 
  | | `-- raw-body@2.1.7 
  | +-- chai@3.5.0 
  | | +-- assertion-error@1.0.2 
  | | +-- deep-eql@0.1.3 
  | | | `-- type-detect@0.1.1 
  | | `-- type-detect@1.0.0 
  | +-- cleankill@1.0.3 
  | +-- findup-sync@0.2.1 
  | | `-- glob@4.3.5 
  | +-- glob@5.0.15 
  | | +-- inflight@1.0.6 
  | | `-- minimatch@2.0.10 
  | +-- lodash@3.10.1 
  | +-- mocha@3.2.0 
  | | +-- browser-stdout@1.3.0 
  | | +-- debug@2.2.0 
  | | | `-- ms@0.7.1 
  | | +-- diff@1.4.0 
  | | +-- glob@7.0.5 
  | | +-- growl@1.9.2 
  | | +-- json3@3.3.2 
  | | +-- lodash.create@3.1.1 
  | | | +-- lodash._baseassign@3.2.0 
  | | | +-- lodash._basecreate@3.0.3 
  | | | `-- lodash._isiterateecall@3.0.9 
  | | `-- supports-color@3.1.2 
  | |   `-- has-flag@1.0.0 
  | +-- multer@1.2.1 
  | | +-- append-field@0.1.0 
  | | +-- busboy@0.2.13 
  | | | +-- dicer@0.2.5 
  | | | | +-- readable-stream@1.1.14 
  | | | | | `-- isarray@0.0.1 
  | | | | `-- streamsearch@0.1.2 
  | | | `-- readable-stream@1.1.14 
  | | |   `-- isarray@0.0.1 
  | | +-- concat-stream@1.6.0 
  | | | `-- typedarray@0.0.6 
  | | `-- object-assign@3.0.0 
  | +-- nomnom@1.8.1 
  | | +-- chalk@0.4.0 
  | | | +-- ansi-styles@1.0.0 
  | | | +-- has-color@0.1.7 
  | | | `-- strip-ansi@0.1.1 
  | | `-- underscore@1.6.0 
  | +-- promisify-node@0.4.0 
  | | `-- nodegit-promise@4.0.0 
  | |   `-- asap@2.0.5 
  | +-- send@0.11.1 
  | | +-- debug@2.1.3 
  | | +-- depd@1.0.1 
  | | +-- destroy@1.0.3 
  | | +-- escape-html@1.0.1 
  | | +-- etag@1.5.1 
  | | +-- fresh@0.2.4 
  | | +-- mime@1.2.11 
  | | +-- ms@0.7.0 
  | | +-- on-finished@2.2.1 
  | | | `-- ee-first@1.1.0 
  | | `-- range-parser@1.0.3 
  | +-- serve-waterfall@1.1.1 
  | | `-- send@0.11.1 
  | |   +-- debug@2.1.3 
  | |   +-- depd@1.0.1 
  | |   +-- destroy@1.0.3 
  | |   +-- escape-html@1.0.1 
  | |   +-- etag@1.5.1 
  | |   +-- fresh@0.2.4 
  | |   +-- mime@1.2.11 
  | |   +-- ms@0.7.0 
  | |   +-- on-finished@2.2.1 
  | |   | `-- ee-first@1.1.0 
  | |   `-- range-parser@1.0.3 
  | +-- server-destroy@1.0.1 
  | +-- sinon@1.17.6 
  | | +-- formatio@1.1.1 
  | | +-- lolex@1.3.2 
  | | +-- samsam@1.1.2 
  | | `-- util@0.10.3 
  | |   `-- inherits@2.0.1 
  | +-- sinon-chai@2.8.0 
  | +-- socket.io@1.7.2 
  | | +-- debug@2.3.3 
  | | +-- engine.io@1.8.2 
  | | | +-- base64id@1.0.0 
  | | | +-- debug@2.3.3 
  | | | +-- engine.io-parser@1.3.2 
  | | | | +-- after@0.8.2 
  | | | | +-- arraybuffer.slice@0.0.6 
  | | | | +-- base64-arraybuffer@0.1.5 
  | | | | +-- blob@0.0.4 
  | | | | `-- wtf-8@1.0.0 
  | | | `-- ws@1.1.1 
  | | |   +-- options@0.0.6 
  | | |   `-- ultron@1.0.2 
  | | +-- has-binary@0.1.7 
  | | | `-- isarray@0.0.1 
  | | +-- socket.io-adapter@0.5.0 
  | | | `-- debug@2.3.3 
  | | +-- socket.io-client@1.7.2 
  | | | +-- backo2@1.0.2 
  | | | +-- component-bind@1.0.0 
  | | | +-- component-emitter@1.2.1 
  | | | +-- debug@2.3.3 
  | | | +-- engine.io-client@1.8.2 
  | | | | +-- component-emitter@1.2.1 
  | | | | +-- component-inherit@0.0.3 
  | | | | +-- debug@2.3.3 
  | | | | +-- has-cors@1.1.0 
  | | | | +-- parsejson@0.0.3 
  | | | | +-- parseqs@0.0.5 
  | | | | +-- xmlhttprequest-ssl@1.5.3 
  | | | | `-- yeast@0.1.2 
  | | | +-- indexof@0.0.1 
  | | | +-- object-component@0.0.3 
  | | | +-- parseuri@0.0.5 
  | | | | `-- better-assert@1.0.2 
  | | | |   `-- callsite@1.0.0 
  | | | `-- to-array@0.1.4 
  | | `-- socket.io-parser@2.3.1 
  | |   +-- component-emitter@1.1.2 
  | |   +-- debug@2.2.0 
  | |   | `-- ms@0.7.1 
  | |   `-- isarray@0.0.1 
  | +-- stacky@1.3.1 
  | | `-- lodash@3.10.1 
  | +-- test-fixture@1.1.1  (git://github.com/polymerelements/test-fixture.git#18144479b7e364cef7936672b56b4512c7e8e1d0)
  | +-- update-notifier@0.6.3 
  | | `-- boxen@0.3.1 
  | +-- wct-local@2.0.13 
  | | +-- @types/freeport@1.0.20 
  | | +-- @types/which@1.0.28 
  | | +-- freeport@1.0.5 
  | | +-- launchpad@0.5.4 
  | | | +-- async@2.1.4 
  | | | +-- browserstack@1.5.0 
  | | | +-- plist@2.0.1 
  | | | | +-- base64-js@1.1.2 
  | | | | +-- xmlbuilder@8.2.2 
  | | | | `-- xmldom@0.1.27 
  | | | +-- restify@4.3.0 
  | | | | +-- assert-plus@0.1.5 
  | | | | +-- backoff@2.5.0 
  | | | | | `-- precond@0.2.3 
  | | | | +-- bunyan@1.8.5 
  | | | | | +-- dtrace-provider@0.8.0 
  | | | | | +-- moment@2.17.1 
  | | | | | +-- mv@2.1.1 
  | | | | | | +-- ncp@2.0.0 
  | | | | | | `-- rimraf@2.4.5 
  | | | | | |   `-- glob@6.0.4 
  | | | | | `-- safe-json-stringify@1.0.3 
  | | | | +-- csv@0.4.6 
  | | | | | +-- csv-generate@0.0.6 
  | | | | | +-- csv-parse@1.1.7 
  | | | | | +-- csv-stringify@0.0.8 
  | | | | | `-- stream-transform@0.1.1 
  | | | | +-- dtrace-provider@0.6.0 
  | | | | | `-- nan@2.5.0 
  | | | | +-- escape-regexp-component@1.0.2 
  | | | | +-- formidable@1.0.17 
  | | | | +-- http-signature@0.11.0 
  | | | | | `-- asn1@0.1.11 
  | | | | +-- keep-alive-agent@0.0.1 
  | | | | +-- lru-cache@4.0.2 
  | | | | +-- node-uuid@1.4.7 
  | | | | +-- qs@6.3.0 
  | | | | +-- semver@4.3.6 
  | | | | +-- spdy@3.4.4 
  | | | | | +-- handle-thing@1.2.5 
  | | | | | +-- http-deceiver@1.2.7 
  | | | | | +-- select-hose@2.0.0 
  | | | | | `-- spdy-transport@2.0.18 
  | | | | |   +-- hpack.js@2.1.6 
  | | | | |   +-- obuf@1.1.1 
  | | | | |   `-- wbuf@1.7.2 
  | | | | |     `-- minimalistic-assert@1.0.0 
  | | | | +-- vasync@1.6.3 
  | | | | | `-- verror@1.6.0 
  | | | | |   `-- extsprintf@1.2.0 
  | | | | `-- verror@1.9.0 
  | | | |   +-- assert-plus@1.0.0 
  | | | |   `-- extsprintf@1.3.0 
  | | | `-- underscore@1.8.3 
  | | +-- selenium-standalone@5.9.0 
  | | | +-- async@1.2.1 
  | | | +-- commander@2.6.0 
  | | | +-- lodash@3.9.3 
  | | | +-- minimist@1.1.0 
  | | | +-- mkdirp@0.5.0 
  | | | | `-- minimist@0.0.8 
  | | | +-- progress@1.1.8 
  | | | +-- request@2.51.0 
  | | | | +-- aws-sign2@0.5.0 
  | | | | +-- bl@0.9.5 
  | | | | | `-- readable-stream@1.0.34 
  | | | | |   `-- isarray@0.0.1 
  | | | | +-- caseless@0.8.0 
  | | | | +-- combined-stream@0.0.7 
  | | | | | `-- delayed-stream@0.0.5 
  | | | | +-- forever-agent@0.5.2 
  | | | | +-- form-data@0.2.0 
  | | | | | +-- async@0.9.2 
  | | | | | `-- mime-types@2.0.14 
  | | | | |   `-- mime-db@1.12.0 
  | | | | +-- hawk@1.1.1 
  | | | | | +-- boom@0.4.2 
  | | | | | +-- cryptiles@0.2.2 
  | | | | | +-- hoek@0.9.1 
  | | | | | `-- sntp@0.2.4 
  | | | | +-- http-signature@0.10.1 
  | | | | | +-- asn1@0.1.11 
  | | | | | `-- assert-plus@0.1.5 
  | | | | +-- mime-types@1.0.2 
  | | | | +-- node-uuid@1.4.7 
  | | | | +-- oauth-sign@0.5.0 
  | | | | `-- qs@2.3.3 
  | | | +-- urijs@1.16.1 
  | | | +-- which@1.1.1 
  | | | | `-- is-absolute@0.1.7 
  | | | |   `-- is-relative@0.1.3 
  | | | `-- yauzl@2.7.0 
  | | |   `-- fd-slicer@1.0.1 
  | | |     `-- pend@1.2.0 
  | | `-- which@1.2.12 
  | |   `-- isexe@1.1.2 
  | +-- wct-sauce@1.8.6 
  | | +-- lodash@3.10.1 
  | | +-- sauce-connect-launcher@1.1.1 
  | | | +-- adm-zip@0.4.7 
  | | | `-- async@2.1.4 
  | | `-- uuid@2.0.3 
  | `-- wd@0.3.12 
  |   +-- archiver@0.14.4 
  |   | +-- async@0.9.2 
  |   | +-- buffer-crc32@0.2.13 
  |   | +-- glob@4.3.5 
  |   | | `-- minimatch@2.0.10 
  |   | +-- lazystream@0.1.0 
  |   | +-- lodash@3.2.0 
  |   | +-- readable-stream@1.0.34 
  |   | | `-- isarray@0.0.1 
  |   | +-- tar-stream@1.1.5 
  |   | | `-- bl@0.9.5 
  |   | `-- zip-stream@0.5.2 
  |   |   +-- compress-commons@0.2.9 
  |   |   | +-- crc32-stream@0.3.4 
  |   |   | | `-- readable-stream@1.0.34 
  |   |   | |   `-- isarray@0.0.1 
  |   |   | +-- node-int64@0.3.3 
  |   |   | `-- readable-stream@1.0.34 
  |   |   |   `-- isarray@0.0.1 
  |   |   +-- lodash@3.2.0 
  |   |   `-- readable-stream@1.0.34 
  |   |     `-- isarray@0.0.1 
  |   +-- async@1.0.0 
  |   +-- lodash@3.9.3 
  |   +-- q@1.4.1 
  |   +-- request@2.55.0 
  |   | +-- aws-sign2@0.5.0 
  |   | +-- bl@0.9.5 
  |   | | `-- readable-stream@1.0.34 
  |   | |   `-- isarray@0.0.1 
  |   | +-- caseless@0.9.0 
  |   | +-- combined-stream@0.0.7 
  |   | | `-- delayed-stream@0.0.5 
  |   | +-- form-data@0.2.0 
  |   | | `-- async@0.9.2 
  |   | +-- har-validator@1.8.0 
  |   | | `-- bluebird@2.11.0 
  |   | +-- hawk@2.3.1 
  |   | +-- http-signature@0.10.1 
  |   | | +-- asn1@0.1.11 
  |   | | +-- assert-plus@0.1.5 
  |   | | `-- ctype@0.5.3 
  |   | +-- mime-types@2.0.14 
  |   | | `-- mime-db@1.12.0 
  |   | +-- node-uuid@1.4.7 
  |   | +-- oauth-sign@0.6.0 
  |   | `-- qs@2.4.2 
  |   `-- vargs@0.1.0 
  +-- yeoman-environment@1.6.6 
  | +-- debug@2.5.1 
  | | `-- ms@0.7.2 
  | +-- diff@2.2.3 
  | +-- globby@4.1.0 
  | | `-- glob@6.0.4 
  | +-- grouped-queue@0.3.3 
  | +-- log-symbols@1.0.2 
  | +-- mem-fs@1.1.3 
  | | `-- vinyl-file@2.0.0 
  | |   `-- strip-bom-stream@2.0.0 
  | |     `-- first-chunk-stream@2.0.0 
  | +-- text-table@0.2.0 
  | `-- untildify@2.1.0 
  `-- yeoman-generator@0.23.4 
    +-- async@1.5.2 
    +-- class-extend@0.1.2 
    | `-- object-assign@2.1.1 
    +-- cli-table@0.3.1 
    +-- cross-spawn@3.0.1 
    | `-- lru-cache@4.0.2 
    |   +-- pseudomap@1.0.2 
    |   `-- yallist@2.0.0 
    +-- dargs@4.1.0 
    | `-- number-is-nan@1.0.1 
    +-- dateformat@1.0.12 
    | `-- get-stdin@4.0.1 
    +-- detect-conflict@1.0.1 
    +-- download@4.4.3 
    | +-- caw@1.2.0 
    | | +-- get-proxy@1.1.0 
    | | `-- object-assign@3.0.0 
    | +-- each-async@1.1.1 
    | | `-- set-immediate-shim@1.0.1 
    | +-- filenamify@1.2.1 
    | | +-- filename-reserved-regex@1.0.0 
    | | +-- strip-outer@1.0.0 
    | | `-- trim-repeated@1.0.0 
    | +-- got@5.7.1 
    | | +-- create-error-class@3.0.2 
    | | | `-- capture-stack-trace@1.0.0 
    | | +-- duplexer2@0.1.4 
    | | +-- is-redirect@1.0.0 
    | | +-- is-retry-allowed@1.1.0 
    | | +-- lowercase-keys@1.0.0 
    | | +-- node-status-codes@1.0.0 
    | | +-- parse-json@2.2.0 
    | | | `-- error-ex@1.3.0 
    | | |   `-- is-arrayish@0.2.1 
    | | +-- timed-out@3.1.0 
    | | +-- unzip-response@1.0.2 
    | | `-- url-parse-lax@1.0.0 
    | |   `-- prepend-http@1.0.4 
    | +-- gulp-decompress@1.2.0 
    | | +-- archive-type@3.2.0 
    | | | `-- file-type@3.9.0 
    | | `-- decompress@3.0.0 
    | |   +-- buffer-to-vinyl@1.1.0 
    | |   | `-- uuid@2.0.3 
    | |   +-- decompress-tar@3.1.0 
    | |   | +-- is-tar@1.0.0 
    | |   | +-- object-assign@2.1.1 
    | |   | +-- strip-dirs@1.1.1 
    | |   | | +-- is-absolute@0.1.7 
    | |   | | | `-- is-relative@0.1.3 
    | |   | | +-- is-natural-number@2.1.1 
    | |   | | +-- minimist@1.2.0 
    | |   | | `-- sum-up@1.0.3 
    | |   | +-- through2@0.6.5 
    | |   | | `-- readable-stream@1.0.34 
    | |   | |   `-- isarray@0.0.1 
    | |   | `-- vinyl@0.4.6 
    | |   |   `-- clone@0.2.0 
    | |   +-- decompress-tarbz2@3.1.0 
    | |   | +-- is-bzip2@1.0.0 
    | |   | +-- object-assign@2.1.1 
    | |   | +-- seek-bzip@1.0.5 
    | |   | | `-- commander@2.8.1 
    | |   | +-- through2@0.6.5 
    | |   | | `-- readable-stream@1.0.34 
    | |   | |   `-- isarray@0.0.1 
    | |   | `-- vinyl@0.4.6 
    | |   |   `-- clone@0.2.0 
    | |   +-- decompress-targz@3.1.0 
    | |   | +-- object-assign@2.1.1 
    | |   | +-- through2@0.6.5 
    | |   | | `-- readable-stream@1.0.34 
    | |   | |   `-- isarray@0.0.1 
    | |   | `-- vinyl@0.4.6 
    | |   |   `-- clone@0.2.0 
    | |   +-- decompress-unzip@3.4.0 
    | |   | +-- is-zip@1.0.0 
    | |   | `-- stat-mode@0.2.2 
    | |   `-- vinyl-assign@1.2.1 
    | +-- gulp-rename@1.2.2 
    | +-- is-url@1.2.2 
    | +-- read-all-stream@3.1.0 
    | +-- stream-combiner2@1.1.1 
    | | `-- duplexer2@0.1.4 
    | `-- ware@1.3.0 
    |   `-- wrap-fn@0.1.5 
    |     `-- co@3.1.0 
    +-- error@7.0.2 
    | `-- string-template@0.2.1 
    +-- find-up@1.1.2 
    +-- github-username@2.1.0 
    | `-- gh-got@2.4.0 
    +-- glob@7.1.1 
    | `-- fs.realpath@1.0.0 
    +-- gruntfile-editor@1.2.1 
    | `-- ast-query@2.0.0 
    |   +-- escodegen-wallaby@1.6.8 
    |   | +-- estraverse@1.9.3 
    |   | +-- esutils@2.0.2 
    |   | `-- source-map@0.2.0 
    |   `-- traverse@0.6.6 
    +-- html-wiring@1.2.0 
    | +-- cheerio@0.19.0 
    | | +-- css-select@1.0.0 
    | | | +-- boolbase@1.0.0 
    | | | +-- css-what@1.0.0 
    | | | +-- domutils@1.4.3 
    | | | `-- nth-check@1.0.1 
    | | +-- dom-serializer@0.1.0 
    | | | `-- domelementtype@1.1.3 
    | | +-- entities@1.1.1 
    | | +-- htmlparser2@3.8.3 
    | | | +-- domelementtype@1.3.0 
    | | | +-- domhandler@2.3.0 
    | | | +-- domutils@1.5.1 
    | | | +-- entities@1.0.0 
    | | | `-- readable-stream@1.1.14 
    | | |   `-- isarray@0.0.1 
    | | `-- lodash@3.10.1 
    | `-- detect-newline@1.0.3 
    |   `-- minimist@1.2.0 
    +-- istextorbinary@1.0.2 
    | +-- binaryextensions@1.0.1 
    | `-- textextensions@1.0.2 
    +-- mem-fs-editor@2.3.0 
    | +-- commondir@1.0.1 
    | +-- ejs@2.5.5 
    | +-- globby@4.1.0 
    | | `-- glob@6.0.4 
    | `-- multimatch@2.1.0 
    +-- nopt@3.0.6 
    | `-- abbrev@1.0.9 
    +-- path-exists@2.1.0 
    +-- pretty-bytes@3.0.1 
    +-- read-chunk@1.0.1 
    +-- read-pkg-up@1.0.1 
    | `-- read-pkg@1.1.0 
    |   +-- load-json-file@1.1.0 
    |   `-- path-type@1.1.0 
    +-- shelljs@0.7.5 
    | `-- rechoir@0.6.2 
    +-- underscore.string@3.0.3 
    +-- user-home@2.0.0 
    +-- yeoman-assert@2.2.2 
    | `-- lodash@3.10.1 
    +-- yeoman-test@1.6.0 
    | `-- yeoman-generator@0.24.1 
    |   +-- async@2.1.4 
    |   +-- cross-spawn@4.0.2 
    |   | `-- lru-cache@4.0.2 
    |   +-- istextorbinary@2.1.0 
    |   | `-- editions@1.3.3 
    |   `-- user-home@2.0.0 
    `-- yeoman-welcome@1.0.1 

Juks-MacBook-Pro:~ juk$ ssh juk@printexpress.cloud
juk@printexpress.cloud's password: 
Last login: Fri Dec 23 21:05:58 2016 from ppp-115-87-66-105.revip4.asianet.co.th
[juk@printexpress ~]$ su
Password: 
[root@printexpress juk]# yum update
Loaded plugins: fastestmirror
Repository nodesource is listed more than once in the configuration
Repository nodesource-source is listed more than once in the configuration
Loading mirror speeds from cached hostfile
 * base: mirror.0x.sg
 * elrepo: elrepo.mirror.angkasa.id
 * epel: epel.mirror.angkasa.id
 * extras: mirror.0x.sg
 * remi-safe: remi.check-update.co.uk
 * updates: mirror.0x.sg
No packages marked for update
[root@printexpress juk]# yum upgrade
Loaded plugins: fastestmirror
Repository nodesource is listed more than once in the configuration
Repository nodesource-source is listed more than once in the configuration
Loading mirror speeds from cached hostfile
 * base: mirror.0x.sg
 * elrepo: elrepo.mirror.angkasa.id
 * epel: epel.mirror.angkasa.id
 * extras: mirror.0x.sg
 * remi-safe: remi.check-update.co.uk
 * updates: mirror.0x.sg
No packages marked for update
[root@printexpress juk]# uname -r
4.9.0-1.el7.elrepo.x86_64
[root@printexpress juk]# vi /etc/sysctl.conf

# System default settings live in /usr/lib/sysctl.d/00-system.conf.
# To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
#
# For more information, see sysctl.conf(5) and sysctl.d(5).

# Vultr: Accept IPv6 advertisements when forwarding is enabled
net.core.somaxconn = 1024
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 3240000

# Avoid a smurf attack
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Turn on protection for bad icmp error messages
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Turn on syncookies for SYN flood attack protection
net.ipv4.tcp_syncookies = 1


# Disable ipv6 for security
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1

# Allows the server to close the connection after a client stops responding.
reset_timedout_connection on;
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
~                                                                                                                       
"/etc/sysctl.conf" 30L, 982C
  [Restored Dec 25, 2559 BE, 17:06:52]
Last login: Sun Dec 25 16:15:34 on ttys000
Restored session: Sun Dec 25 17:06:45 ICT 2016
Juks-MacBook-Pro:~ juk$ ssh juk@printexpress.cloud
juk@printexpress.cloud's password: 
Last login: Sun Dec 25 16:55:50 2016 from ppp-115-87-66-105.revip4.asianet.co.th
[juk@printexpress ~]$ sudo su
[sudo] password for juk: 
[root@printexpress juk]# cd /etc/
[root@printexpress etc]# rm -rf .
./                ../               .java/            .pwd.lock         .sysctl.conf.swp  .updated
[root@printexpress etc]# rm -rf .sysctl.conf.swp 
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# vi nginx/nginx.conf 
[root@printexpress etc]# systemctl restart nginx
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 1024
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 3240000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
sysctl: cannot stat /proc/sys/kernel/exec-shield: No such file or directory
kernel.randomize_va_space = 1
net.ipv6.conf.default.router_solicitations = 0
net.ipv6.conf.default.accept_ra_rtr_pref = 0
net.ipv6.conf.default.accept_ra_pinfo = 0
net.ipv6.conf.default.accept_ra_defrtr = 0
net.ipv6.conf.default.autoconf = 0
net.ipv6.conf.default.dad_transmits = 0
net.ipv6.conf.default.max_addresses = 1
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.ip_local_port_range = 2000 65000
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 87380 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_window_scaling = 1
sysctl: /etc/sysctl.conf(88): invalid syntax, continuing...
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -t
sysctl: invalid option -- 't'

Usage:
 sysctl [options] [variable[=value] ...]

Options:
  -a, --all            display all variables
  -A                   alias of -a
  -X                   alias of -a
      --deprecated     include deprecated parameters to listing
  -b, --binary         print value without new line
  -e, --ignore         ignore unknown variables errors
  -N, --names          print variable names without values
  -n, --values         print only values of a variables
  -p, --load[=<file>]  read values from file
  -f                   alias of -p
      --system         read values from all system directories
  -r, --pattern <expression>
                       select setting that match expression
  -q, --quiet          do not echo variable set
  -w, --write          enable writing a value to variable
  -o                   does nothing
  -x                   does nothing
  -d                   alias of -h

 -h, --help     display this help and exit
 -V, --version  output version information and exit

For more details see sysctl(8).
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 1024
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 3240000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
sysctl: cannot stat /proc/sys/kernel/exec-shield: No such file or directory
kernel.randomize_va_space = 1
net.ipv6.conf.default.router_solicitations = 0
net.ipv6.conf.default.accept_ra_rtr_pref = 0
net.ipv6.conf.default.accept_ra_pinfo = 0
net.ipv6.conf.default.accept_ra_defrtr = 0
net.ipv6.conf.default.autoconf = 0
net.ipv6.conf.default.dad_transmits = 0
net.ipv6.conf.default.max_addresses = 1
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.ip_local_port_range = 2000 65000
sysctl: /etc/sysctl.conf(88): invalid syntax, continuing...
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 1024
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
sysctl: cannot stat /proc/sys/kernel/exec-shield: No such file or directory
kernel.randomize_va_space = 1
net.ipv6.conf.default.router_solicitations = 0
net.ipv6.conf.default.accept_ra_rtr_pref = 0
net.ipv6.conf.default.accept_ra_pinfo = 0
net.ipv6.conf.default.accept_ra_defrtr = 0
net.ipv6.conf.default.autoconf = 0
net.ipv6.conf.default.dad_transmits = 0
net.ipv6.conf.default.max_addresses = 1
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.ip_local_port_range = 2000 65000
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 1024
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 320000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.ip_local_port_range = 2000 65000
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 87380 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_window_scaling = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 1024
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 320000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.ip_local_port_range = 2000 65000
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 87380 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_window_scaling = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# yum install util-linux
Loaded plugins: fastestmirror
Repository nodesource is listed more than once in the configuration
Repository nodesource-source is listed more than once in the configuration
Loading mirror speeds from cached hostfile
 * base: mirror.0x.sg
 * elrepo: elrepo.mirror.angkasa.id
 * epel: epel.mirror.angkasa.id
 * extras: mirror.0x.sg
 * remi-safe: remi.check-update.co.uk
 * updates: mirror.0x.sg
Package util-linux-2.23.2-33.el7.x86_64 already installed and latest version
Nothing to do
[root@printexpress etc]# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                1
On-line CPU(s) list:   0
Thread(s) per core:    1
Core(s) per socket:    1
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 61
Model name:            Virtual CPU a7769a6388d5
Stepping:              2
CPU MHz:               2399.996
BogoMIPS:              4799.99
Hypervisor vendor:     KVM
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              4096K
NUMA node0 CPU(s):     0
[root@printexpress etc]# tuned-adm active
Current active profile: latency-performance
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p /etc/sysctl.conf
net.core.somaxconn = 50000
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
sysctl: cannot stat /proc/sys/kernel/exec-shield: No such file or directory
kernel.randomize_va_space = 1
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 87380 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.netdev_max_backlog = 5000
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
sysctl: /etc/sysctl.conf(76): invalid syntax, continuing...
[root@printexpress etc]# sysctl -a | grep -i shield
sysctl: reading key "net.ipv6.conf.all.stable_secret"
sysctl: reading key "net.ipv6.conf.default.stable_secret"
sysctl: reading key "net.ipv6.conf.eth0.stable_secret"
sysctl: reading key "net.ipv6.conf.eth1.stable_secret"
sysctl: reading key "net.ipv6.conf.lo.stable_secret"
[root@printexpress etc]# dmesg | grep ‘[NX|DX]*protection’
bash: $'DX]*protection\342\200\231': command not found
grep: Unmatched [ or [^
[root@printexpress etc]# ls /proc/sys/kernel/
Display all 109 possibilities? (y or n)
acct                               numa_balancing_scan_period_min_ms  sched_min_granularity_ns
acpi_video_flags                   numa_balancing_scan_size_mb        sched_nr_migrate
auto_msgmni                        osrelease                          sched_rr_timeslice_ms
bootloader_type                    ostype                             sched_rt_period_us
bootloader_version                 overflowgid                        sched_rt_runtime_us
cad_pid                            overflowuid                        sched_schedstats
cap_last_cap                       panic                              sched_shares_window_ns
core_pattern                       panic_on_io_nmi                    sched_time_avg_ms
core_pipe_limit                    panic_on_oops                      sched_tunable_scaling
core_uses_pid                      panic_on_rcu_stall                 sched_wakeup_granularity_ns
ctrl-alt-del                       panic_on_stackoverflow             sem
dmesg_restrict                     panic_on_unrecovered_nmi           sem_next_id
domainname                         panic_on_warn                      shmall
ftrace_dump_on_oops                perf_cpu_time_max_percent          shmmax
ftrace_enabled                     perf_event_max_contexts_per_stack  shmmni
hardlockup_all_cpu_backtrace       perf_event_max_sample_rate         shm_next_id
hardlockup_panic                   perf_event_max_stack               shm_rmid_forced
hostname                           perf_event_mlock_kb                softlockup_all_cpu_backtrace
hotplug                            perf_event_paranoid                softlockup_panic
io_delay_type                      pid_max                            soft_watchdog
kexec_load_disabled                poweroff_cmd                       stack_tracer_enabled
keys/                              print-fatal-signals                sysctl_writes_strict
kptr_restrict                      printk                             sysrq
kstack_depth_to_print              printk_delay                       tainted
max_lock_depth                     printk_devkmsg                     threads-max
modprobe                           printk_ratelimit                   timer_migration
modules_disabled                   printk_ratelimit_burst             traceoff_on_warning
msgmax                             pty/                               tracepoint_printk
msgmnb                             random/                            unknown_nmi_panic
msgmni                             randomize_va_space                 unprivileged_bpf_disabled
msg_next_id                        real-root-dev                      usermodehelper/
ngroups_max                        sched_autogroup_enabled            version
nmi_watchdog                       sched_cfs_bandwidth_slice_us       watchdog
ns_last_pid                        sched_child_runs_first             watchdog_cpumask
numa_balancing                     sched_domain/                      watchdog_thresh
numa_balancing_scan_delay_ms       sched_latency_ns                   
numa_balancing_scan_period_max_ms  sched_migration_cost_ns            
[root@printexpress etc]# ls /proc/sys/kernel/random
random/             randomize_va_space  
[root@printexpress etc]# ls /proc/sys/kernel/random
random/             randomize_va_space  
[root@printexpress etc]# vi /proc/sys/kernel/randomize_va_space 
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 4096
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_timestamps = 1
net.ipv4.ip_local_port_range = 2000 65535
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
kernel.randomize_va_space = 1
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 87380 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
sysctl: /etc/sysctl.conf(86): invalid syntax, continuing...
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 4096
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_timestamps = 1
net.ipv4.ip_local_port_range = 2000 65535
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 87380 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
sysctl: /etc/sysctl.conf(86): invalid syntax, continuing...
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 4096
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_timestamps = 1
net.ipv4.ip_local_port_range = 2000 65535
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
kernel.randomize_va_space = 1
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.tcp_rmem = 4096 87380 8388608
net.ipv4.tcp_wmem = 4096 87380 8388608
net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 8192
net.core.optmem_max = 25165824
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_timestamps = 1
net.ipv4.ip_local_port_range = 2000 65535
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rfc1337 = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
kernel.randomize_va_space = 2
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
[root@printexpress etc]# vi nginx/nginx.conf 
[root@printexpress etc]# vi sysc
sysconfig/          sysctl.conf         sysctl.conf.rpmnew  sysctl.d/           
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -p
net.core.somaxconn = 16384
net.core.optmem_max = 25165824
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_timestamps = 1
net.ipv4.ip_local_port_range = 2000 65535
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rfc1337 = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
kernel.randomize_va_space = 2
fs.file-max = 65535
kernel.pid_max = 65536
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 87380 16777216
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
[root@printexpress etc]# vi sysctl.conf
[root@printexpress etc]# sysctl -A
abi.vsyscall32 = 1
debug.exception-trace = 1
debug.kprobes-optimization = 1
dev.cdrom.autoclose = 1
dev.cdrom.autoeject = 0
dev.cdrom.check_media = 0
dev.cdrom.debug = 0
dev.cdrom.info = CD-ROM information, Id: cdrom.c 3.20 2003/12/17
dev.cdrom.info = 
dev.cdrom.info = drive name:		sr0
dev.cdrom.info = drive speed:		4
dev.cdrom.info = drive # of slots:	1
dev.cdrom.info = Can close tray:		1
dev.cdrom.info = Can open tray:		1
dev.cdrom.info = Can lock tray:		1
dev.cdrom.info = Can change speed:	1
dev.cdrom.info = Can select disk:	0
dev.cdrom.info = Can read multisession:	1
dev.cdrom.info = Can read MCN:		1
dev.cdrom.info = Reports media changed:	1
dev.cdrom.info = Can play audio:		1
dev.cdrom.info = Can write CD-R:		0
dev.cdrom.info = Can write CD-RW:	0
dev.cdrom.info = Can read DVD:		1
dev.cdrom.info = Can write DVD-R:	0
dev.cdrom.info = Can write DVD-RAM:	0
dev.cdrom.info = Can read MRW:		1
dev.cdrom.info = Can write MRW:		1
dev.cdrom.info = Can write RAM:		1
dev.cdrom.info = 
dev.cdrom.info = 
dev.cdrom.lock = 1
dev.hpet.max-user-freq = 64
dev.mac_hid.mouse_button2_keycode = 97
dev.mac_hid.mouse_button3_keycode = 100
dev.mac_hid.mouse_button_emulation = 0
dev.parport.default.spintime = 500
dev.parport.default.timeslice = 200
dev.raid.speed_limit_max = 200000
dev.raid.speed_limit_min = 1000
dev.scsi.logging_level = 0
fs.aio-max-nr = 65536
fs.aio-nr = 128
fs.binfmt_misc.jexec = enabled
fs.binfmt_misc.jexec = interpreter /usr/java/default/lib/jexec
fs.binfmt_misc.jexec = flags: 
fs.binfmt_misc.jexec = offset 0
fs.binfmt_misc.jexec = magic 504b0304
fs.binfmt_misc.status = enabled
fs.dentry-state = 329376	319668	45	0	0	0
fs.dir-notify-enable = 1
fs.epoll.max_user_watches = 203837
fs.file-max = 65535
fs.file-nr = 1536	0	65535
fs.inode-nr = 20463	2236
fs.inode-state = 20463	2236	0	0	0	0	0
fs.inotify.max_queued_events = 16384
fs.inotify.max_user_instances = 128
fs.inotify.max_user_watches = 8192
fs.lease-break-time = 45
fs.leases-enable = 1
fs.mount-max = 100000
fs.mqueue.msg_default = 10
fs.mqueue.msg_max = 10
fs.mqueue.msgsize_default = 8192
fs.mqueue.msgsize_max = 8192
fs.mqueue.queues_max = 256
fs.nr_open = 1048576
fs.overflowgid = 65534
fs.overflowuid = 65534
fs.pipe-max-size = 1048576
fs.pipe-user-pages-hard = 0
fs.pipe-user-pages-soft = 16384
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.quota.allocated_dquots = 0
fs.quota.cache_hits = 0
fs.quota.drops = 0
fs.quota.free_dquots = 0
fs.quota.lookups = 0
fs.quota.reads = 0
fs.quota.syncs = 2
fs.quota.warnings = 1
fs.quota.writes = 0
fs.suid_dumpable = 0
kernel.acct = 4	2	30
kernel.acpi_video_flags = 0
kernel.auto_msgmni = 0
kernel.bootloader_type = 114
kernel.bootloader_version = 2
kernel.cad_pid = 1
kernel.cap_last_cap = 37
kernel.core_pattern = core
kernel.core_pipe_limit = 0
kernel.core_uses_pid = 1
kernel.ctrl-alt-del = 0
kernel.dmesg_restrict = 0
kernel.domainname = (none)
kernel.ftrace_dump_on_oops = 0
kernel.ftrace_enabled = 1
kernel.hardlockup_all_cpu_backtrace = 0
kernel.hardlockup_panic = 1
kernel.hostname = printexpress.cloud
kernel.hotplug = 
kernel.io_delay_type = 0
kernel.kexec_load_disabled = 0
kernel.keys.gc_delay = 300
kernel.keys.maxbytes = 20000
kernel.keys.maxkeys = 200
kernel.keys.persistent_keyring_expiry = 259200
kernel.keys.root_maxbytes = 25000000
kernel.keys.root_maxkeys = 1000000
kernel.kptr_restrict = 0
kernel.kstack_depth_to_print = 12
kernel.max_lock_depth = 1024
kernel.modprobe = /sbin/modprobe
kernel.modules_disabled = 0
kernel.msg_next_id = -1
kernel.msgmax = 8192
kernel.msgmnb = 16384
kernel.msgmni = 32000
kernel.ngroups_max = 65536
kernel.nmi_watchdog = 0
kernel.ns_last_pid = 26882
kernel.numa_balancing = 0
kernel.numa_balancing_scan_delay_ms = 1000
kernel.numa_balancing_scan_period_max_ms = 60000
kernel.numa_balancing_scan_period_min_ms = 1000
kernel.numa_balancing_scan_size_mb = 256
kernel.osrelease = 4.9.0-1.el7.elrepo.x86_64
kernel.ostype = Linux
kernel.overflowgid = 65534
kernel.overflowuid = 65534
kernel.panic = 0
kernel.panic_on_io_nmi = 0
kernel.panic_on_oops = 1
kernel.panic_on_rcu_stall = 0
kernel.panic_on_stackoverflow = 0
kernel.panic_on_unrecovered_nmi = 0
kernel.panic_on_warn = 0
kernel.perf_cpu_time_max_percent = 25
kernel.perf_event_max_contexts_per_stack = 8
kernel.perf_event_max_sample_rate = 100000
kernel.perf_event_max_stack = 127
kernel.perf_event_mlock_kb = 516
kernel.perf_event_paranoid = 2
kernel.pid_max = 65536
kernel.poweroff_cmd = /sbin/poweroff
kernel.print-fatal-signals = 0
kernel.printk = 4	4	1	7
kernel.printk_delay = 0
kernel.printk_devkmsg = ratelimit
kernel.printk_ratelimit = 5
kernel.printk_ratelimit_burst = 10
kernel.pty.max = 4096
kernel.pty.nr = 2
kernel.pty.reserve = 1024
kernel.random.boot_id = adde8606-bca7-49b1-a94f-55ce03f8b142
kernel.random.entropy_avail = 1178
kernel.random.poolsize = 4096
kernel.random.read_wakeup_threshold = 64
kernel.random.urandom_min_reseed_secs = 60
kernel.random.uuid = a197e6ac-a8f1-4e99-ab34-d88b457bf600
kernel.random.write_wakeup_threshold = 896
kernel.randomize_va_space = 2
kernel.real-root-dev = 0
kernel.sched_autogroup_enabled = 1
kernel.sched_cfs_bandwidth_slice_us = 5000
kernel.sched_child_runs_first = 0
kernel.sched_latency_ns = 6000000
kernel.sched_migration_cost_ns = 5000000
kernel.sched_min_granularity_ns = 10000000
kernel.sched_nr_migrate = 32
kernel.sched_rr_timeslice_ms = 100
kernel.sched_rt_period_us = 1000000
kernel.sched_rt_runtime_us = 950000
kernel.sched_schedstats = 0
kernel.sched_shares_window_ns = 10000000
kernel.sched_time_avg_ms = 1000
kernel.sched_tunable_scaling = 1
kernel.sched_wakeup_granularity_ns = 1000000
kernel.sem = 32000	1024000000	500	32000
kernel.sem_next_id = -1
kernel.shm_next_id = -1
kernel.shm_rmid_forced = 0
kernel.shmall = 18446744073692774399
kernel.shmmax = 18446744073692774399
kernel.shmmni = 4096
kernel.soft_watchdog = 1
kernel.softlockup_all_cpu_backtrace = 0
kernel.softlockup_panic = 0
kernel.stack_tracer_enabled = 0
kernel.sysctl_writes_strict = 1
kernel.sysrq = 16
kernel.tainted = 0
kernel.threads-max = 7775
kernel.timer_migration = 1
kernel.traceoff_on_warning = 0
kernel.tracepoint_printk = 0
kernel.unknown_nmi_panic = 0
kernel.unprivileged_bpf_disabled = 0
kernel.usermodehelper.bset = 4294967295	63
kernel.usermodehelper.inheritable = 4294967295	63
kernel.version = #1 SMP Sun Dec 11 15:43:54 EST 2016
kernel.watchdog = 1
kernel.watchdog_cpumask = 0
kernel.watchdog_thresh = 10
net.bridge.bridge-nf-call-arptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-filter-pppoe-tagged = 0
net.bridge.bridge-nf-filter-vlan-tagged = 0
net.bridge.bridge-nf-pass-vlan-input-dev = 0
net.core.bpf_jit_enable = 0
net.core.bpf_jit_harden = 0
net.core.busy_poll = 0
net.core.busy_read = 0
net.core.default_qdisc = pfifo_fast
net.core.dev_weight = 64
net.core.flow_limit_cpu_bitmap = 0
net.core.flow_limit_table_len = 4096
net.core.max_skb_frags = 17
net.core.message_burst = 10
net.core.message_cost = 5
net.core.netdev_budget = 300
net.core.netdev_max_backlog = 65536
net.core.netdev_rss_key = 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
net.core.netdev_tstamp_prequeue = 1
net.core.optmem_max = 25165824
net.core.rmem_default = 212992
net.core.rmem_max = 16777216
net.core.rps_sock_flow_entries = 0
net.core.somaxconn = 16384
net.core.tstamp_allow_data = 1
net.core.warnings = 0
net.core.wmem_default = 212992
net.core.wmem_max = 16777216
net.core.xfrm_acq_expires = 30
net.core.xfrm_aevent_etime = 10
net.core.xfrm_aevent_rseqth = 2
net.core.xfrm_larval_drop = 1
net.ipv4.cipso_cache_bucket_size = 10
net.ipv4.cipso_cache_enable = 1
net.ipv4.cipso_rbm_optfmt = 0
net.ipv4.cipso_rbm_strictvalid = 1
net.ipv4.conf.all.accept_local = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.arp_accept = 0
net.ipv4.conf.all.arp_announce = 0
net.ipv4.conf.all.arp_filter = 0
net.ipv4.conf.all.arp_ignore = 0
net.ipv4.conf.all.arp_notify = 0
net.ipv4.conf.all.bootp_relay = 0
net.ipv4.conf.all.disable_policy = 0
net.ipv4.conf.all.disable_xfrm = 0
net.ipv4.conf.all.drop_gratuitous_arp = 0
net.ipv4.conf.all.drop_unicast_in_l2_multicast = 0
net.ipv4.conf.all.force_igmp_version = 0
net.ipv4.conf.all.forwarding = 0
net.ipv4.conf.all.igmpv2_unsolicited_report_interval = 10000
net.ipv4.conf.all.igmpv3_unsolicited_report_interval = 1000
net.ipv4.conf.all.ignore_routes_with_linkdown = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.all.mc_forwarding = 0
net.ipv4.conf.all.medium_id = 0
net.ipv4.conf.all.promote_secondaries = 1
net.ipv4.conf.all.proxy_arp = 0
net.ipv4.conf.all.proxy_arp_pvlan = 0
net.ipv4.conf.all.route_localnet = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.shared_media = 1
net.ipv4.conf.all.src_valid_mark = 0
net.ipv4.conf.all.tag = 0
net.ipv4.conf.default.accept_local = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.arp_accept = 0
net.ipv4.conf.default.arp_announce = 0
net.ipv4.conf.default.arp_filter = 0
net.ipv4.conf.default.arp_ignore = 0
net.ipv4.conf.default.arp_notify = 0
net.ipv4.conf.default.bootp_relay = 0
net.ipv4.conf.default.disable_policy = 0
net.ipv4.conf.default.disable_xfrm = 0
net.ipv4.conf.default.drop_gratuitous_arp = 0
net.ipv4.conf.default.drop_unicast_in_l2_multicast = 0
net.ipv4.conf.default.force_igmp_version = 0
net.ipv4.conf.default.forwarding = 0
net.ipv4.conf.default.igmpv2_unsolicited_report_interval = 10000
net.ipv4.conf.default.igmpv3_unsolicited_report_interval = 1000
net.ipv4.conf.default.ignore_routes_with_linkdown = 0
net.ipv4.conf.default.log_martians = 1
net.ipv4.conf.default.mc_forwarding = 0
net.ipv4.conf.default.medium_id = 0
net.ipv4.conf.default.promote_secondaries = 1
net.ipv4.conf.default.proxy_arp = 0
net.ipv4.conf.default.proxy_arp_pvlan = 0
net.ipv4.conf.default.route_localnet = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.default.shared_media = 1
net.ipv4.conf.default.src_valid_mark = 0
net.ipv4.conf.default.tag = 0
net.ipv4.conf.eth0.accept_local = 0
net.ipv4.conf.eth0.accept_redirects = 1
net.ipv4.conf.eth0.accept_source_route = 0
net.ipv4.conf.eth0.arp_accept = 0
net.ipv4.conf.eth0.arp_announce = 0
net.ipv4.conf.eth0.arp_filter = 0
net.ipv4.conf.eth0.arp_ignore = 0
net.ipv4.conf.eth0.arp_notify = 0
net.ipv4.conf.eth0.bootp_relay = 0
net.ipv4.conf.eth0.disable_policy = 0
net.ipv4.conf.eth0.disable_xfrm = 0
net.ipv4.conf.eth0.drop_gratuitous_arp = 0
net.ipv4.conf.eth0.drop_unicast_in_l2_multicast = 0
net.ipv4.conf.eth0.force_igmp_version = 0
net.ipv4.conf.eth0.forwarding = 0
net.ipv4.conf.eth0.igmpv2_unsolicited_report_interval = 10000
net.ipv4.conf.eth0.igmpv3_unsolicited_report_interval = 1000
net.ipv4.conf.eth0.ignore_routes_with_linkdown = 0
net.ipv4.conf.eth0.log_martians = 0
net.ipv4.conf.eth0.mc_forwarding = 0
net.ipv4.conf.eth0.medium_id = 0
net.ipv4.conf.eth0.promote_secondaries = 1
net.ipv4.conf.eth0.proxy_arp = 0
net.ipv4.conf.eth0.proxy_arp_pvlan = 0
net.ipv4.conf.eth0.route_localnet = 0
net.ipv4.conf.eth0.rp_filter = 1
net.ipv4.conf.eth0.secure_redirects = 1
net.ipv4.conf.eth0.send_redirects = 1
net.ipv4.conf.eth0.shared_media = 1
net.ipv4.conf.eth0.src_valid_mark = 0
net.ipv4.conf.eth0.tag = 0
net.ipv4.conf.eth1.accept_local = 0
net.ipv4.conf.eth1.accept_redirects = 1
net.ipv4.conf.eth1.accept_source_route = 0
net.ipv4.conf.eth1.arp_accept = 0
net.ipv4.conf.eth1.arp_announce = 0
net.ipv4.conf.eth1.arp_filter = 0
net.ipv4.conf.eth1.arp_ignore = 0
net.ipv4.conf.eth1.arp_notify = 0
net.ipv4.conf.eth1.bootp_relay = 0
net.ipv4.conf.eth1.disable_policy = 0
net.ipv4.conf.eth1.disable_xfrm = 0
net.ipv4.conf.eth1.drop_gratuitous_arp = 0
net.ipv4.conf.eth1.drop_unicast_in_l2_multicast = 0
net.ipv4.conf.eth1.force_igmp_version = 0
net.ipv4.conf.eth1.forwarding = 0
net.ipv4.conf.eth1.igmpv2_unsolicited_report_interval = 10000
net.ipv4.conf.eth1.igmpv3_unsolicited_report_interval = 1000
net.ipv4.conf.eth1.ignore_routes_with_linkdown = 0
net.ipv4.conf.eth1.log_martians = 0
net.ipv4.conf.eth1.mc_forwarding = 0
net.ipv4.conf.eth1.medium_id = 0
net.ipv4.conf.eth1.promote_secondaries = 1
net.ipv4.conf.eth1.proxy_arp = 0
net.ipv4.conf.eth1.proxy_arp_pvlan = 0
net.ipv4.conf.eth1.route_localnet = 0
net.ipv4.conf.eth1.rp_filter = 1
net.ipv4.conf.eth1.secure_redirects = 1
net.ipv4.conf.eth1.send_redirects = 1
net.ipv4.conf.eth1.shared_media = 1
net.ipv4.conf.eth1.src_valid_mark = 0
net.ipv4.conf.eth1.tag = 0
net.ipv4.conf.lo.accept_local = 0
net.ipv4.conf.lo.accept_redirects = 1
net.ipv4.conf.lo.accept_source_route = 1
net.ipv4.conf.lo.arp_accept = 0
net.ipv4.conf.lo.arp_announce = 0
net.ipv4.conf.lo.arp_filter = 0
net.ipv4.conf.lo.arp_ignore = 0
net.ipv4.conf.lo.arp_notify = 0
net.ipv4.conf.lo.bootp_relay = 0
net.ipv4.conf.lo.disable_policy = 1
net.ipv4.conf.lo.disable_xfrm = 1
net.ipv4.conf.lo.drop_gratuitous_arp = 0
net.ipv4.conf.lo.drop_unicast_in_l2_multicast = 0
net.ipv4.conf.lo.force_igmp_version = 0
net.ipv4.conf.lo.forwarding = 0
net.ipv4.conf.lo.igmpv2_unsolicited_report_interval = 10000
net.ipv4.conf.lo.igmpv3_unsolicited_report_interval = 1000
net.ipv4.conf.lo.ignore_routes_with_linkdown = 0
net.ipv4.conf.lo.log_martians = 0
net.ipv4.conf.lo.mc_forwarding = 0
net.ipv4.conf.lo.medium_id = 0
net.ipv4.conf.lo.promote_secondaries = 0
net.ipv4.conf.lo.proxy_arp = 0
net.ipv4.conf.lo.proxy_arp_pvlan = 0
net.ipv4.conf.lo.route_localnet = 0
net.ipv4.conf.lo.rp_filter = 0
net.ipv4.conf.lo.secure_redirects = 1
net.ipv4.conf.lo.send_redirects = 1
net.ipv4.conf.lo.shared_media = 1
net.ipv4.conf.lo.src_valid_mark = 0
net.ipv4.conf.lo.tag = 0
net.ipv4.fib_multipath_use_neigh = 0
net.ipv4.fwmark_reflect = 0
net.ipv4.icmp_echo_ignore_all = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_errors_use_inbound_ifaddr = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.icmp_msgs_burst = 50
net.ipv4.icmp_msgs_per_sec = 1000
net.ipv4.icmp_ratelimit = 1000
net.ipv4.icmp_ratemask = 6168
net.ipv4.igmp_link_local_mcast_reports = 1
net.ipv4.igmp_max_memberships = 20
net.ipv4.igmp_max_msf = 10
net.ipv4.igmp_qrv = 2
net.ipv4.inet_peer_maxttl = 600
net.ipv4.inet_peer_minttl = 120
net.ipv4.inet_peer_threshold = 65664
net.ipv4.ip_default_ttl = 64
net.ipv4.ip_dynaddr = 0
net.ipv4.ip_early_demux = 1
net.ipv4.ip_forward = 0
net.ipv4.ip_forward_use_pmtu = 0
net.ipv4.ip_local_port_range = 2000	65535
net.ipv4.ip_local_reserved_ports = 
net.ipv4.ip_no_pmtu_disc = 0
net.ipv4.ip_nonlocal_bind = 0
net.ipv4.ipfrag_high_thresh = 4194304
net.ipv4.ipfrag_low_thresh = 3145728
net.ipv4.ipfrag_max_dist = 64
net.ipv4.ipfrag_secret_interval = 0
net.ipv4.ipfrag_time = 30
net.ipv4.neigh.default.anycast_delay = 100
net.ipv4.neigh.default.app_solicit = 0
net.ipv4.neigh.default.base_reachable_time_ms = 30000
net.ipv4.neigh.default.delay_first_probe_time = 5
net.ipv4.neigh.default.gc_interval = 30
net.ipv4.neigh.default.gc_stale_time = 60
net.ipv4.neigh.default.gc_thresh1 = 128
net.ipv4.neigh.default.gc_thresh2 = 512
net.ipv4.neigh.default.gc_thresh3 = 1024
net.ipv4.neigh.default.locktime = 100
net.ipv4.neigh.default.mcast_resolicit = 0
net.ipv4.neigh.default.mcast_solicit = 3
net.ipv4.neigh.default.proxy_delay = 80
net.ipv4.neigh.default.proxy_qlen = 64
net.ipv4.neigh.default.retrans_time_ms = 1000
net.ipv4.neigh.default.ucast_solicit = 3
net.ipv4.neigh.default.unres_qlen = 31
net.ipv4.neigh.default.unres_qlen_bytes = 65536
net.ipv4.neigh.eth0.anycast_delay = 100
net.ipv4.neigh.eth0.app_solicit = 0
net.ipv4.neigh.eth0.base_reachable_time_ms = 30000
net.ipv4.neigh.eth0.delay_first_probe_time = 5
net.ipv4.neigh.eth0.gc_stale_time = 60
net.ipv4.neigh.eth0.locktime = 100
net.ipv4.neigh.eth0.mcast_resolicit = 0
net.ipv4.neigh.eth0.mcast_solicit = 3
net.ipv4.neigh.eth0.proxy_delay = 80
net.ipv4.neigh.eth0.proxy_qlen = 64
net.ipv4.neigh.eth0.retrans_time_ms = 1000
net.ipv4.neigh.eth0.ucast_solicit = 3
net.ipv4.neigh.eth0.unres_qlen = 31
net.ipv4.neigh.eth0.unres_qlen_bytes = 65536
net.ipv4.neigh.eth1.anycast_delay = 100
net.ipv4.neigh.eth1.app_solicit = 0
net.ipv4.neigh.eth1.base_reachable_time_ms = 30000
net.ipv4.neigh.eth1.delay_first_probe_time = 5
net.ipv4.neigh.eth1.gc_stale_time = 60
net.ipv4.neigh.eth1.locktime = 100
net.ipv4.neigh.eth1.mcast_resolicit = 0
net.ipv4.neigh.eth1.mcast_solicit = 3
net.ipv4.neigh.eth1.proxy_delay = 80
net.ipv4.neigh.eth1.proxy_qlen = 64
net.ipv4.neigh.eth1.retrans_time_ms = 1000
net.ipv4.neigh.eth1.ucast_solicit = 3
net.ipv4.neigh.eth1.unres_qlen = 31
net.ipv4.neigh.eth1.unres_qlen_bytes = 65536
net.ipv4.neigh.lo.anycast_delay = 100
net.ipv4.neigh.lo.app_solicit = 0
net.ipv4.neigh.lo.base_reachable_time_ms = 30000
net.ipv4.neigh.lo.delay_first_probe_time = 5
net.ipv4.neigh.lo.gc_stale_time = 60
net.ipv4.neigh.lo.locktime = 100
net.ipv4.neigh.lo.mcast_resolicit = 0
net.ipv4.neigh.lo.mcast_solicit = 3
net.ipv4.neigh.lo.proxy_delay = 80
net.ipv4.neigh.lo.proxy_qlen = 64
net.ipv4.neigh.lo.retrans_time_ms = 1000
net.ipv4.neigh.lo.ucast_solicit = 3
net.ipv4.neigh.lo.unres_qlen = 31
net.ipv4.neigh.lo.unres_qlen_bytes = 65536
net.ipv4.ping_group_range = 1	0
net.ipv4.route.error_burst = 5000
net.ipv4.route.error_cost = 1000
net.ipv4.route.gc_elasticity = 8
net.ipv4.route.gc_interval = 60
net.ipv4.route.gc_min_interval = 0
net.ipv4.route.gc_min_interval_ms = 500
net.ipv4.route.gc_thresh = -1
net.ipv4.route.gc_timeout = 300
net.ipv4.route.max_size = 2147483647
net.ipv4.route.min_adv_mss = 256
net.ipv4.route.min_pmtu = 552
net.ipv4.route.mtu_expires = 600
net.ipv4.route.redirect_load = 20
net.ipv4.route.redirect_number = 9
net.ipv4.route.redirect_silence = 20480
net.ipv4.tcp_abort_on_overflow = 0
net.ipv4.tcp_adv_win_scale = 1
net.ipv4.tcp_allowed_congestion_control = cubic reno
net.ipv4.tcp_app_win = 31
net.ipv4.tcp_autocorking = 1
net.ipv4.tcp_available_congestion_control = cubic reno
net.ipv4.tcp_base_mss = 1024
net.ipv4.tcp_challenge_ack_limit = 1000
net.ipv4.tcp_congestion_control = cubic
net.ipv4.tcp_dsack = 1
net.ipv4.tcp_early_retrans = 3
net.ipv4.tcp_ecn = 2
net.ipv4.tcp_ecn_fallback = 1
net.ipv4.tcp_fack = 1
net.ipv4.tcp_fastopen = 1
net.ipv4.tcp_fastopen_key = 00000000-00000000-00000000-00000000
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_frto = 2
net.ipv4.tcp_fwmark_accept = 0
net.ipv4.tcp_invalid_ratelimit = 500
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_limit_output_bytes = 262144
net.ipv4.tcp_low_latency = 0
net.ipv4.tcp_max_orphans = 4096
net.ipv4.tcp_max_reordering = 300
net.ipv4.tcp_max_syn_backlog = 30000
net.ipv4.tcp_max_tw_buckets = 1440000
net.ipv4.tcp_mem = 10872	14496	21744
net.ipv4.tcp_min_rtt_wlen = 300
net.ipv4.tcp_min_tso_segs = 2
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_mtu_probing = 0
net.ipv4.tcp_no_metrics_save = 0
net.ipv4.tcp_notsent_lowat = -1
net.ipv4.tcp_orphan_retries = 0
net.ipv4.tcp_pacing_ca_ratio = 120
net.ipv4.tcp_pacing_ss_ratio = 200
net.ipv4.tcp_probe_interval = 600
net.ipv4.tcp_probe_threshold = 8
net.ipv4.tcp_recovery = 1
net.ipv4.tcp_reordering = 3
net.ipv4.tcp_retrans_collapse = 1
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 15
net.ipv4.tcp_rfc1337 = 1
net.ipv4.tcp_rmem = 4096	87380	16777216
net.ipv4.tcp_sack = 1
net.ipv4.tcp_slow_start_after_idle = 1
net.ipv4.tcp_stdurg = 0
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_thin_dupack = 0
net.ipv4.tcp_thin_linear_timeouts = 0
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tso_win_divisor = 3
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_wmem = 4096	87380	16777216
net.ipv4.tcp_workaround_signed_windows = 0
net.ipv4.udp_mem = 21744	28993	43488
net.ipv4.udp_rmem_min = 4096
net.ipv4.udp_wmem_min = 4096
net.ipv4.xfrm4_gc_thresh = 2147483647
net.ipv6.anycast_src_echo_reply = 0
net.ipv6.auto_flowlabels = 1
net.ipv6.bindv6only = 0
net.ipv6.calipso_cache_bucket_size = 10
net.ipv6.calipso_cache_enable = 1
net.ipv6.conf.all.accept_dad = 1
net.ipv6.conf.all.accept_ra = 1
net.ipv6.conf.all.accept_ra_defrtr = 1
net.ipv6.conf.all.accept_ra_from_local = 0
net.ipv6.conf.all.accept_ra_min_hop_limit = 1
net.ipv6.conf.all.accept_ra_mtu = 1
net.ipv6.conf.all.accept_ra_pinfo = 1
net.ipv6.conf.all.accept_ra_rt_info_max_plen = 0
net.ipv6.conf.all.accept_ra_rtr_pref = 1
net.ipv6.conf.all.accept_redirects = 1
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.all.autoconf = 1
net.ipv6.conf.all.dad_transmits = 1
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.all.drop_unicast_in_l2_multicast = 0
net.ipv6.conf.all.drop_unsolicited_na = 0
net.ipv6.conf.all.force_mld_version = 0
net.ipv6.conf.all.force_tllao = 0
net.ipv6.conf.all.forwarding = 0
net.ipv6.conf.all.hop_limit = 64
net.ipv6.conf.all.ignore_routes_with_linkdown = 0
net.ipv6.conf.all.keep_addr_on_down = 0
net.ipv6.conf.all.max_addresses = 16
net.ipv6.conf.all.max_desync_factor = 600
net.ipv6.conf.all.mc_forwarding = 0
net.ipv6.conf.all.mldv1_unsolicited_report_interval = 10000
net.ipv6.conf.all.mldv2_unsolicited_report_interval = 1000
net.ipv6.conf.all.mtu = 1280
net.ipv6.conf.all.ndisc_notify = 0
net.ipv6.conf.all.optimistic_dad = 0
net.ipv6.conf.all.proxy_ndp = 0
net.ipv6.conf.all.regen_max_retry = 3
net.ipv6.conf.all.router_probe_interval = 60
net.ipv6.conf.all.router_solicitation_delay = 1
net.ipv6.conf.all.router_solicitation_interval = 4
net.ipv6.conf.all.router_solicitation_max_interval = 3600
net.ipv6.conf.all.router_solicitations = -1
sysctl: reading key "net.ipv6.conf.all.stable_secret"
net.ipv6.conf.all.suppress_frag_ndisc = 1
net.ipv6.conf.all.temp_prefered_lft = 86400
net.ipv6.conf.all.temp_valid_lft = 604800
net.ipv6.conf.all.use_oif_addrs_only = 0
net.ipv6.conf.all.use_optimistic = 0
net.ipv6.conf.all.use_tempaddr = 0
net.ipv6.conf.default.accept_dad = 1
net.ipv6.conf.default.accept_ra = 1
net.ipv6.conf.default.accept_ra_defrtr = 0
net.ipv6.conf.default.accept_ra_from_local = 0
net.ipv6.conf.default.accept_ra_min_hop_limit = 1
net.ipv6.conf.default.accept_ra_mtu = 1
net.ipv6.conf.default.accept_ra_pinfo = 0
net.ipv6.conf.default.accept_ra_rt_info_max_plen = 0
net.ipv6.conf.default.accept_ra_rtr_pref = 0
net.ipv6.conf.default.accept_redirects = 1
net.ipv6.conf.default.accept_source_route = 0
net.ipv6.conf.default.autoconf = 0
net.ipv6.conf.default.dad_transmits = 0
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.default.drop_unicast_in_l2_multicast = 0
net.ipv6.conf.default.drop_unsolicited_na = 0
net.ipv6.conf.default.force_mld_version = 0
net.ipv6.conf.default.force_tllao = 0
net.ipv6.conf.default.forwarding = 0
net.ipv6.conf.default.hop_limit = 64
net.ipv6.conf.default.ignore_routes_with_linkdown = 0
net.ipv6.conf.default.keep_addr_on_down = 0
net.ipv6.conf.default.max_addresses = 1
net.ipv6.conf.default.max_desync_factor = 600
net.ipv6.conf.default.mc_forwarding = 0
net.ipv6.conf.default.mldv1_unsolicited_report_interval = 10000
net.ipv6.conf.default.mldv2_unsolicited_report_interval = 1000
net.ipv6.conf.default.mtu = 1280
net.ipv6.conf.default.ndisc_notify = 0
net.ipv6.conf.default.optimistic_dad = 0
net.ipv6.conf.default.proxy_ndp = 0
net.ipv6.conf.default.regen_max_retry = 3
net.ipv6.conf.default.router_probe_interval = 60
net.ipv6.conf.default.router_solicitation_delay = 1
net.ipv6.conf.default.router_solicitation_interval = 4
net.ipv6.conf.default.router_solicitation_max_interval = 3600
net.ipv6.conf.default.router_solicitations = 0
sysctl: reading key "net.ipv6.conf.default.stable_secret"
net.ipv6.conf.default.suppress_frag_ndisc = 1
net.ipv6.conf.default.temp_prefered_lft = 86400
net.ipv6.conf.default.temp_valid_lft = 604800
net.ipv6.conf.default.use_oif_addrs_only = 0
net.ipv6.conf.default.use_optimistic = 0
net.ipv6.conf.default.use_tempaddr = 0
net.ipv6.conf.eth0.accept_dad = 1
net.ipv6.conf.eth0.accept_ra = 1
net.ipv6.conf.eth0.accept_ra_defrtr = 1
net.ipv6.conf.eth0.accept_ra_from_local = 0
net.ipv6.conf.eth0.accept_ra_min_hop_limit = 1
net.ipv6.conf.eth0.accept_ra_mtu = 1
net.ipv6.conf.eth0.accept_ra_pinfo = 1
net.ipv6.conf.eth0.accept_ra_rt_info_max_plen = 0
net.ipv6.conf.eth0.accept_ra_rtr_pref = 1
net.ipv6.conf.eth0.accept_redirects = 1
net.ipv6.conf.eth0.accept_source_route = 0
net.ipv6.conf.eth0.autoconf = 1
net.ipv6.conf.eth0.dad_transmits = 1
net.ipv6.conf.eth0.disable_ipv6 = 1
net.ipv6.conf.eth0.drop_unicast_in_l2_multicast = 0
net.ipv6.conf.eth0.drop_unsolicited_na = 0
net.ipv6.conf.eth0.force_mld_version = 0
net.ipv6.conf.eth0.force_tllao = 0
net.ipv6.conf.eth0.forwarding = 0
net.ipv6.conf.eth0.hop_limit = 64
net.ipv6.conf.eth0.ignore_routes_with_linkdown = 0
net.ipv6.conf.eth0.keep_addr_on_down = 0
net.ipv6.conf.eth0.max_addresses = 16
net.ipv6.conf.eth0.max_desync_factor = 600
net.ipv6.conf.eth0.mc_forwarding = 0
net.ipv6.conf.eth0.mldv1_unsolicited_report_interval = 10000
net.ipv6.conf.eth0.mldv2_unsolicited_report_interval = 1000
net.ipv6.conf.eth0.mtu = 1500
net.ipv6.conf.eth0.ndisc_notify = 0
net.ipv6.conf.eth0.optimistic_dad = 0
net.ipv6.conf.eth0.proxy_ndp = 0
net.ipv6.conf.eth0.regen_max_retry = 3
net.ipv6.conf.eth0.router_probe_interval = 60
net.ipv6.conf.eth0.router_solicitation_delay = 1
net.ipv6.conf.eth0.router_solicitation_interval = 4
net.ipv6.conf.eth0.router_solicitation_max_interval = 3600
net.ipv6.conf.eth0.router_solicitations = -1
sysctl: reading key "net.ipv6.conf.eth0.stable_secret"
net.ipv6.conf.eth0.suppress_frag_ndisc = 1
net.ipv6.conf.eth0.temp_prefered_lft = 86400
net.ipv6.conf.eth0.temp_valid_lft = 604800
net.ipv6.conf.eth0.use_oif_addrs_only = 0
net.ipv6.conf.eth0.use_optimistic = 0
net.ipv6.conf.eth0.use_tempaddr = 0
net.ipv6.conf.eth1.accept_dad = 1
net.ipv6.conf.eth1.accept_ra = 1
net.ipv6.conf.eth1.accept_ra_defrtr = 1
net.ipv6.conf.eth1.accept_ra_from_local = 0
net.ipv6.conf.eth1.accept_ra_min_hop_limit = 1
net.ipv6.conf.eth1.accept_ra_mtu = 1
net.ipv6.conf.eth1.accept_ra_pinfo = 1
net.ipv6.conf.eth1.accept_ra_rt_info_max_plen = 0
net.ipv6.conf.eth1.accept_ra_rtr_pref = 1
net.ipv6.conf.eth1.accept_redirects = 1
net.ipv6.conf.eth1.accept_source_route = 0
net.ipv6.conf.eth1.autoconf = 1
net.ipv6.conf.eth1.dad_transmits = 1
net.ipv6.conf.eth1.disable_ipv6 = 1
net.ipv6.conf.eth1.drop_unicast_in_l2_multicast = 0
net.ipv6.conf.eth1.drop_unsolicited_na = 0
net.ipv6.conf.eth1.force_mld_version = 0
net.ipv6.conf.eth1.force_tllao = 0
net.ipv6.conf.eth1.forwarding = 0
net.ipv6.conf.eth1.hop_limit = 64
net.ipv6.conf.eth1.ignore_routes_with_linkdown = 0
net.ipv6.conf.eth1.keep_addr_on_down = 0
net.ipv6.conf.eth1.max_addresses = 16
net.ipv6.conf.eth1.max_desync_factor = 600
net.ipv6.conf.eth1.mc_forwarding = 0
net.ipv6.conf.eth1.mldv1_unsolicited_report_interval = 10000
net.ipv6.conf.eth1.mldv2_unsolicited_report_interval = 1000
net.ipv6.conf.eth1.mtu = 1450
net.ipv6.conf.eth1.ndisc_notify = 0
net.ipv6.conf.eth1.optimistic_dad = 0
net.ipv6.conf.eth1.proxy_ndp = 0
net.ipv6.conf.eth1.regen_max_retry = 3
net.ipv6.conf.eth1.router_probe_interval = 60
net.ipv6.conf.eth1.router_solicitation_delay = 1
net.ipv6.conf.eth1.router_solicitation_interval = 4
net.ipv6.conf.eth1.router_solicitation_max_interval = 3600
net.ipv6.conf.eth1.router_solicitations = -1
sysctl: reading key "net.ipv6.conf.eth1.stable_secret"
net.ipv6.conf.eth1.suppress_frag_ndisc = 1
net.ipv6.conf.eth1.temp_prefered_lft = 86400
net.ipv6.conf.eth1.temp_valid_lft = 604800
net.ipv6.conf.eth1.use_oif_addrs_only = 0
net.ipv6.conf.eth1.use_optimistic = 0
net.ipv6.conf.eth1.use_tempaddr = 0
net.ipv6.conf.lo.accept_dad = -1
net.ipv6.conf.lo.accept_ra = 1
net.ipv6.conf.lo.accept_ra_defrtr = 1
net.ipv6.conf.lo.accept_ra_from_local = 0
net.ipv6.conf.lo.accept_ra_min_hop_limit = 1
net.ipv6.conf.lo.accept_ra_mtu = 1
net.ipv6.conf.lo.accept_ra_pinfo = 1
net.ipv6.conf.lo.accept_ra_rt_info_max_plen = 0
net.ipv6.conf.lo.accept_ra_rtr_pref = 1
net.ipv6.conf.lo.accept_redirects = 1
net.ipv6.conf.lo.accept_source_route = 0
net.ipv6.conf.lo.autoconf = 1
net.ipv6.conf.lo.dad_transmits = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.lo.drop_unicast_in_l2_multicast = 0
net.ipv6.conf.lo.drop_unsolicited_na = 0
net.ipv6.conf.lo.force_mld_version = 0
net.ipv6.conf.lo.force_tllao = 0
net.ipv6.conf.lo.forwarding = 0
net.ipv6.conf.lo.hop_limit = 64
net.ipv6.conf.lo.ignore_routes_with_linkdown = 0
net.ipv6.conf.lo.keep_addr_on_down = 0
net.ipv6.conf.lo.max_addresses = 16
net.ipv6.conf.lo.max_desync_factor = 600
net.ipv6.conf.lo.mc_forwarding = 0
net.ipv6.conf.lo.mldv1_unsolicited_report_interval = 10000
net.ipv6.conf.lo.mldv2_unsolicited_report_interval = 1000
net.ipv6.conf.lo.mtu = 65536
net.ipv6.conf.lo.ndisc_notify = 0
net.ipv6.conf.lo.optimistic_dad = 0
net.ipv6.conf.lo.proxy_ndp = 0
net.ipv6.conf.lo.regen_max_retry = 3
net.ipv6.conf.lo.router_probe_interval = 60
net.ipv6.conf.lo.router_solicitation_delay = 1
net.ipv6.conf.lo.router_solicitation_interval = 4
net.ipv6.conf.lo.router_solicitation_max_interval = 3600
net.ipv6.conf.lo.router_solicitations = -1
sysctl: reading key "net.ipv6.conf.lo.stable_secret"
net.ipv6.conf.lo.suppress_frag_ndisc = 1
net.ipv6.conf.lo.temp_prefered_lft = 86400
net.ipv6.conf.lo.temp_valid_lft = 604800
net.ipv6.conf.lo.use_oif_addrs_only = 0
net.ipv6.conf.lo.use_optimistic = 0
net.ipv6.conf.lo.use_tempaddr = -1
net.ipv6.flowlabel_consistency = 1
net.ipv6.flowlabel_state_ranges = 0
net.ipv6.fwmark_reflect = 0
net.ipv6.icmp.ratelimit = 1000
net.ipv6.idgen_delay = 1
net.ipv6.idgen_retries = 3
net.ipv6.ip6frag_high_thresh = 4194304
net.ipv6.ip6frag_low_thresh = 3145728
net.ipv6.ip6frag_secret_interval = 0
net.ipv6.ip6frag_time = 60
net.ipv6.ip_nonlocal_bind = 0
net.ipv6.mld_max_msf = 64
net.ipv6.mld_qrv = 2
net.ipv6.neigh.default.anycast_delay = 100
net.ipv6.neigh.default.app_solicit = 0
net.ipv6.neigh.default.base_reachable_time_ms = 30000
net.ipv6.neigh.default.delay_first_probe_time = 5
net.ipv6.neigh.default.gc_interval = 30
net.ipv6.neigh.default.gc_stale_time = 60
net.ipv6.neigh.default.gc_thresh1 = 128
net.ipv6.neigh.default.gc_thresh2 = 512
net.ipv6.neigh.default.gc_thresh3 = 1024
net.ipv6.neigh.default.locktime = 0
net.ipv6.neigh.default.mcast_resolicit = 0
net.ipv6.neigh.default.mcast_solicit = 3
net.ipv6.neigh.default.proxy_delay = 80
net.ipv6.neigh.default.proxy_qlen = 64
net.ipv6.neigh.default.retrans_time_ms = 1000
net.ipv6.neigh.default.ucast_solicit = 3
net.ipv6.neigh.default.unres_qlen = 31
net.ipv6.neigh.default.unres_qlen_bytes = 65536
net.ipv6.neigh.eth0.anycast_delay = 100
net.ipv6.neigh.eth0.app_solicit = 0
net.ipv6.neigh.eth0.base_reachable_time_ms = 30000
net.ipv6.neigh.eth0.delay_first_probe_time = 5
net.ipv6.neigh.eth0.gc_stale_time = 60
net.ipv6.neigh.eth0.locktime = 0
net.ipv6.neigh.eth0.mcast_resolicit = 0
net.ipv6.neigh.eth0.mcast_solicit = 3
net.ipv6.neigh.eth0.proxy_delay = 80
net.ipv6.neigh.eth0.proxy_qlen = 64
net.ipv6.neigh.eth0.retrans_time_ms = 1000
net.ipv6.neigh.eth0.ucast_solicit = 3
net.ipv6.neigh.eth0.unres_qlen = 31
net.ipv6.neigh.eth0.unres_qlen_bytes = 65536
net.ipv6.neigh.eth1.anycast_delay = 100
net.ipv6.neigh.eth1.app_solicit = 0
net.ipv6.neigh.eth1.base_reachable_time_ms = 30000
net.ipv6.neigh.eth1.delay_first_probe_time = 5
net.ipv6.neigh.eth1.gc_stale_time = 60
net.ipv6.neigh.eth1.locktime = 0
net.ipv6.neigh.eth1.mcast_resolicit = 0
net.ipv6.neigh.eth1.mcast_solicit = 3
net.ipv6.neigh.eth1.proxy_delay = 80
net.ipv6.neigh.eth1.proxy_qlen = 64
net.ipv6.neigh.eth1.retrans_time_ms = 1000
net.ipv6.neigh.eth1.ucast_solicit = 3
net.ipv6.neigh.eth1.unres_qlen = 31
net.ipv6.neigh.eth1.unres_qlen_bytes = 65536
net.ipv6.neigh.lo.anycast_delay = 100
net.ipv6.neigh.lo.app_solicit = 0
net.ipv6.neigh.lo.base_reachable_time_ms = 30000
net.ipv6.neigh.lo.delay_first_probe_time = 5
net.ipv6.neigh.lo.gc_stale_time = 60
net.ipv6.neigh.lo.locktime = 0
net.ipv6.neigh.lo.mcast_resolicit = 0
net.ipv6.neigh.lo.mcast_solicit = 3
net.ipv6.neigh.lo.proxy_delay = 80
net.ipv6.neigh.lo.proxy_qlen = 64
net.ipv6.neigh.lo.retrans_time_ms = 1000
net.ipv6.neigh.lo.ucast_solicit = 3
net.ipv6.neigh.lo.unres_qlen = 31
net.ipv6.neigh.lo.unres_qlen_bytes = 65536
net.ipv6.route.gc_elasticity = 9
net.ipv6.route.gc_interval = 30
net.ipv6.route.gc_min_interval = 0
net.ipv6.route.gc_min_interval_ms = 500
net.ipv6.route.gc_thresh = 1024
net.ipv6.route.gc_timeout = 60
net.ipv6.route.max_size = 4096
net.ipv6.route.min_adv_mss = 1220
net.ipv6.route.mtu_expires = 600
net.ipv6.xfrm6_gc_thresh = 2147483647
net.netfilter.nf_conntrack_acct = 0
net.netfilter.nf_conntrack_buckets = 8192
net.netfilter.nf_conntrack_checksum = 1
net.netfilter.nf_conntrack_count = 7
net.netfilter.nf_conntrack_events = 1
net.netfilter.nf_conntrack_expect_max = 128
net.netfilter.nf_conntrack_frag6_high_thresh = 4194304
net.netfilter.nf_conntrack_frag6_low_thresh = 3145728
net.netfilter.nf_conntrack_frag6_timeout = 60
net.netfilter.nf_conntrack_generic_timeout = 600
net.netfilter.nf_conntrack_helper = 0
net.netfilter.nf_conntrack_icmp_timeout = 30
net.netfilter.nf_conntrack_icmpv6_timeout = 30
net.netfilter.nf_conntrack_log_invalid = 0
net.netfilter.nf_conntrack_max = 32768
net.netfilter.nf_conntrack_tcp_be_liberal = 0
net.netfilter.nf_conntrack_tcp_loose = 1
net.netfilter.nf_conntrack_tcp_max_retrans = 3
net.netfilter.nf_conntrack_tcp_timeout_close = 10
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_established = 432000
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_last_ack = 30
net.netfilter.nf_conntrack_tcp_timeout_max_retrans = 300
net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 60
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_unacknowledged = 300
net.netfilter.nf_conntrack_timestamp = 0
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 180
net.netfilter.nf_log.0 = NONE
net.netfilter.nf_log.1 = NONE
net.netfilter.nf_log.10 = NONE
net.netfilter.nf_log.11 = NONE
net.netfilter.nf_log.12 = NONE
net.netfilter.nf_log.2 = NONE
net.netfilter.nf_log.3 = NONE
net.netfilter.nf_log.4 = NONE
net.netfilter.nf_log.5 = NONE
net.netfilter.nf_log.6 = NONE
net.netfilter.nf_log.7 = NONE
net.netfilter.nf_log.8 = NONE
net.netfilter.nf_log.9 = NONE
net.nf_conntrack_max = 32768
net.unix.max_dgram_qlen = 512
user.max_cgroup_namespaces = 3887
user.max_ipc_namespaces = 3887
user.max_mnt_namespaces = 3887
user.max_net_namespaces = 3887
user.max_pid_namespaces = 3887
user.max_user_namespaces = 3887
user.max_uts_namespaces = 3887
vm.admin_reserve_kbytes = 8192
vm.block_dump = 0
vm.compact_unevictable_allowed = 1
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 3
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000
vm.dirty_ratio = 10
vm.dirty_writeback_centisecs = 500
vm.dirtytime_expire_seconds = 43200
vm.drop_caches = 0
vm.extfrag_threshold = 500
vm.hugepages_treat_as_movable = 0
vm.hugetlb_shm_group = 0
vm.laptop_mode = 0
vm.legacy_va_layout = 0
vm.lowmem_reserve_ratio = 256	256	32
vm.max_map_count = 262144
vm.memory_failure_early_kill = 0
vm.memory_failure_recovery = 1
vm.min_free_kbytes = 45056
vm.min_slab_ratio = 5
vm.min_unmapped_ratio = 1
vm.mmap_min_addr = 4096
vm.mmap_rnd_bits = 28
vm.mmap_rnd_compat_bits = 8
vm.nr_hugepages = 0
vm.nr_hugepages_mempolicy = 0
vm.nr_overcommit_hugepages = 0
vm.nr_pdflush_threads = 0
vm.numa_zonelist_order = default
vm.oom_dump_tasks = 1
vm.oom_kill_allocating_task = 0
vm.overcommit_kbytes = 0
vm.overcommit_memory = 0
vm.overcommit_ratio = 50
vm.page-cluster = 3
vm.panic_on_oom = 0
vm.percpu_pagelist_fraction = 0
vm.stat_interval = 1
vm.swappiness = 10
vm.user_reserve_kbytes = 30967
vm.vfs_cache_pressure = 100
vm.watermark_scale_factor = 10
vm.zone_reclaim_mode = 0
[root@printexpress etc]# vi sysctl.conf

# System default settings live in /usr/lib/sysctl.d/00-system.conf.
# To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
#
# For more information, see sysctl.conf(5) and sysctl.d(5).

# Increase number of incoming connections
net.core.somaxconn = 16384

# Increase the maximum amount of option memory buffers
net.core.optmem_max = 25165824

# Increase Linux auto tuning TCP buffer limits
# min, default, and max number of bytes to use
# set max to at least 4MB, or higher if you use very high BDP paths
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 65536

# Decrease the time default value for tcp_fin_timeout connection
net.ipv4.tcp_fin_timeout = 15

# Number of times SYNACKs for passive TCP connection.
net.ipv4.tcp_synack_retries = 2

# Decrease the time default value for connections to keep alive
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# Enable timestamps as defined in RFC1323:
net.ipv4.tcp_timestamps = 1

# Allowed local port range
net.ipv4.ip_local_port_range = 2000 65535

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
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

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

# Disable ipv6 for security (!! Optional if you're using ipv6 !!)
net.ipv6.conf.all.disable_ipv6= 1
net.ipv6.conf.default.disable_ipv6= 1
net.ipv6.conf.lo.disable_ipv6= 1
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

### Yum Utils 

**Yum-utils** is a collection of utilities and plugins extending and supplementing yum in different ways. It included in the base repo (which is enabled by default). However if you install as minimal you have to install it manually by typing. 

```
sudo yum -y install yum-utils
```

Read more about yum-utils 
https://www.if-not-true-then-false.com/2012/delete-remove-old-kernels-on-fedora-centos-red-hat-rhel/
http://www.tecmint.com/linux-yum-package-management-with-yum-utils/

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

### Search it 
```
sudo yum search htop
sudo yum info htop
sudo yum install htop
```

And, there you have it, a larger number of packages to install from EPEL repo on a CentOS and 
Red Hat Enterprise Linux (RHEL) version 7.x.


### OpenSSL(with ALPN support)

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

As you can see, only Ubuntu 16.04 LTS supports ALPN. This means that if you’re running your website on any other major operating system, the OpenSSL version shipped with the operating system does not support ALPN and Chrome users will be downgraded to HTTP/1.

So to enable HTTP/2 on ALPN in chrome browser you need to be sure that you have already installed ***OpenSSL that supported ALPN** which is version >= 1.0.2.

According to https://en.wikipedia.org/wiki/OpenSSL#Major_version_releases


### The following steps describe how to upgrade OpenSSL 

To compile openssl from source you need to install compiler and required libraries to build nginx.
```
sudo yum groupinstall 'Development Tools'
sudo yum -y install autoconf automake bind-utils wget curl unzip gcc-c++ pcre-devel zlib-devel libtool make nmap-netcat ntp pam-devel
```

1). Verify the current openssl version by command
```
openssl version
OpenSSL 1.0.1e-fips 11 Feb 2013
```

2). To view the lastest openssl package from base repository
```
yum info openssl
```

3). Download the latest version of OpenSSL, do as follows:
```
cd /usr/src
wget https://www.openssl.org/source/openssl-1.0.2-latest.tar.gz
tar -zxf openssl-1.0.2-latest.tar.gz
```

Note: If you want to install other version you can download it from https://www.openssl.org/source/

4). Go to the source directory, then generate a config file by follow commands
```
cd openssl-1.0.2j
./config
```

5). Compile the source, test and then install the package (must login as root)
```
make
make test
make install
```

Note: This will take a while upon CPU capacity. If your CPU have more than 1 core you can add suffix -j4 for using 4 cores to compile.
For example: make -j4 to use 4 cores to compile the source code.

6). Move old openssl installed version to the root folder for backup or you can delete it
```
mv /usr/bin/openssl /root/
```

7). Create a symbolic link
```
ln -s /usr/local/ssl/bin/openssl /usr/bin/openssl
```

8). Verify the OpenSSL version 
```
openssl version
OpenSSL 1.0.2j  26 Sep 2016
```

!! Done. Easy right let continue the next package ? !!


### Nginx from source (with ALPN support + HTTP2)

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/NGINX_logo.png" alt="NGINX"/>
</p>

Compiling NGINX from the sources provides you with more flexibility: you can add particular NGINX modules or 3rd party modules and apply latest security patches. 

For example of core NGINX Modules: 

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


For example of 3rd Party modules: 

**ngx_brotli module** - An open source data compression library introducing by Google, based on a modern variant of the LZ77 algorithm, Huffman coding and 2nd order context modeling. (By default nginx bundle with **gzip** compression library)
 
Brotli performance https://www.opencpu.org/posts/brotli-benchmarks/

Installing nginx brotli module https://github.com/google/ngx_brotli

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/nginx_page_speed.png" alt="NGINX_PAGESPEED"/>
</p>

**ngx_pagespeed** - Speeds up your site and reduces page load time by automatically applying web performance best practices to pages and associated assets (CSS, JavaScript, images) without requiring you to modify your existing content or workflow.  
Rewrites webpages and associated assets to reduce latency and bandwidth
 
Installing page speed module https://developers.google.com/speed/pagespeed/module/build_ngx_pagespeed_from_source

-------------------------------------------------------------------------------------------------------------------------

View more NGINX 3rd Party Modules: https://www.nginx.com/resources/wiki/modules/

Read more about extending nginx https://www.nginx.com/resources/wiki/extending/

Read more about dynamic modules https://www.nginx.com/blog/dynamic-modules-nginx-1-9-11/

-------------------------------------------------------------------------------------------------------------------------

### Choosing Between a Stable or a Mainline Version

NGINX Open Source is available in 2 versions:

**The mainline version** This version includes the latest features and bugfixes and is always up-to-date. It is reliable, but it may include some experimental modules, and it may also have some number of new bugs.

**The stable version** This version doesn’t have new features, but includes critical bug fixes that are always backported to the mainline version. The stable version is recommended for production servers.


### Prerequisites

**!! Before we getting start you need to verify that you have installed opnssl with ALPN feature support**
```
openssl version
```
The openssl version must be 1.0.2 or higher.


### Step 1 Installing compiler and libraries (If you already installed skip this step)

To compile nginx from source you need to install compiler and required libraries to build nginx.

1.1 Need to install epel repo for additional libraries, if not install by typing
```
sudo yum install -y epel-release
```
1.2 Then update the os environment
```
sudo yum update 
```
1.3 Install the compiler 
```
sudo yum groupinstall 'Development Tools'
sudo yum -y install autoconf automake bind-utils wget curl unzip gcc-c++ pcre-devel zlib-devel libtool make nmap-netcat ntp pam-devel
```

### Step 2 Installing NGINX Dependencies

Prior to compiling NGINX from the sources, it is necessary to install its dependencies:

2.1 Install PCRE library
The PCRE library required by NGINX Core and Rewrite modules and provides support for regular expressions:
```
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.39.tar.gz
tar -zxf pcre-8.39.tar.gz
cd pcre-8.39
./configure
make
sudo make install
```

2.2 Install ZLIB library
The zlib library required by NGINX Gzip module for headers compression:
```
wget http://zlib.net/zlib-1.2.8.tar.gz
tar -zxf zlib-1.2.8.tar.gz
cd zlib-1.2.8
./configure
make
sudo make install
```

2.3 Install OpenSSL library (lastest)
The OpenSSL library required by NGINX SSL modules to support the HTTPS protocol:

Please see the OpenSSL (with ALPN support) section



### Step 3 Download NGINX source code

```
wget http://nginx.org/download/nginx-1.10.2.tar.gz
tar zxf nginx-1.xx.x.tar.gz
cd nginx-1.xx.x
```

Other version please see https://nginx.org/en/download.html

3.1 Config the nginx for built
```
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
--with-pcre=/usr/local/src/pcre-8.39 \
--with-zlib=/usr/local/src/zlib-1.2.8 \
--with-openssl=/usr/local/src/openssl-1.0.2j \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-file-aio \
--with-threads \
--with-ipv6 \
--with-http_addition_module \
--with-http_auth_request_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_random_index_module \
--with-http_realip_module \
--with-http_secure_link_module \
--with-http_slice_module \
--with-http_sub_module \
--with-http_v2_module \
 --with-mail \
--with-mail_ssl_module \
--with-stream \
--with-stream_ssl_module \
--with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic'
```

3.2 Compile and install the build:
```
make
sudo make install
```

3.3 After the installation process has finished with success add nginx system user (with /etc/nginx/ as his home directory and with no valid shell), the user that Nginx will run as by issuing the following command.

```
useradd -d /etc/nginx/ -s /sbin/nologin nginx
```

3.4 Change user in nginx configure file
```
vi /etc/nginx/nginx.conf
# Then change the user from nobody to nginx, then save and exit.

user nginx;
```

3.5 Config the firewall
```
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
systemctl restart firewalld.service
```

3.5 start nginx service by using below command
```
/usr/sbin/nginx -c /etc/nginx/nginx.conf
```

3.6 check nginx process 
```
ps -ef|grep nginx
```

3.7 to stop nginx service using below command
```
kill -9 PID-Of-Nginx
```

3.8 add nginx as systemd service by create a file "nginx.service" in /lib/systemd/system/nginx.service
 then copy below into the file
 
```
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target
 
[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
 ```
 
3.9 then reload the system files
```
systemctl daemon-reload
```

3.10 create startup script 
```
sudo ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx
sudo nano /etc/init.d/nginx then copy script below
```

```
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
   user=`$nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
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

3.11 set permission to make this script be executable 
```
chmod +x /etc/init.d/nginx
sudo systemctl enable nginx
```

3.12 To make sure that Nginx starts and stops every time with the Droplet, add it to the default runlevels with the command:
```
sudo chkconfig nginx on
sudo service nginx restart
```

3.13 Finally, to verify nginx version and opensssl that built by type**
```
nginx -V
nginx version: nginx/1.9.13 (nginx-plus-r9)
built by gcc 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04.1)
built with OpenSSL 1.0.1f 6 Jan 2014
```

3.14 Start nginx 
```
sudo nginx
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
NOTE: If you want to install speedtest module (By Google) you must installed following libraries
```
gcc 
gcc-c++ 
pcre-devel 
zlib-devel 
make 
unzip 
openssl-devel 
libaio-devel
glibc 
glibc-devel 
glibc-headers
libevent
linux-vdso.so.1
libpthread.so.0
libcrypt.so.1
libstdc++.so.6
librt.so.1
libm.so.6
libpcre.so.0
libssl.so.10
libcrypto.so.10
libdl.so.2
libz.so.1
libgcc_s.so.1
libc.so.6
/lib64/ld-linux-x86-64.so.2
libfreebl3.so
libgssapi_krb5.so.2
libkrb5.so.3
libcom_err.so.2
libk5crypto.so.3
libkrb5support.so.0
libkeyutils.so.1
libresolv.so.2
libselinux.so.1
```

### Step 4 Setup Nginx web server (SSL + HTTP2)

For using http2 you need to have certificate (SSL) installed

#1. Go to nginx global file directory

```
sudo su
cd /etc/nginx/
vi nginx.conf
```

#2. Config the file as below 

```
user  nginx;
worker_processes auto;
pid   /var/run/nginx.pid;
worker_rlimit_nofile 100000;
error_log /var/log/nginx/error.log crit;

events {
    worker_connections 1024; 
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    types_hash_max_size 2048; #host bucket
    reset_timedout_connection on;
    server_tokens off; #hide server version
    server_name_in_redirect off;
    limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;
    limit_req_zone $binary_remote_addr zone=req_limit_per_ip:10m rate=5r/s;
    open_file_cache max=200000 inactive=20s; 
    open_file_cache_valid 30s; 
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    ##
    # Buffer Limit Settings
    ##
    client_body_buffer_size      128k;
    client_max_body_size         10m;
    client_body_timeout          10s;
    client_header_buffer_size    1k;
    client_header_timeout        10s;
    large_client_header_buffers  2 1k;
    output_buffers               1 32k;
    postpone_output              1460;
    keepalive_timeout            30s;
    keepalive_requests           100000;
    send_timeout                 30s;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ##
    # Logging Settings
    ##
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log error;
    access_log off;
    log_not_found off;

    ##
    # Gzip Settings
    ##
    gzip on;
    gzip_buffers                16 8k;
    gzip_comp_level                 6;
    gzip_disable              "msie6";
    gzip_http_version             1.1;
    gzip_vary                      on;
    gzip_min_length              1000;
    gzip_proxied                  any;
    gzip_types application/x-javascript text/css application/javascript text/javascript text/plain text/xml application/json application/vnd.ms-fontobject application/x-font-opentype application/x-font-truetype application/x-font-ttf application/xml font/eot font/opentype font/otf image/svg+xml;

    ##
    # Proxy Cache
    ##
    proxy_cache_path /var/nginx/cache levels=1:2 keys_zone=one:10m inactive=60m max_size=200m;
    proxy_cache one;
    proxy_cache_min_uses 3;
    proxy_cache_revalidate on;
    proxy_cache_key "$host$request_uri$cookie_user";
    proxy_cache_valid any      1m;
    proxy_cache_valid 200 302 30m;

    ##
    # Global Security
    ##
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";

    ##
    # Virtual Host Configs for multiple size
    ##
    server_names_hash_bucket_size 64;
    include /etc/nginx/sites-enabled/*;
}
```

#3. Save and test nginx config then restart nginx service

```nginx -t
systemctl restart nginx.service
```

#4. Go to sites-available and create the following file to build a block hosting

```
cd sites-available/
vi <domainname>.conf
```

#5. Config file as below

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name <your-domain-name> www.<your-domain-name>;
    return 301 https://printexpress.cloud$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    root /var/www/<web-folder-name>/html;
    index index.html;
    server_name <your-domain-name>;
    charset utf-8;
    error_page  404     /404.html;
    error_page  403     /403.html;
    error_page  405     /404.html;

    location / {
        limit_conn conn_limit_per_ip 10;
        limit_req zone=req_limit_per_ip burst=5 nodelay;
        try_files $uri.html $uri $uri/ =404;
    }

    # cache.appcache, your document html and data
    location ~* \.(?:manifest|appcache|html?|xml|json)$ {
    access_log off;
    add_header Pragma "no-cache";
    add_header Cache-Control "max-age=0, no-cache, no-store, must-revalidate";
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"; # HSTS (ngx_http_headers_module required)
    # add_header Public-Key-Pins 'pin-sha256="VoxDDZJgiz7LBx4LmQjxcuqL3y6du03E3UqsyTIzABg="; pin-sha256=""; max-age=5184000; includeSubDomains' always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdnjs.cloudflare.com https://www.google-analytics.com https://www.gstatic.com/ https://www.google.com/recaptcha/; img-src 'self' data: https://www.google-analytics.com https://www.google.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://cdnjs.cloudflare.com https://www.google.com/recaptcha/; font-src 'self'; child-src https://www.gstatic.com https://www.facebook.com https://s-static.ak.facebook.com; frame-src https://www.google.com/recaptcha/; object-src 'none';";
    }

     # Media: images, icons, video, audio, HTC, CSS, JS
     location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc|woff2|css|js)$ {
     expires 30d;
     access_log off;
     add_header Cache-Control "public";
     }

     # Deny hotlinking
     location ~ .(gif|png|jpe?g)$ {
     valid_referers none blocked printexpress.cloud *.printexpress.cloud;
     if ($invalid_referer) {
        return   403;
       }
     }

     # Deny referal spam
     if ( $http_referer ~* (jewelry|viagra|nude|girl|nudit|casino|poker|porn|sex|teen|babes) ) {
     return 403;
     }

     # Block bad robots
     if ($http_user_agent ~ (agent1|Cheesebot|msnbot|Purebot|Baiduspider|Lipperhey|Mail.Ru|scrapbot) ) {
     return 403;
     }

     # Block download agenta
     if ($http_user_agent ~* LWP::Simple|wget|libwww-perl) {
     return 403;
     }
         # Deny scripts inside writable directories
     location ~* /(img|cache|media|logs|tmp|image|images)/.*.(php|pl|py|jsp|asp|sh|cgi)$ {
     return 403;
     }

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificat
    ssl_certificate /etc/ssl/<yourweb-ssl-folder>/cert.crt;
    ssl_certificate_key /etc/ssl/<yourweb-ssl-folder>/privkey.key;
    ssl_session_timeout 1h;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 4096 bits
    ssl_dhparam  /etc/ssl/<yourweb-ssl-folder>/dhparam.pem;

    # modern configuration. tweak to your needs.
    ssl_protocols TLSv1.2;
    ssl_ecdh_curve secp384r1;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/<yourweb-ssl-folder>/trustchain.crt;

    # Google DNS to resolve domain
    resolver 8.8.8.8 8.8.4.4 valid=360s ipv6=off;
    resolver_timeout 5s;
}
```

#6. Enable config file 

```
cd /etc/nginx/sites-enabled/
ls -s /etc/nginx/sites-available/<domainname>.conf /etc/nginx/sites-enabled/
```

#7. Save and test nginx config then restart nginx service

```nginx -t
systemctl restart nginx.service
```

### Step 5 Install and config certificate (SSL)

Sorry, this section is currently in progress.

Read more about 7 Tips for Faster HTTP/2 Performance
https://www.nginx.com/blog/7-tips-for-faster-http2-performance/

### Step 6 Install and config certificate (SSL)


## PHP7.0, Nginx, MariaDB 10.1 and PHP APC enabled 
(Alternative PHP Cache / Opcode Cache)

<p align="center">
    <img src="https://cdn.rawgit.com/jukbot/secure-centos/master/php7_logo.svg" alt="PHP7"/>
</p>

First add repository into CentOS 7
```
## Remi Dependency on CentOS 7 and Red Hat (RHEL) 7 ##
rpm -Uvh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-8.noarch.rpm

## CentOS 7 and Red Hat (RHEL) 7 ##
rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
```

Then install required packages
```
yum --enablerepo=remi-php70 install php-fpm php-common php-mysqlnd php-cli php-mbstring php-xml php-gd php-json php-curl php-pdo php-opcache php-opcache php-pecl-apcu php-pear
```

Start php-fpm (FastCGI Process Manager)
```
/etc/init.d/php-fpm start ## use restart after update
## OR ##
service php-fpm start ## use restart after update
```

Auto start php-fpm on boot
```
systemctl enable php-fpm.service
```

## Set Up FirewallD

### Introduction

After setting up the bare minimum configuration for a new server, there are some additional steps that are highly recommended in most cases.

### Prerequisites

In this guide, we will be focusing on configuring some optional but recommended components. This will involve setting our system up with a firewall and a swap file.

### Configuring a Basic Firewall

Firewalls provide a basic level of security for your server. These applications are responsible for denying traffic to every port on your server with exceptions for ports/services you have approved. CentOS ships with a firewall called firewalld. A tool called firewall-cmd can be used to configure your firewall policies. Our basic strategy will be to lock down everything that we do not have a good reason to keep open.

The firewalld service has the ability to make modifications without dropping current connections, so we can turn it on before creating our exceptions:
```
sudo systemctl start firewalld
```
Now that the service is up and running, we can use the firewall-cmd utility to get and set policy information for the firewall. The firewalld application uses the concept of "zones" to label the trustworthiness of the other hosts on a network. This labelling gives us the ability to assign different rules depending on how much we trust a network.

In this guide, we will only be adjusting the policies for the default zone. When we reload our firewall, this will be the zone applied to our interfaces. We should start by adding exceptions to our firewall for approved services. The most essential of these is SSH, since we need to retain remote administrative access to the server.

You can enable the service by name (eg ssh-daemon) by typing:
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

If you plan on running a conventional HTTP web server, you will need to enable the http service:
```
sudo firewall-cmd --permanent --add-service=http
```
If you plan to run a web server with SSL/TLS enabled, you should allow traffic for https as well:
```
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
