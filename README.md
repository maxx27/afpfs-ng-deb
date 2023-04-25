
# About

Here some notes about attaching Apple TimeCapsule disk to Ubuntu 23.04.

There are different ways to do it:

- mount using afpfs-ng
- mount using samba1
  - need kernel < 5.14, or
  - rebuild kernel with reverted commit
- use `netatalk` as client

Some files are cached in repo. Thus you may use saved copies of these files.

Any pull request to update information are welcome 😊

# Disclaimer

This is mostly my internal notes. So they are partly incomplete. But it is good point to start.

# afpfs-ng

Ubuntu doesn't have the `afpfs-ng` since v14.

```
$ apt-file search afp_client
<empty>
```

Nothing is found also: https://packages.ubuntu.com/search?suite=bionic&arch=any&mode=filename&searchon=contents&keywords=afp_client


## Build from sources

Credits to https://superuser.com/a/1089747/439273

```sh
sudo apt-get install g++ libfuse-dev libreadline-dev libncurses5-dev git

git clone https://github.com/simonvetter/afpfs-ng
cd afpfs-ng/
./configure && make && sudo make install
sudo ldconfig
```

But:

1. This failed using `g++12`. I have no time to adapt sources or installed older compiler.
2. I like to use packages (e.g. to uninstall quickly or manage dependencies)

So see next section.

## Use package from Arch

You may use package from ArchLinux:

- Open https://archlinux.org/packages/community/x86_64/afpfs-ng/
  - Click on the link `Download From Mirror` (right upper corner)
- or, `wget https://mirror.osbeck.com/archlinux/community/os/x86_64/afpfs-ng-0.8.2-3-x86_64.pkg.tar.zst`

> [!note]
> I use `amd64` architecture. If your architecture is another than `amd64`, so please adopt process for it.

Build deb package:

```sh
mkdir -p afpfs-ng/DEBIAN
tar xf afpfs-ng-0.8.2-3-x86_64.pkg.tar.zst -C afpfs-ng/

cat <<EOF > afpfs-ng/DEBIAN/control
Package: afpfs-ng
Architecture: amd64
Maintainer: m.suslov
Priority: optional
Version: 0.8.2-3
Description: A client for the Apple Filing Protocol (AFP)
EOF
```

> [!note]
> I didn't find any dependencies. May be all necessary was already installed in my system. Thus I omitted the following section:
> `Depends: package1,package2`

Build deb package:

```sh
dpkg-deb --build afpfs-ng
```

Install:

```sh
sudo dpkg -i afpfs-ng.deb
```

Man can be found [online](https://linux.die.net/man/1/mount_afp) or offline:

```sh
man mount_afp
```

## Mount as root

```sh
sudo mount_afp afp://myuser:mypass@mycapsule/Data /media/airport
sudo umount /media/airport
```

## Mount as user

Uncomment in `/etc/fuse.conf`:

```diff
-#/etc/fuse.conf
+/etc/fuse.conf
```

otherwise you get

```
fusermount: option allow_other only allowed if 'user_allow_other' is set in /etc/fuse.conf
```

and mount:

```sh
sudo mkdir -p /media/airport
sudo chown $(id -u):$(id -g) /media/airport
sudo mount_afp -o user=maxim afp://myuser:mypass@mycapsule/Data /media/airport
```


# Use samba v1

Unfortunately, Linux 5.15 [drops support for NTLM](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=76a3c92ec9e0668e4cd0e9ff1782eb68f61a179c).

Thus, you may
- use kernel < 5.15
- recompile with samba v1 support

## Older kernel

```sh
sudo mount -rw -t cifs //mycapsule/data /media/airport/ -o username=myuser,password=mypass,sec=ntlm,vers=1.0 --verbose
```

or use file with credentials

```
cat <<EOF > ~/cifs.credo
username=myuser
password=mypass
domain=
EOF
chmod 600 ~/cifs.credo
sudo mount -rw -t cifs //mycapsule/data /media/airport/ -o credentials=$HOME/cifs.credo,sec=ntlm,vers=1.0 --verbose
```

Maybe you need the following options as well:

- `iocharset=utf8`

## Add reverted changes to a newest kernel

So I needed NTLM support added back into the kernel, so I created a backport patch for vanilla kernel 5.15.12
I don't use arch, but I posted the patch over here: [https://821895.bugs.gentoo.org/attachment.cgi?id=761299](https://821895.bugs.gentoo.org/attachment.cgi?id=761299)

From [mount.cifs sec=ntlm became bad option  Newbie Corner  Arch Linux Forums](https://bbs.archlinux.org/viewtopic.php?id=271264)

# netatalk

I haven't try this solution, but it may work. Idea is to connect as afp client.

See https://netatalk.sourceforge.io/3.1/htmldocs/

# Troubleshooting

## Unknown network name

In my case, network name `mycapsule` isn't resolve via `nslookup` or `dig`.

I added a line to `/etc/hosts`:

```diff
+192.168.2.4  mycapsule
```

IP address can be found in AirPort Time Capsule application.

## Case of shared folder

Name is case-sensitive:

```console
$ sudo mount_afp afp://myuser:mypass@mycapsule/data /media/airport
The afpfs daemon does not appear to be running for uid 0, let me start it for you
Mounting mycapsule from data on /media/airport
Volume data does not exist on server MyCapsule.
Choose from: Data, myuser

$ sudo mount_afp afp://myuser:mypass@mycapsule/Data /media/airport
Mounting mycapsule from Data on /media/airport
Mounting of volume Data of server MyCapsule succeeded.
```
