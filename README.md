# arch-in-fydeos
一种使用archlinux替换fydeOS自带Debian系统的方法

<!--more-->

# 前言
之前已经有人写过类似教程了。但是，随着chromeos的版本更新，过去的操作变得不再适用。目前，我们失去了一个很好的安装arch的方法。

fydeOS/ChromeOS官方提供的都是Debian，arch粉丝觉得这是某种垄断！

arch的社区安装确实是个人很喜欢的部分...从前debian系转arch系的时候觉得pacman很逆天，现在觉得ubuntu不人性...

不过说到底，初衷其实不过是在fydeOS上跑我个人认为的满血WPS...All in one真的是一个超级天才的想法


截止发稿日期，对于某些问题的解决可能算是全世界首发？也可能是我信息搜索能力有所欠缺。但这个领域也确实比较小众。

# 安装过程
## 0
我的fydeOS一休眠就会导致我打不开电脑，重启完就进不去lxc了

我当时挂着yay下载出去刷了个碗的功夫就寄了...甚至还连着电源

所以大家可以先确认自己的电脑不会自动休眠

在fydeOS上的操作还有开启linux，并且分配适当空间。

## 1.启动termina
按照我的理解，fydeOS/ChromeOS使用的是Crostini结构。其中，linux虚拟机、安卓与ChromeOS分属三个不同且独立的模块。对于linux，我们需要通过termina进行控制。而在crosh中打开termina就是我们需要克服的第一个难关。

关于Crostini结构的详细介绍，您可以参考[在FydeOS上安装ArchLinux][1]或[官方文档][2]获得更多信息

首先，我们可以按Ctrl+Alt+T打开crosh终端。但是，在过去的教程中，如果您运行了
```crosh
vmc start termina
```
那么您大概率会遇到vm_start问题，获得一个报错。我认为，这是ChromeOS的更新问题导致某些命令不再适用，尽管也并不能排除fydeOS官方做了什么改动。这也体现在我们无法按照archwiki上的操作直接通过vmc container添加容器。

针对前一问题，我发现可以通过运行
```crosh
vmc launch termina
```
启动termina

这时，您应该可以在终端中看到：

(termina) chronos@localhost ~ $

即termina启动成功

## 2.安装Arch容器
我们前面提到过，vmc container无法添加容器。这时，我们可以在termina中进行容器添加。

这里采用了12101111佬的方法，不过目前我们已经不用修改run_container.sh文件了，这位我们的安装工作省下了不小力气

**不过，请您注意，您需要将“你的用户名”替换为在fydeOS系统中使用的用户名，两者必须保持一致**
```termina
bash /usr/bin/run_container.sh --container_name arch --user 你的用户名 --lxd_image archlinux/current --lxd_remote https://mirrors.tuna.tsinghua.edu.cn/lxc-images/
```
**这里，我们推荐将这一命令重复执行两遍，以应对处理先后问题产生的bug**

第二次仍然会有几行报错，提示我们将用户加入wheel。这是正常现象，我们将在后面进行这一过程

## 3.进入arch的shell
```termina
lxc exec arch -- bash
```
这里的shell其实并不是用户身份登陆的，而是一个root管理
```shell
#设置密码.千万不要给root设置密码,否则ChromiumOS集成服务将无法运行，并且，这里的密码应与fydeOS保持一致
passwd 你的用户名
#把用户加入wheel组
usermod -aG wheel 你的用户名
```

如果添加密码时提示用户不存在，那么您需要重新执行
```termina
bash /usr/bin/run_container.sh --container_name arch --user 你的用户名 --lxd_image archlinux/current --lxd_remote https://mirrors.tuna.tsinghua.edu.cn/lxc-images/
```

## 4.基础设置与依赖安装
首先，因为众所周知的原因，在国内用arch需要设置国内源...
```source
# 清华大学
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
## 163
Server = http://mirrors.163.com/archlinux/$repo/os/$arch
## aliyun
Server = http://mirrors.aliyun.com/archlinux/$repo/os/$arch
```
起初我们只能用vi...个人觉得vi挺反人类的。
```shell
vi /etc/pacman.d/mirrorlist
```
复制上面的部分后，可以按i键选择插入，Crtl+Shift+V选择粘贴，以上部分应在官方源之前。

之后，您可以按esc键退出插入模式，再按:键输入wq保存退出

完成上面的设置后就可以pacman -Sy vim了（

之后，设置archlinuxcn源
```shell
vim /etc/pacman.conf
```
在最后面插入
```source
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```
安装一些依赖
```shell
pacman -Syu archlinuxcn-keyring base-devel git gtk3 openssh xdg-utils xkeyboard-config
```
因为我们期望能够在fydeOS的终端应用中启动arch。因此，我们需要开启无密码sudo
```shell
visudo
```
删除以下行前的注释，即删除#这一字符
```visudo
%wheel   ALL=(ALL:ALL) NOPASSWD: ALL
```
之后，我们需要退出到termina 
```shell
exit
```

