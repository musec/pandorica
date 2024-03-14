# Pandorica host setup

We run our lab environment on FreeBSD for its ZFS support and ease of administration. From this environment, we can `chroo`t into client netboot filesystems to make changes without having to actually spin up any clients. However, on clients that use systemd, we may need some workarounds (see [Kali customizations](kali-customizations.md)).

## Create ZFS filesystems

Choose a name and mountpoint for the lab environment's netboot data. For example, in these config files, we will use the ZFS filesystem `zroot/pandorica` and the mount point `/pandorica`.

* `zfs create -o mountpoint=/pandorica zroot/pandorica`
* `zfs create zroot/pandorica/kali`
* `zfs create zroot/pandorica/tftpboot`

## Extract netboot client OS

* Create ZFS filesystems to hold Kali (e.g., `kali/2023.1`):
  * `zfs create zroot/pandorica/kali/2023.1`
  * `zfs create zroot/pandorica/kali/2023.1/installer`
  * `zfs create zroot/pandorica/kali/2023.1/root`
  
* Extract live/installer ISO
  * e.g., https://old.kali.org/kali-images/kali-2023.1/kali-linux-2023.1-live-amd64.iso
  * `cd /pandorica/kali/2023.1/installer`
  * `tar xvf ../kali-linux-2023.1-live-amd64.iso`
  
* Extract live image
  * `pkg install squashfs-tools-ng`
  * `unsquashfs -dest ../root live/filesystem.squashfs`
  * `zfs snapshot zroot/pandorica/kali/2023.1/root@DATE-fresh-install`
  
* Set hostname within client root (stops `sudo` warnings):
  * `/etc/hostname`: `h4x0r.pandorica.ece.engr.mun.ca`
  
  * `/etc/hosts`:
  
    ```hosts
    127.0.0.1 h4x0r.pandorica.engr.mun.ca
    ::1 h4x04.pandorica.engr.mun.ca
    ```

* Set up tmpfs for various system directories in `/etc/fstab`:

  ```fstab
  tmpfs   /tmp                    tmpfs   rw      0 0
  tmpfs   /var/cache              tmpfs   rw      0 0
  tmpfs   /var/lib                tmpfs   rw      0 0
  tmpfs   /var/lib/blueman        tmpfs   rw      0 0
  tmpfs   /var/lib/lightdm        tmpfs   rw      0 0
  tmpfs   /var/log                tmpfs   rw      0 0
  tmpfs   /var/tmp                tmpfs   rw      0 0
  ```

* set up `/etc/skel` by un-tar'ing [skel.tar](skel.tar)

## Create PXE boot environment

* `pkg install tftp-hpa`
* update `/etc/rc.conf`: 

  ```sh
  tftpd_enable="YES"
  tftpd_flags="--secure --address 0.0.0.0:69 /pandorica/tftpboot"
  ```

* acquire netbook-enabled GRUB:

  * `cd /pandorica/tftpboot`
  * `fetch http://archive.ubuntu.com/ubuntu/dists/jammy/main/uefi/grub2-amd64/current/grubnetx64.efi.signed`
  
* copy kernel boot files:

  * `mkdir kali`
  * `cp ../kali/2023.1/root/boot/initrd* kali/`
  * `cp ../kali/2023.1/root/boot/vmlinuz* kali/`
  * you must remember to **repeat these `cp` operations** after **every kernel and initramfs update**

* create `/pandorica/tftpboot/grub/grub.cfg`:

  ```sh
  set timeout=5
  
  menuentry "Kali 2023.1" {
      net_dhcp
      linux kali/vmlinuz-6.1.0-kali5-amd64 root=/dev/nfs ip=dhcp nfsroot=10.0.0.1:/pandorica/kali/2023.1/root ro nosplash nvidia-drm.modeset=1
      initrd kali/initrd.img-6.1.0-kali5-amd64
  }
  ```

## Set up NFS server

* `zfs set sharenfs="ro,maproot=root,network=10.0.0.0,mask=255.255.255.0" zroot/pandorica/kali/2023.1/root`

* edit `/etc/rc.conf`: 

  ```sh
  nfs_client_enable="YES"
  nfs_server_enable="YES"
  nfs_server_flags="-h 10.0.0.1 -u -t"
  mountd_enable="YES"
  mountd_flags="-h 10.0.0.1"
  rpcbind_enable="YES"
  ```

* start NFS server: `service nfsd start`

## DHCP

* `pkg install isc-dhc44-server`
* update `/etc/rc.conf`: 

  ```sh
  dhcpd_ifaces="bnxt0"
  dhcpd_withumask="022"
  dhcpd_chuser_enable="YES"
  dhcpd_withuser="dhcpd"
  dhcpd_withgroup="dhcpd"
  dhcpd_chroot_enable="YES"
  dhcpd_devfs_enable="YES"
  dhcpd_rootdir="/var/db/dhcpd"
  ```

  (but don't use `dhcpd_enable`, as we don't want to risk starting dhcpd when LabNet is running)

* create `/usr/local/etc/dhcpd.conf`:

  ```dhcpd
  server-name "gibson.pandorica.engr.mun.ca";
  option domain-name "pandorica.engr.mun.ca";
  option domain-name-servers 10.0.0.1;
  
  option architecture-type code 98 = unsigned integer 16;
  option root-opts code 130 = string;  # NFS / mount options
  
  option subnet-mask 255.255.255.0;
  default-lease-time 600;
  max-lease-time 7200;
  
  subnet 10.0.0.0 netmask 255.255.255.0 {
          authoritative;
          allow unknown-clients;
  
          server-identifier 10.0.0.1;
          next-server 10.0.0.1;
          option broadcast-address 10.0.0.255;
          option routers 10.0.0.1;
    
          range 10.0.0.100 10.0.0.199;
          allow booting;
          option root-path "/pandorica/kali/2023.1/root";
    
          filename "grubnetx64.efi.signed";
  }
  ```

# Web server

* `pkg install nginx`
* TODO: stuff

# CTF server

## CTFd

This is what we've been using for our first couple of CTFs, but it doesn't seem terribly well-maintained: CTFd has a requirement (pybluemonday) that depends on a Go library (bluemonday), but pybluemonday doesn't seem to build with recent versions of Go due to the way it uses `go get` out of tree in its build process.

* `git clone https://github.com/CTFd/CTFd`
* `pkg install cmake go python py39-cffi py39-pillow py39-pip rust`
* `pip install wheel`
* Set up venv:
  * `python -m venv venv`
  * `source venv/bin/activate.YOURSHELL`
  * `pip install -r requirements.txt`
* **TODO:** figure out (or work around) pybluemonday issue

### RootTheBox

* `git clone https://github.com/moloch--/RootTheBox`
* `cd RootTheBox`
* `pkg install mariadb 1011-client`
* Set up venv:
  * `python -m venv venv`
  * `source venv/bin/activate.YOURSHELL`
  * `pip install -r setup/requirements.txt`
* Create config:
  * `./rootthebox.py --save`
  * TODO
