---
gallery_installer_menu:
  - url: /assets/images/12/base-system/installer_menu.png
    image_path: /assets/images/12/base-system/installer_menu.png
    alt: "installer menu"

gellery_select_location:
  - url: /assets/images/12/base-system/select_location.png
    image_path: /assets/images/12/base-system/select_location.png
    alt: "select location"

gallery_partition:
  - url: /assets/images/12/base-system/partition.png
    image_path: /assets/images/12/base-system/partition.png
    alt: "partition"

gallery_software_selection:
  - url: /assets/images/12/base-system/software_selection.png
    image_path: /assets/images/12/base-system/software_selection.png
    alt: "software selection"

gallery_grub_install:
  - url: /assets/images/12/base-system/grub_install.png
    image_path: /assets/images/12/base-system/grub_install.png
    alt: "GRUB install"
---
# Debian 12 "bookworm" installation

{% include version_warning.html %}

## Installer configuration

### Installer menu

Choose "Install" instead of "Graphical install." The keyboard should be enough on simple screens.

{% include figure popup=true image_path="/assets/images/12/base-system/installer_menu.png" alt="installer menu" %}

### Select a language

Choose "English - English". As described on the screen, this will be the system's default language. English will be the safest choice in an emergency (e.g., accessing from a very limited environment).

### Select your location

This will determine the system timezone. This doesn't have to be where the actual server is located.  
If your choice is not in the first list, you can find more options from "other" at the bottom.

{% include figure popup=true image_path="/assets/images/12/base-system/select_location.png" alt="select location" %}

### Configure locales

You will be asked to choose one if the installer can't automatically guess the locale. For example, language: English and locale: Japan is not a normal combination, and the system will show this screen.  
"en_??.UTF-8" is recommended as a safe choice.

### Configure the keyboard

Choose the keyboard layout you're using now for installation.

## Configure the network

If DHCP is on, the installer will autoconfigure the network. However, the server should have a static IP, and manual configuration should be required.

1. IP address  
   It accepts both IPv4 and IPv6. If the server has both v4 and v6 addresses, set v4 here and add v6 after the installation.
2. Netmask  
   It should be something like `255.255.255.0` or `255.255.254.0`.
3. Gateway  
   Configure the IP provided by your server provider.
4. Name server addresses  
   If there are secondary DNS, list all of them separated by a space, not a comma.  
   `192.168.0.10 192.168.0.11`
5. Hostname  
   The first part of the server's FQDN. The "hostname" part of `hostname.example.com`.
6. Domain name  
   The "example.com" part of `hostname.example.com`.

## Set up users and passwords

Set the root password and create a new user.

1. Root password  
   "root" is the administrator with all privileges. This password must be kept secret and should not be used often, but never forget it.
2. Full name for the new user  
   This doesn't have to be your real name. Anything goes.
3. User name for your account  
   This will be the actual account name, which will be your primary mail address. (You can add aliases for email, though.)
4. Choose a password for the new user  
   This password should be different from the root user. This will be required when you `sudo`.

## Partition disks

"Guided - user entire disk" should work fine for most cases.  
In my case, I used Manual configuration to consider the following factors.

- More swap to cover the lack of main memory  
  Due to the small VPS, there is insufficient memory for peak usage. A larger swap file will be required.
- XFS for MongoDB  
  MongoDB requires an XFS filesystem for data storage. Instead of making a partition, you can use the loopback mount to make an XFS area on an ext4 filesystem.

{% include figure popup=true image_path="/assets/images/12/base-system/partition.png" alt="partition" %}

## Configure the package manager

Choose the country nearest to the server location.

1. Scan extra installation media  
   Choose "No" to continue. Required packages should be downloaded from Debian mirrors.
2. Debian archive mirror country  
   You must consider where the physical server is located because it should access the nearest Debian mirrors.
3. Debian archive mirror  
   There will be a list of Debian mirrors in the country you choose. Choose the one that is suitable to your network environment.  
   (In Japan, `ftp.jp.debian.org` is the recommended CDN mirror unless you're in the same network as one of the listed mirrors.)
4. HTTP proxy  
   This will be required only if you're in the restricted network.

## Configuring popularity-contest

If you're ok, choose "Yes" to provide the package usage anonymized data to help Debian developers.

## Software selection

Choose nothing (unselect all). Even without "standard system utilities", it will still install "required" utilities.  
You can check what is in "standard system utilities" with this command.  
`tasksel --task-packages standard`

{% include figure popup=true image_path="/assets/images/12/base-system/software_selection.png" alt="software selection" %}

## Configuring grub-pc

The "server" should have only one OS, Debian. GRUB boot loader should be installed into the primary drive.

Unless you're doing something uncommon, you should have only one option (device) to install GRUB.

{% include figure popup=true image_path="/assets/images/12/base-system/grub_install.png" alt="GRUB install" %}

## Finish the installation

Install is complete. Continue to finish the installation and remove the install media before rebooting.
