By default, Debian 12 does not come with `sudo` installed.
For now we install it and configure the `admin` user to be part of the sudo group

1. Switch to the `root` user in a login shell by running
```bash
su -
```
1. Update software:
```bash
apt update
apt upgrade
```
1. Install sudo
```bash
apt install sudo
```
1. Add the `admin` user to the `sudo` group:
```bash
usermod -aGsudo admin
```
Note: this is why we had to run in a root login shell; otherwise the path would not include usermod.

Leave the root shell by running `exit`, then log out (run `exit` as your user).  
After you log back in you should be able to run commands with sudo.
