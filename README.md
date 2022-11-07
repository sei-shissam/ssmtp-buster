# ssmtp-buster
Build sSMTP on Raspbian and Debian Buster

A once popular smtp agent, [sSMTP](https://wiki.debian.org/sSMTP), is no longer maintained for debian-based systems (Buster and later), including Raspbian. Included in this repo are additional ```dpkg``` build files and patches to the [debian package repo for sSMTP](https://salsa.debian.org/debian/ssmtp) so that sSMTP can be built for Buster (and later).

## Problem:

On a fresh install of Raspbian Buster, sSMTP would fail to work with ```gmail.com``` generating logs in ```syslog``` appearing like:

```
Jan 12 08:32:55 pi sSMTP[886]: Creating SSL connection to host
Jan 12 08:32:55 pi sSMTP[886]: SSL connection using ECDHE_RSA_AES_256_GCM_SHA384
Jan 12 08:32:55 pi sSMTP[886]:  (mail.<redacted>)
```

A search will lead to [this post](https://forums.raspberrypi.com/viewtopic.php?t=247108) which is similar in nature to others which report the same problem. The pattern is the same:
* the ```syslog``` entry above,
* trying to hit ```gmail.com``` (other servers may be listed as well),
* all stating that as of Debian Buster, sSMTP is no longer maintained with the recommendation to move to `msmtp`.

Looking into the original code and the initial Debian distribution, there was a Debian patch to the original source code which switched from [OpenSSL](https://en.wikipedia.org/wiki/OpenSSL) to [GnuTLS](https://en.wikipedia.org/wiki/GnuTLS) (see [```debian/patches/01-374327-use-gnutls.patch```](https://salsa.debian.org/debian/ssmtp)).

A quick look at the binary of ```/usr/sbin/ssmtp``` confirmed that the binary dpkg distribution was certainly using ```GnuTLS```.

```
$ ls -l /usr/sbin/ssmtp; (ldd /usr/sbin/ssmtp | grep ssl)
-rwsr-xr-x 1 ssmtp root 30588 Jul 20  2014 /usr/sbin/ssmtp
	libgnutls-openssl.so.27 => /usr/lib/arm-linux-gnueabihf/libgnutls-openssl.so.27 (0x76efb000)
```

## Solution:

Switch back to OpenSSL!

Inspired by this [post at Erik's Online Corner](https://espinoza.tv/post/ssmtp-buster/) and using the latest (and last?) [from the Debian repo here](https://salsa.debian.org/debian/ssmtp.git), sSMTP was recompiled with all the debian patches to use OpenSSL rather than GnuTLS. It worked.

```
$ ls -l /usr/sbin/ssmtp; (ldd /usr/sbin/ssmtp | grep ssl)
-rwsr-xr-x 1 ssmtp root 109632 Jan 20 16:54 /usr/sbin/ssmtp
	libssl.so.1.1 => /lib/arm-linux-gnueabihf/libssl.so.1.1 (0xb6ed6000)
```

In this repo, are the changes made to the Debian GitLab instance of sSMTP to take Erik's steps and integrate that along with the configuration changes to allow building sSMTP using ```dpkg-buildpackage``` to build either sSMTP with the original GnuTLS (using ```--enable-gnutls```) or OpenSSL (using ```--enable-openssl```) (but not both).

## Patch/Build Recipe:

Here are the susscinct steps which have been tested on Raspbian Stretch, Raspbian Buster, and Ubuntu 18.04 Bionic Buster.

```
cd ~/src
#
# grab these patches for building under Buster
#
git clone https://github.com/shissam/ssmtp-buster.git
#
# grab the Debian sSMTP repo
#
git clone https://salsa.debian.org/debian/ssmtp.git
#
cd ssmtp
#
# these patches are based on the following commit revision
#
git log | head -3
commit d4631356318ae3f78f15bd55eeae295dec2a48e2
Author: Jelmer Vernooĳ <jelmer@jelmer.uk>
Date:   Wed Feb 3 12:30:25 2021 +0000
#
# copy the changes and new files from the patches repo
cd ../ssmtp-buster
tar cf - * | (cd ../ssmtp/ ; tar xfp -)
cd ../ssmtp
#
```
#### Now choose to build either with GnuTLS or OpenSSL
```
#
# to select GnuTLS (the default for Debian, which fails under Buster)
#
dpkg-buildpackage -b
#
# check if GnuTLS is linked in
ldd ./ssmtp|grep ssl
	libgnutls-openssl.so.27 => /lib/arm-linux-gnueabihf/libgnutls-openssl.so.27 (0xb6f08000)
```
Alternatively
```
#
# to select OpenSSL (the reason for this repo)
# (see warning below about broken versions of dpkg-buildpackage)
#
dpkg-buildpackage --rules-file=debian/rules.openssl -b
#
# check if GnuTLS is linked in
ldd ./ssmtp|grep ssl
	libssl.so.1.1 => /lib/arm-linux-gnueabihf/libssl.so.1.1 (0xb6eff000)
#
```
#### If necessary, resolve dependencies

There can only be one ```debian/control``` file, so it is necessary to install both GnuTLS and OpenSSL devs otherwise edit ```debian/control``` and remove the one not needed

```
apt-get install po-debconf libgnutls-openssl-dev libssl-dev
```

### Install sSMTP as you see fit

Original install for sSMTP was inspired from [this post](https://wiki.freebsd.org/Ports/mail/ssmtp). Therefore installing from the prescribed patch/build went this way (using ```sudo```):

```
#
# add a sSMTP pseudo-user to protect information in ```ssmtp.conf```
#
useradd -g nogroup -d /home/ssmtp -s /usr/sbin/nologin -c "sSMTP pseudo-user" ssmtp
#
# copy newly created files to their place (see ```make -n install``` for help)
#
cp -i ssmtp /usr/sbin/ssmtp 
mkdir -p /etc/ssmtp
#
# don't accidently overwrite previous created file config files
#
# cp -i revaliases /etc/ssmtp/
# cp -i ssmtp.conf /etc/ssmtp/
#
# set the file modes correctly to work with the sSMTP pseudo-user
#
chown ssmtp.root /usr/sbin/ssmtp /etc/ssmtp/revaliases /etc/ssmtp/ssmtp.conf
chmod 4755 /usr/sbin/ssmtp
chmod 644 /etc/ssmtp/revaliases
chmod 400 /etc/ssmtp/ssmtp.conf
#
# do the man pages
#
cp -i debian/tmp/usr/share/man/man5/ssmtp.conf.5.gz /usr/share/man/man5/ssmtp.conf.5.gz
cp -i debian/tmp/usr/share/man/man8/ssmtp.8.gz /usr/share/man/man8/ssmtp.8.gz
cp -i debian/tmp/usr/share/man/man8/newaliases.8.gz /usr/share/man/man8/newaliases.8.gz
cp -i debian/tmp/usr/share/man/man8/mailq.8.gz /usr/share/man/man8/mailq.8.gz
ln -s ssmtp.8.gz /usr/share/man/man8/sendmail.8.gz
#
# if this is the first install you may need to run ```generate_config```
# https://www.techrepublic.com/blog/it-security/use-ssmtp-to-send-e-mail-simply-and-securely/ provides a good explaination of ```ssmtp.conf``` and ```revaliases```
#
./generate_config /etc/ssmtp/ssmtp.conf
```

## Warnings

### dpkg-buildpackage version 1.19.0.5 on Ubuntu 18.04.6 LTS

There is a mistake in command line parsing (which has been fixed downstream) where the command line arg ```--rules-file=``` is not correct in ```/usr/bin/dpkg-buildpackage```. Here is a ```diff``` to show the repair which may need to be made:

```
diff -Naur /usr/bin/dpkg-buildpackage /tmp/dpkg-buildpackage 
--- /usr/bin/dpkg-buildpackage	2019-09-05 21:05:14.000000000 +0000
+++ /tmp/dpkg-buildpackage	2022-01-21 21:19:38.471797357 +0000
@@ -343,7 +343,7 @@
     } elsif (m/^-[EW]$/) {
 	# Deprecated option
 	warning(g_('-E and -W are deprecated, they are without effect'));
-    } elsif (/^-R(.*)$/ or /^--rules-target=(.*)$/) {
+    } elsif (/^-R(.*)$/ or /^--rules-file=(.*)$/) {
 	my $arg = $1;
 	@debian_rules = split ' ', $arg;
     } else {
```
### sSMTP is fill with bugs

Just because sSMTP can be made to work with Buster does not necessarily mean that it is safe or better. Do check out the [list of bugs captured at Debian here](https://bugs.debian.org/cgi-bin/pkgreport.cgi?pkg=ssmtp;dist=unstable).

### sSMTP debug

Speaking of bugs, be careful, to diagnose problems and running sSMTP with verbose debug (e.g., ```ssmtp -v -d9```) will result in your secrets in ```ssmtp.conf``` to be spilled to log files (e.g., ```/var/log/syslog```).

## TODOs
