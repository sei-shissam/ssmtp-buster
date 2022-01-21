# ssmtp-buster
Build Ssmtp on Raspbian and Debian Buster

A once popular smtp agent, [sSMTP](https://wiki.debian.org/sSMTP), is no longer maintained for debian-based systems (Buster and later), including Raspbian. Included in this repo are additional dpkg build files and patches to the [debian package repo for sSMTP](https://salsa.debian.org/debian/ssmtp) so that it can be built for Buster (and later).

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

In this repo, are the changes I made to the Debian GitLab instance of sSMTP to take Erik's steps and integrate that along with the configuration changes to allow building sSMTP using ```dpkg-buildpackage``` to build either sSMTP with the original GnuTLS (using ```--enable-gnutls```) or OpenSSL (using ```--enable-openssl```) (but not both).

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
# now the choice, to build with the default, Debian
# selected GnuTLS, simply
#
dpkg buildpackage -b
#
# alternatively, to build with OpenSSL, simply
#
dpkg buildpackage --rules-file=debian/rules.openssl
#
```

#### If necessary, resolved dependencies

There can only be one debian/control file, so it is necessary to install both GnuTLS and OpenSSL devs otherwise edit debian/control and remove the one not needed

```
apt-get install po-debconf libgnutls-openssl-dev libssl-dev
```

### Install sSMTP as you see fit

Original install for sSMTP was inspired from this post. Therefore installing from the prescribed patch/build went this way:

```

```
