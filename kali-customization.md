# Kali lab customizations

We can customize the client Kali netboot filesystem from within a `chroot` on the FreeBSD system. This allows us to make changes in place without having to expose the root filesystem as read-write to clients at any time.

## Set up `chroot`

* `kldload linux64`
* `cd /pandorica/kali/2023.1/root`
* Mount all of the Linux special filesystems:
  * `mount -t devfs devfs dev`
  * `mount -o rw,linrdlnk -t fdescfs fdescfs dev/fd`
  * `mount -o mode=1777 -t tmpfs tmpfs dev/shm`
  * `mount -t linprocfs linprocfs proc`
  * `mount -t linsysfs linsysfs proc`
  * `mount -t tmpfs tmpfs tmp`
  * `mount -t tmpfs tmpfs var/tmp`
* in the `chroot`:
  * `systemctl enable ssh`

## Update systemd for read-only netboot

* Add more tmpfs mounts in `/etc/passwd`:

  ```fstab
  tmpfs   /tmp                    tmpfs   rw      0 0
  tmpfs   /var/cache              tmpfs   rw      0 0
  tmpfs   /var/lib                tmpfs   rw      0 0
  tmpfs   /var/lib/binfmts        tmpfs   rw      0 0
  tmpfs   /var/lib/blueman        tmpfs   rw      0 0
  tmpfs   /var/lib/lightdm        tmpfs   rw      0 0
  tmpfs   /var/log                tmpfs   rw      0 0
  tmpfs   /var/tmp                tmpfs   rw      0 0
  ```

* Create mount points for tmpfs filesystems:

  * `mkdir /var/lib/blueman`

* Don't expect read-write access to `/etc` in `/usr/lib/tmpfiles.d`:

  ```diff
  --- credstore.conf.orig 2024-03-14 12:16:50.544238000 +0000
  +++ credstore.conf      2024-03-14 12:15:32.711971000 +0000
  @@ -7,7 +7,7 @@
  
   # See tmpfiles.d(5) for details
  
  -d /etc/credstore 0700 root root
  -d /etc/credstore.encrypted 0700 root root
  +#d /etc/credstore 0700 root root
  +#d /etc/credstore.encrypted 0700 root root
   z /run/credstore 0700 root root
   z /run/credstore.encrypted 0700 root root
  ```

* Don't let expect read-write access to `/root`:

  ```diff
  --- /root/.zshrc.orig 2024-03-14 12:35:13.845428000 +0000
  +++ /root/.zshrc      2024-03-14 12:35:54.756623000 +0000
  @@ -47,9 +47,9 @@
   zstyle ':completion:*:kill:*' command 'ps -u $USER -o pid,%cpu,tty,cputime,cmd'
  
   # History configurations
  -HISTFILE=~/.zsh_history
  -HISTSIZE=1000
  -SAVEHIST=2000
  +HISTFILE=""
  +HISTSIZE=0
  +SAVEHIST=0
   setopt hist_expire_dups_first # delete duplicates first when HISTFILE size exceeds HISTSIZE
   setopt hist_ignore_dups       # ignore duplicated commands history list
   setopt hist_ignore_space      # ignore commands that start with space
  ```

  

## Update base packages