## 5.登陆到容器
目前，我们应该已经退回到了termina中
```termina
lxc console arch
```
这时arch并不会提示输入用户名，但是需要直接输入...并且输入密码不可见

然后我们可以先验证一下网络
```shell
ip -4 a show dev eth0
```
输出不为空则证明网络正常。**否则**，请执行
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

“登录成功后安装aur上的cros-container-guest-tools-git。由于需要从chromium.googlesource.com下载文件，因此请自行解决网络问题。注意，Android或者Chromium OS里的代理设置不会应用到虚拟机。”

这里，我们可以使用手机开启代理，并通过v*n hostpot热点分享代理网络进行下载

btw，数字佬是用了git clone的方法下载...但既然都arch了，那不妨直接yay
```shell
pacman -Sy yay
yay -Sy cros-container-guest-tools-git
```
wayland包和xorg-xwayland也要装一下，否则没有gui
```shell
pacman -S wayland
pacman -S xorg-xwayland
```
安装中的提示全部默认即可

开启对应服务
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

## 6.替换默认容器
首先按下Ctrl+A，然后按下Q退出到termina。

为了更好的本地化运行，我们需要将arch改名为penguin
```termina
lxc stop --force arch
lxc stop --force penguin
lxc rename penguin debian
lxc rename arch penguin
lxc start penguin
```
然后，我们需要重新启动子系统进行更改
```termina
lxc console penguin
```
```shell
reboot
```
之后，再进入penguin
```termina
lxc console penguin
```
```shell
systemctl --failed
systemctl --user --failed
```
已检查是否系统服务均正常运行

也可以再检查一下网络
```shell
ip -4 a show dev eth0
```
如果返回为空，请执行
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

最后，如果一切正常，重启下fydeOS的系统再进终端就可以直接用arch了！

# 后记
Archwiki会推荐一上来先干掉自带的debian，不过这样的话就会无法通过lxc安装容器...

同时，他也提到lxc删除的容器并不彻底。而关于这个，我还没有什么太好的办法

有时会出现其他奇奇怪怪的问题...我的解决方法是remake子系统，关闭再重开那种...

有些操作可能也并非那么必要？比如用户名或者密码？不过我也没试过...懒得再折腾了

另外，之前的架构可能也并不绝对？或者说，至少在网络层面上，安卓v2rayng的代理可以作用于满血的google chrome...
这还挺反直觉的

最后，我不知道你会不会记得我最开始的初衷是在fydeOS上跑满血WPS，就是能用wps云的国内版本

但是，我在尝试将这个版本部署到fydeOS上时遇到了一些问题...在这个系统上遇到谁也没见过的问题再正常不过了，继续对自己的操作debug更是一种痛苦

不过，yay安装的方式相较sudo本质上或许更加温和，似乎可以规避wps的登录黑洞

因此，我真的安装了一个满血的arch，不是manjaro...但是，wps确实跑通了，赞美arch，赞美金山

除此之外，fydeOS的启动选择界面支持读取了我安装的所有系统，并且有很漂亮的图标，包括后面安装的arch...

这一点狠狠的赞美

### 但是：

我一开始只是想用一个简洁省电的系统来着...之前不知道在哪看到有人现身说法arch比win省电，装了“臃肿的”manjaro一个多小时电池就寄了，但win能跑一天，不知道arch会怎么样

所以fydeOS可能没用了？或许吧（

本质上我还是非常喜欢这个系统的来着...但是，当arch子系统的安装失去必要性（没法装wps满血），那么我或许会用回省心省事的debian？

不过这样一来主流linux发行版似乎都尝试过了...

目前在日常使用方面，个人感觉还是arch+gnome的观感更好

不过，或许用于服务器的系统不会这样？arch感觉更受发烧友的喜爱，但还是难以撼动老大哥们的江湖地位（


# 参考资料
[在FydeOS上安装ArchLinux][3]<br>
[Chrome OS devices/Crostini-wiki.archlinuxcn][4]<br>
[arch添加国内源以及社区源][5]<br>


  [1]: https://12101111.github.io/install-archlinux-on-fydeos/
  [2]: https://chromium.googlesource.com/chromiumos/docs/+/master/containers_and_vms.md#Glossary
  [3]: https://12101111.github.io/install-archlinux-on-fydeos/
  [4]: https://wiki.archlinuxcn.org/wiki/Chrome_OS_devices/Crostini
  [5]: https://www.jianshu.com/p/4444bb4f8452
