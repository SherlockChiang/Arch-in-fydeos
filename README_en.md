# arch-in-fydeos
A method to replace the pre-installed Debian system in FydeOS with Arch Linux

<!--more-->

**This article is translated by chatGPT**
# Preface
Similar tutorials have been written before. However, with the updates of ChromeOS versions, the previous methods are no longer applicable. Currently, we have lost a good way to install Arch Linux.

FydeOS/ChromeOS officially provides Debian, and Arch Linux enthusiasts see this as a kind of monopoly!

The Arch Linux community installation is indeed a part that many people like. When switching from the Debian-based system to the Arch-based system, I found Pacman to be very powerful, and now I find Ubuntu less user-friendly.

But in the end, the original intention was simply to run what I consider a full-featured WPS on FydeOS... the "All in one" idea is really brilliant.

As of the date of publication, the solutions to certain issues discussed here might be the first in the world, or it could be that my information searching abilities are lacking. Nevertheless, this area is indeed relatively niche.

# Installation Process
## 0
My FydeOS goes to sleep and I can't wake it up; after a restart, I can't enter lxc.

When I was downloading with yay, I went to wash some dishes, and it was already sent... I even left it plugged in.

So, you should first make sure that your computer won't go to sleep automatically.

On FydeOS, you also need to enable Linux and allocate appropriate space.

## 1. Launching Termina
As far as I understand, FydeOS/ChromeOS uses the Crostini structure. In this structure, Linux, Android, and ChromeOS are three different and independent modules. For Linux, we need to control it through Termina. However, opening Termina in Crosh is the first challenge we need to overcome.

For a detailed introduction to the Crostini structure, you can refer to [Install Arch Linux on FydeOS][1] or [the official documentation][2] for more information.

First, we can open the Crosh terminal by pressing Ctrl+Alt+T. However, in previous tutorials, if you run:

```crosh
vmc start termina
```
you will most likely encounter a vm_start problem and get an error message. I believe this is due to ChromeOS updates, which made some commands no longer applicable. Although I cannot rule out the possibility that FydeOS has made some modifications as well. This is also reflected in the fact that we cannot directly add containers through vmc container following the operations on the ArchWiki.

To address the previous issue, I found that you can start Termina by running:

```crosh
vmc launch termina
```
Now, you should see in the terminal:

(termina) chronos@localhost ~ $

indicating that Termina has started successfully.

## 2. Installing the Arch Container
As mentioned earlier, vmc container cannot be used to add containers. Therefore, we will add the container within Termina.

Here, we use the method from user "12101111," but we no longer need to modify the run_container.sh file. This saves us a lot of effort during the installation process.

However, please note that you need to replace "your_username" with the username you use in the FydeOS system; they must match.

```termina
bash /usr/bin/run_container.sh --container_name arch --user your_username --lxd_image archlinux/current --lxd_remote https://mirrors.tuna.tsinghua.edu.cn/lxc-images/
```
We recommend running this command twice to handle any potential bugs related to order processing.

On the second run, you will still get a few lines of error messages, prompting you to add the user to the wheel group. This is normal, and we will address this in the next step.

## 3. Accessing the Arch Shell
```termina
lxc exec arch -- bash
```
The shell here is not logged in as the user but as a root administrator.

```shell
# Set a password. Do not set a password for root, otherwise the integrated ChromiumOS services will not work. Also, the password should be the same as the one used in FydeOS.
passwd your_username
# Add the user to the wheel group
usermod -aG wheel your_username
```
If you receive a message saying the user doesn't exist when adding the password, then you need to execute the command again:

```termina
bash /usr/bin/run_container.sh --container_name arch --user your_username --lxd_image archlinux/current --lxd_remote https://mirrors.tuna.tsinghua.edu.cn/lxc-images/
```
## 4. Basic Setup and Dependency Installation
```shell
pacman -Syu vim base-devel git gtk3 openssh xdg-utils xkeyboard-config
```
Because we expect to run Arch in FydeOS's terminal application, we need to enable passwordless sudo