* set up APT:

  * create `/etc/apt/apt.conf.d`: `APT::Cache-Start 251658240;`

  * edit `/var/lib/dpkg/info/libc6:amd64.postinst` to avoid trying to change init runlevel after updating `libc`:

    ```diff
    --- libc6:amd64.postinst.old    2024-03-07 00:39:09.676778000 +0000
    +++ libc6:amd64.postinst        2024-03-07 00:39:15.492663000 +0000
    @@ -181,7 +181,7 @@
         # Restart init.  Currently handles chroots, systemd and upstart, and
         # assumes anything else is going to not fail at behaving like
         # sysvinit:
    -    TELINIT=yes
    +    TELINIT=no
         if ischroot 2>/dev/null; then
             # Don't bother trying to re-exec init from a chroot:
             TELINIT=no
    ```

  * binary patch `/usr/lib/x86_64-linux-gnu/systemd/libsystemd-shared-255.so` in order to work around a Linux/FreeBSD incompatibility with file locking (by patching `take_etc_passwd_lock` to just return 0):

    ```diff
    --- libsystemd-shared-255.so.backup.dump	2024-03-06 22:39:48
    +++ libsystemd-shared-255.so.dump	2024-03-06 22:43:58
    @@ -414625,10 +414625,16 @@
       21965f: 90                           	nop
    
     0000000000219660 <take_etc_passwd_lock>:
    -  219660: f3 0f 1e fa                  	endbr64
    -  219664: 41 54                        	pushq	%r12
    -  219666: 48 89 fe                     	movq	%rdi, %rsi
    -  219669: 48 c7 c1 ff ff ff ff         	movq	$-1, %rcx
    +  219660: 48 c7 c0 00 00 00 00         	movq	$0, %rax
    +  219667: c3                           	retq
    +  219668: 90                           	nop
    +  219669: 90                           	nop
    +  21966a: 90                           	nop
    +  21966b: 90                           	nop
    +  21966c: 90                           	nop
    +  21966d: 90                           	nop
    +  21966e: 90                           	nop
    +  21966f: 90                           	nop
       219670: 31 ff                        	xorl	%edi, %edi
       219672: 55                           	pushq	%rbp
       219673: 48 8d 15 72 72 0b 00         	leaq	750194(%rip), %rdx      # 0x2d08ec
    ```

* `apt update`

* `apt upgrade`

* and maybe `apt dist-upgrade`

* `zfs snapshot zroot/pandorica/kali/2023.1/root@DATE-post-install-update`

## Create users

* create `/usr/lib/sysusers.d/pandorica.conf`:

  ```sh
  #
  # First, the user that students actually log in as
  #
  u l33t 1337:users "l33t h4x0r" /home/l33t /bin/zsh
  m l33t sudo
  m l33t wireshark
  
  #
  # Next, some generic users
  #
  u alice -:users "Alice Aliceson" /home/alice /bin/zsh
  u bob -:users "Bob Balderson" /home/bob /bin/zsh
  u charlie -:users "charlie" /home/charlie /bin/zsh
  u dana -:users "dana" /home/dana /bin/zsh
  u eli -:users "eli" /home/eli /bin/zsh
  u francesca -:users "francesca" /home/francesca /bin/zsh
  u georg -:users "georg" /home/georg /bin/zsh
  
  #
  # Wizardly users for the annual School of Witchcraft and Wizardry
  #
  u harry -:users "harry" /home/harry /bin/zsh
  u hermione -:users "hermione" /home/hermione /bin/zsh
  u ron -:users "ron" /home/ron /bin/zsh
  u draco -:users "draco" /home/draco /bin/zsh
  u luna -:users "luna" /home/luna /bin/zsh
  u mcgonagall -:users "mcgonagall" /home/mcgonagall /bin/zsh
  ```

* in the `chroot`, run `system-sysusers`

* create empty home directories within the chroot's `/home`:

  ```sh
  mkdir -m 700 /home/l33t /media/l33t
  mkdir -m 755 alice bob charlie dana eli francesca georg
  mkdir -m 755 harry hermione ron draco luna mcgonagall
  
  for username in `ls`; do chown $username $username; done
  chgrp users *
  ```

