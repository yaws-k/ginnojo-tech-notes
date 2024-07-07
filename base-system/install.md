# Debian 12 "bookworm" installation

Explanation of installation.

## Installer configuration

1. Installer menu  
   Choose "Install" instead of "Graphical install". Keyboard should be enough in simple screens.  
   ![installer menu](/assets/pages/base-system/installer_menu.png)
2. Select a language  
   Choose "English - English". As described on the screen, this will be the default language of the system. In case of emergency (e.g. accessing from the very limited envrionments), English will be the safest choice.
3. Select your location  
   This will detemine the system timezone. This doesn't have to be where the actual server is located.  
   If your choice is not in the first list, you can find more options from "other" in the bottom.  
   ![select location](/assets/pages/base-system/select_location.png)
4. Configure locales  
   If the installer couldn't automatically guess the locale, it will ask you to choose one. For example, language: English and locale: Japan is not a normal combination and the system will show this screen.  
   "en_??.UTF-8" is recommended as a safe choice.
5. Configure the keyboard  
   Choose the keyboard layout you're using now for installation.

## Configure the network

If DHCP is on, the installer will autoconfigure the network. But the server should have static IP and manual configuration should be required.

1. IP address  
   It accepts both IPv4 and IPv6. If the server has both v4 and v6 address, set v4 here and add v6 after the installation.
2. Netmask  
   It should be something like `255.255.255.0` or `255.255.254.0`.
3. Gateway  
   Configure the IP provided from your server provider.
4. Name server addresses  
   If there are secondary DNS, list all of them separated by space, not comma.  
   `192.168.0.10 192.168.0.11`
5. Hostname  
   The first part of the server's FQDN. The "hostname" part of `hostname.example.com`.
6. Domain name  
   The "example.com" part of `hostname.example.com`.

## Set up users and passwords

Set root password and create a new user.

1. Root password  
   "root" is the administrator with all arivileges. This password must be kept secret and should not be used often, but never forget it.
2. Full name for the new user  
   This doesn't have to be your real name. Anything goes.
3. User name for your account  
   This will be the actual username of the account, which will be your primary mail address. (You can add aliases for email though.)
4. Choose a password for the new user  
   This password should be different from the root user. This will be required when you `sudo`.

## Partition disks

"Guided - user entire disk" should work fine for most cases.  
In my case, I used Manual configuration to consider the following factors.

- More swap to cover the lack of main memory  
  Due to small VPS, there are not enough memory for the peak usages. Larger swap file will be required.
- XFS for MongoDB  
  MongoDB requires XFS filesystem for the data storage. Instead of making a partition, you can use the loopback mount way to make an XFS area on a ext4 filesystem.

![partition](/assets/pages/base-system/partition.png)

## Configure the package manager

Choose the country nearest to the server location.

1. Scan extra installation media  
   Choose "No" to continue. Required packages should be downloaded from Debian mirrors.
2. Debian archive mirror country  
   You have to consider where the physical server is located because it should access the nearest debian mirrors.
3. Debian archive mirror  
   There will be the list of debian mirrors in the country you chose. Choose the one that is suitable to your network environment.  
   (In Japan, `ftp.jp.debian.org` is the CDN mirror that is recommended unless you're in the same network as one of the listed mirror.)
4. HTTP proxy  
   This will be required only if you're in the restricted network.

## Configuring popularity-contest

If you're ok, choose "Yes" to provide the package usage anonymized data to help Debian developers.

## Software selection

Choose nothing (unselect all). Even without "standard system utilities", it will still install "required" utilities.  
You can check what are in "standard system utilities" with this command.  
`tasksel --task-packages standard`

![software selection](/assets/pages/base-system/software_selection.png)

## Configuring grub-pc

The "server" should have only one OS, Debian. GRUB boot loader should be installed into the primary drive.

Unless you're doing something uncommon, you should have only one option (device) to install GRUB.

![GRUB install](/assets/pages/base-system/grub_install.png)

## Finish the installation

Install is complete. Continue to finish installation and remove the install media before rebooting.