```shell
visudo
```
Remove the comment symbol (#) in front of the following line:

```visudo
%wheel   ALL=(ALL:ALL) NOPASSWD: ALL
```
After that, we need to exit to Termina

```shell
exit
```
## 5. Logging into the Container
At this point, we should be back in Termina.

```termina
lxc console arch
```
In Arch, it won't prompt for the username but requires you to type it directly... and the password input won't be visible.

Next, let's verify the network first

```shell
ip -4 a show dev eth0
```
If it returns something, it means the network is functioning properly. Otherwise, please execute the following commands:

```shell
sudo dhcpcd eth0
sudo pacman -S dhclient
sudo systemctl disable systemd-networkd
sudo systemctl disable systemd-resolved
sudo unlink /etc/resolv.conf
sudo touch /etc/resolv.conf
sudo systemctl enable dhclient@eth0
sudo systemctl start dhclient@eth0
```
"After logging in successfully, install cros-container-guest-tools-git from AUR. Since it requires downloading files from chromium.googlesource.com, please resolve any network issues on your own. Note that the proxy settings in Android or Chromium OS won't apply to the virtual machine."

Here, we can use a phone to enable a proxy and share the proxy network with V*n Hotspot for downloading.

By the way, the user before used git clone to download... but since we're using Arch, why not use yay directly?

```shell
pacman -Sy yay
yay -Sy cros-container-guest-tools-git
```
We also need to install the wayland package and xorg-xwayland, otherwise there will be no GUI.

```shell
pacman -S wayland
pacman -S xorg-xwayland
```
All the prompts during installation can be accepted as default.

Next, enable the corresponding services

```shell
sudo systemctl enable cros-sftp
systemctl --user enable sommelier@0.service
systemctl --user enable sommelier@1.service
systemctl --user enable sommelier-x@0.service
systemctl --user enable sommelier-x@1.service
systemctl --user enable cros-garcon.service
systemctl --user start sommelier@0
systemctl --user start sommelier-x@0
systemctl --user start sommelier@1
systemctl --user start sommelier-x@1
```
## 6. Replacing the Default Container
First, press Ctrl+A, then press Q to exit to Termina.

For better localization, we need to rename arch to penguin.

```termina
lxc stop --force arch
lxc stop --force penguin
lxc rename penguin debian
lxc rename arch penguin
lxc start penguin
```
Then, we need to reboot the subsystem to apply the changes.

```termina
lxc console penguin
```
```shell
reboot
```
After that, log in to penguin again.

```termina
lxc console penguin
```
```shell
systemctl --failed
systemctl --user --failed
```
Check if all the system services are running normally.

You can also check the network again.

```shell
ip -4 a show dev eth0
If it returns nothing, please execute the following commands:
```

```shell
sudo dhcpcd eth0
sudo pacman -S dhclient
sudo systemctl disable systemd-networkd
sudo systemctl disable systemd-resolved
sudo unlink /etc/resolv.conf
sudo touch /etc/resolv.conf
sudo systemctl enable dhclient@eth0
sudo systemctl start dhclient@eth0
```
Finally, if everything is fine, reboot your FydeOS system and enter the terminal again. Now, you can directly use Arch!

# Afterword
Archwiki recommends removing the pre-installed Debian first, but this will prevent us from installing containers via LXC...

Also, it's mentioned that deleting LXC containers is not complete. As for this, I don't have a better solution yet.

Sometimes, you may encounter other strange issues... My solution is to "remake" the subsystem, close it, and then restart it...

Some operations may not be necessary? Such as the username and password? But I haven't tried it... too lazy to tinker anymore.

Moreover, the previous architecture may not be absolute? Or at least at the network level, the proxy set up in Android or VPN can work with a full-featured Google Chrome... which is quite counterintuitive.

Currently, in terms of daily use, I feel that Arch with GNOME looks better.

# References
[Install Arch Linux on FydeOS][3]<br>
[Chrome OS devices/Crostini-wiki.archlinuxcn][4]<br>
[Add Chinese mirrors and community mirrors for Arch][5]<br>

  [1]: https://12101111.github.io/install-archlinux-on-fydeos/
  [2]: https://chromium.googlesource.com/chromiumos/docs/+/master/containers_and_vms.md#Glossary
  [3]: https://12101111.github.io/install-archlinux-on-fydeos/
  [4]: https://wiki.archlinuxcn.org/wiki/Chrome_OS_devices/Crostini
  [5]: https://www.jianshu.com/p/4444bb4f8452