* add `tmpfs` mounts for users in `/etc/fstab`:

  ```sh
  # Actual user:
  tmpfs   /home/l33t              tmpfs   uid=l33t,gid=users,mode=755             0 0
  tmpfs   /media/l33t             tmpfs   uid=l33t,gid=users,mode=755             0 0
  
  # Synthetic users
  tmpfs   /home/alice             tmpfs   uid=alice,gid=users,mode=755            0 0
  tmpfs   /home/bob               tmpfs   uid=bob,gid=users,mode=755              0 0
  tmpfs   /home/charlie           tmpfs   uid=charlie,gid=users,mode=755          0 0
  tmpfs   /home/dana              tmpfs   uid=dana,gid=users,mode=755             0 0
  tmpfs   /home/eli               tmpfs   uid=eli,gid=users,mode=755              0 0
  tmpfs   /home/francesca         tmpfs   uid=francesca,gid=users,mode=755        0 0
  tmpfs   /home/georg             tmpfs   uid=georg,gid=users,mode=755            0 0
  
  # Wizardly users:
  tmpfs   /home/harry             tmpfs   uid=harry,gid=users,mode=755            0 0
  tmpfs   /home/hermione          tmpfs   uid=hermione,gid=users,mode=755         0 0
  tmpfs   /home/ron               tmpfs   uid=ron,gid=users,mode=755              0 0
  tmpfs   /home/draco             tmpfs   uid=draco,gid=users,mode=755            0 0
  tmpfs   /home/luna              tmpfs   uid=luna,gid=users,mode=755             0 0
  tmpfs   /home/mcgonagall        tmpfs   uid=mcgonagall,gid=users,mode=755       0 0
  ```

* create user passwords from [passwords.txt](passwords.txt):

  ```
  chpasswd < passwords.txt
  ```

  (and don't forget to delete passwords.txt afterwards!)

* and now create a back-door professor account with an SSH key:

  * in the chroot: `

## Install proprietary nVidia driver

* the open-source `nouveau` driver locks up sometimes with our network cards

* `apt install nvidia-driver`

* blacklist the open-source `nouveau` driver

  ```sh
  cat <<EOF | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
  blacklist nouveau
  options nouveau modeset=0
  EOF
  ```

* update initramfs:

  * within the `chroot`: `update-initramfs -u` (ignore warnings like `Couldn't resolve device zroot/ROOT/default`)
  * update the TFTP copies: `cp /pandorica/kali/2023.1/root/boot/initrd* /pandorica/tftpboot/kali/`

## Remove unnecessary things

Some default packages will cause problems in a netboot environment; we don't need these anyway

* `apt remove colord system-config-printer tumbler`
* `systemctl disable systemctl disable smartd`

## Tools

### Ubuntu/Kali packages

These tools are very easy to install: just `apt install` them!

| Tool(s)                       | Package(s)                                                   | Notes                                                        | Source                                           |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------ |
| Additional Kali meta-packages | `kali-linux-large kali-tools-crypto-stego kali-tools-database kali-tools-forensics kali-tools-fuzzing kali-tools-passwords kali-tools-reverse-engineering kali-tools-sniffing-spoofing kali-tools-social-engineering kali-tools-voip kali-tools-web` | Many GB of additional tools                                  | Various                                          |
| Audacity                      | `audacity`                                                   | Audio editor                                                 | https://github.com/audacity/audacity             |
| CyberChef                     | `cyberchef`                                                  | Misc. tools from GCHQ for fooling around with encodings, etc. | https://github.com/gchq/CyberChef                |
| DMTX                          | `dmtx-utils`                                                 | Data matrix barcode tools                                    | https://github.com/dmtx/dmtx-utils               |
| EXIF tools                    | `exif`                                                       | JPEG image metadata tool                                     | https://github.com/libexif/exif                  |
| FFMpeg                        | `ffmpeg`                                                     | Video transcoder and Swiss Army knife                        | https://github.com/FFmpeg/FFmpeg                 |
| Ghidra                        | `ghidra`                                                     | Software Reverse Engineering tool from NSA                   | https://github.com/NationalSecurityAgency/ghidra |
| Hex editors                   | `hexedit hexer ht ncurses-hexedit wxhexeditor tweak`         | Various hex editors                                          | Various                                          |
| LibreOffice                   | `libreoffice`                                                | Office suite                                                 | https://github.com/LibreOffice                   |
| LRZip                         | `lrzip`                                                      | Compression tool for large files                             | https://github.com/ckolivas/lrzip                |
| PulseView                     | `pulseview`                                                  | Waveform viewer                                              | https://sigrok.org/wiki/PulseView                |
| PwnTools                      | `python3-pwntools`                                           | Python library for exploit development                       | https://github.com/Gallopsled/pwntools           |
| System status viewers         | `bpytop btm htop`                                            | Command-line status viewers that look nicer than `top`       | Various                                          |
| Terminals and shells          | `alacritty fish xfce4-terminal`                              | Optional: set Alacritty as default terminal with `update-alternatives --set x-terminal-emulator /usr/bin/alacritty` | Various                                          |
| VLC                           | `vlc`                                                        | VideoLAN Client: media player                                | https://github.com/videolan/vlc                  |
| ZBar                          | `zbar`                                                       | Barcode tools                                                | https://github.com/mchehab/zbar                  |
| Other command-line tools      | `bat eza ripgrep sd`                                         | Nicer versions of `cat`, `ls`, `grep` and `sed`              | Various                                          |

### Other tools

These tools are a bit more complicated to install

#### 010editor

* `zfs create zroot/pandorica/software`

* fetch https://www.sweetscape.com/download/010EditorLinux64Installer.tar.gz

* untarr'ed installer bundle to `/usr/local/010editor`

* create `/usr/share/applications/010editor.desktop`:

  ```ini
  [Desktop Entry]
  Version=1.0
  Type=Application
  Name=010 Editor
  Exec=/usr/local/010editor/010editor
  Icon=/usr/local/010editor/010_icon_128x128.png
  ```

* `mkdir /etc/skel/Desktop`
* `cp /usr/share/applications/010editor.desktop /etc/skel/Desktop/`

#### Bless hex editor

* TODO: needs to be built from source?

#### Zoo

* This is an historic archiving tool
* Build from https://github.com/troglobit/zoo

## GUI

* Background images

  * Add [IntroSec](IntroSec) at `/usr/share/backgrounds/IntroSec` 

  * Desktop wallpaper: symlink `/usr/share/background/kali-16x9/default` to `../IntroSec/stairwell.png`

  * LightDM wallpaper: `/etc/lightdm/lightdm-gtk-greeter.conf`

    ```ini
    [greeter]
    background=/usr/share/backgrounds/IntroSec/csf-grainy-red.png
    [... customize other stuff if you like ...]
    theme-name=Adwaita-dark
    icon-theme-name=Flat-Remix-Purple-Dark
    
    [...]
    screensaver-timeout=1800
    ```

* Firefox

  * Run on startup: `/etc/xdg/autostart/Firefox.desktop`
    ```ini
    [Desktop Entry]
    Name=Firefox
    Exec=/usr/lib/firefox-esr/firefox-esr
    Terminal=false
    Type=Application
    Icon=firefox-esr
    StartupWMClass=Firefox-esr
    StartupNotify=true
    X-XFCE-Source=file:///usr/share/applications/firefox-esr.desktop
    ```

  * Customize home page and remove things that won't work from the "new tab" page: `/usr/share/firefox-esr/distribution/policies.json`

    ```diff
    --- policies.json.backup        2024-03-07 02:40:21.789423000 +0000
    +++ policies.json       2024-03-07 02:40:49.217592000 +0000
    @@ -5,7 +5,7 @@
         "OverrideFirstRunPage": "",
         "OverridePostUpdatePage": "",
         "Homepage": {
    -      "URL": "file:///usr/share/kali-defaults/web/homepage.html",
    +      "URL": "http://10.0.0.1/",
           "Locked": false,
           "StartPage": "homepage"
         },
    @@ -16,8 +16,8 @@
         },
         "CaptivePortal": false,
         "FirefoxHome": {
    -      "Search": true,
    -      "TopSites": true,
    +      "Search": false,
    +      "TopSites": false,
           "Highlights": false,
           "Pocket": false,
           "Snippets": false,
    ```
    

* Update `/etc/xdg/xfce4/helpers.rc`:
  
  ```diff
  --- helpers.rc.original 2024-03-07 00:49:37.612731000 +0000
  +++ helpers.rc  2024-03-13 17:01:18.881549000 +0000
  @@ -6,5 +6,5 @@
  
   WebBrowser=debian-sensible-browser
   MailReader=thunderbird
  -TerminalEmulator=debian-x-terminal-emulator
  +TerminalEmulator=alacritty
   FileManager=thunar
  ```
  
* Update `/etc/xdg/xfce4/xfconf/xfce-perchannel-xml`:
  
  ```diff
  --- xsettings.xml.original      2023-07-06 16:35:02.000000000 +0000
  +++ xsettings.xml       2023-01-08 05:26:57.000000000 +0000
  @@ -6,8 +6,8 @@
   <?xml version="1.0" encoding="UTF-8"?>
   <channel name="xsettings" version="1.0">
     <property name="Net" type="empty">
  -    <property name="ThemeName" type="string" value="Xfce"/>
  -    <property name="IconThemeName" type="string" value="Tango"/>
  +    <property name="ThemeName" type="string" value="Kali-Purple-Dark"/>
  +    <property name="IconThemeName" type="string" value="gnome"/>
       <property name="DoubleClickTime" type="int" value="400"/>
       <property name="DoubleClickDistance" type="int" value="5"/>
       <property name="DndDragThreshold" type="int" value="8"/>
  @@ -19,9 +19,9 @@
     </property>
     <property name="Xft" type="empty">
       <property name="DPI" type="empty"/>
  -    <property name="Antialias" type="int" value="-1"/>
  -    <property name="Hinting" type="int" value="-1"/>
  -    <property name="HintStyle" type="string" value="hintnone"/>
  +    <property name="Antialias" type="int" value="1"/>
  +    <property name="Hinting" type="int" value="1"/>
  +    <property name="HintStyle" type="string" value="hintfull"/>
       <property name="RGBA" type="string" value="none"/>
       <!-- <property name="Lcdfilter" type="string" value="none"/> -->
     </property>
  @@ -47,6 +48,9 @@
       <property name="TitlebarMiddleClick" type="string" value="lower"/>
     </property>
     <property name="Gdk" type="empty">
  -    <property name="WindowScalingFactor" type="int" value="1"/>
  +    <property name="WindowScalingFactor" type="int" value="2"/>
  +  </property>
  +  <property name="Xfce" type="empty">
  +    <propery name="SyncThemes" type="bool" value="true"/>
     </property>
   </channel>
  ```
  
* Update `/etc/xdg/xfce4/xfconf/xfce-perchannel-xml/xfwm4.xml`:
  
  ```diff
  --- xfwm4.xml.original  2024-03-13 17:22:50.506721000 +0000
  +++ xfwm4.xml   2024-03-13 17:23:07.839814000 +0000
  @@ -3,7 +3,7 @@
   <channel name="xfwm4" version="1.0">
     <property name="general" type="empty">
       <property name="button_layout" type="string" value="O|HMC"/>
  -    <property name="theme" type="string" value="Kali-Dark"/>
  +    <property name="theme" type="string" value="Kali-Dark-xHiDPI"/>
       <property name="title_font" type="string" value="Cantarell Bold 9"/>
       <property name="easy_click" type="string" value="Super"/>
       <property name="workspace_count" type="int" value="4"/>
  ```
  
* edit `/etc/xdg/xfce4/panel/default.xml`:
  
  ```diff
  --- default.xml.original        2024-03-14 12:13:10.732195000 +0000
  +++ default.xml 2024-03-14 12:13:38.135029000 +0000
  @@ -139,7 +139,6 @@
         <property name="square-icons" type="bool" value="true"/>
         <property name="symbolic-icons" type="bool" value="false"/>
       </property>
  -    <property name="plugin-15" type="string" value="genmon"/>
       <property name="plugin-16" type="string" value="pulseaudio"/>
       <property name="plugin-17" type="string" value="notification-plugin"/>
       <property name="plugin-18" type="string" value="power-manager-plugin"/>
  ```
  
  
  
* Other eye candy
  
  * `apt install cowsay neo-cli`
