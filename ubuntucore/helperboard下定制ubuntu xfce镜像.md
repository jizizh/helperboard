## 定制ubuntu xfce镜像

### 环境：

- 芯片：全志a64　　
- 内核：Linux3.10内核
- 主机：ubuntu16.04
- 开发板：[helperboard a64](https://item.taobao.com/item.htm?spm=a230r.1.14.27.6f7076ffgIj8Ws&id=563738220031&ns=1&abbucket=3#detail)
- 公司：[百杰科技](https://www.szbaijie.com/)
- github：[Baijie Technology](https://github.com/jizizh/helperboard.git)

### 一、下载ubuntucore文件系统

**下载地址**：http://cdimage.ubuntu.com/ubuntu-base/releases/18.04/release/

ubuntu下使用wget来快捷下载

```shell
wget http://cdimage.ubuntu.com/ubuntu-base/releases/18.04/release/ubuntu-base-18.04.5-base-arm64.tar.gz
```

创建ubuntucore与rootfs文件夹，然后解压ubuntucore文件系统到rootfs中。然后删除ubuntucore文件系统。

```shell
mkdir -p ubuntucore/rootfs
cp ubuntu-base-18.04.5-base-arm64.tar.gz ubuntucore/rootfs
cd ubuntucore/rootfs
tar -xvf ubuntu-base-18.04.5-base-arm64.tar.gz
rm -rf ubuntu-base-18.04.5-base-arm64.tar.gz
```

### 二、qemu的下载与使用

#### 1、模拟工具

在PC上模拟运行根文件系统。需要安装一个工具：

```shell
sudo apt-get install qemu-user-static
```

然后输入命令：

```shell
cd ubuntucore  sudo cp /usr/bin/qemu-aarch64-static rootfs/usr/bin/
```

#### 2、创建ch-mount.sh脚本：

```shell
#!/bin/bash

function mnt() {
    echo "MOUNTING"
    sudo mount -t proc /proc ${2}proc
    sudo mount -t sysfs /sys ${2}sys
    sudo mount -o bind /dev ${2}dev
    sudo chroot ${2}
}
function umnt() {
    echo "UNMOUNTING"
    sudo umount ${2}proc
    sudo umount ${2}sys
    sudo umount ${2}dev
}

if [ "$1" == "-m" ] && [ -n "$2" ] ;
then
    mnt $1 $2
elif [ "$1" == "-u" ] && [ -n "$2" ];
then
    umnt $1 $2
else
    echo ""
    echo "Either 1'st, 2'nd or both parameters were missing"
    echo ""
    echo "1'st parameter can be one of these: -m(mount) OR -u(umount)"
    echo "2'nd parameter is the full path of rootfs directory(with trailing '/')"
    echo ""
    echo "For example: ch-mount -m /media/sdcard/"
    echo ""
    echo 1st parameter : ${1}
    echo 2nd parameter : ${2}
fi 
```

创建完成后给mount.sh可执行属性：

```shell
chmod +x ch-mount.sh
```

#### 3、运行与退出

运行：

```shell
./ch-mount.sh -m rootfs/
```

退出：

```shell
exit
./ch-mount.sh -u rootfs/
```

**注：如果退出虚拟根文件系统没有执行./mount.sh -u rootfs而重复执行执行./mount.sh -m rootfs。将会导致系统出故障，只能重新启动电脑，然后才能进入虚拟根文件系统。**

### 三、安装必要软件

#### 1、拷贝主机网络配置

拷贝主机网络配置到虚拟根文件系统（这样才能安装软件），命令如下：

```shell
cd ubuntucore/
sudo cp -b /etc/resolv.conf  rootfs/etc/resolv.conf
```

#### 2、安装软件

```shell
./ch-mount.sh -m rootfs/
apt update
apt install wget udev kmod iproute2 net-tools systemd vim
apt-get install language-pack-en-base sudo ssh ethtool
apt-get install wireless-tools ifupdown network-manager iputils-ping rsyslog
apt-get install bash-completion htop lrzsz --no-install-recommends
```

如果出现如下图错误

````shell
Couldn' tcreate temporary file /tmp/apt. conf .06WTRH for passing config to apt-key
````

重新改下 tmp 目录的权限： 

````
chmod 777 /tmp/
````

### 四、用户设置

给系统增加一个帐号：szbaijie

```shell
useradd -s '/bin/bash' -m -G adm,sudo szbaijie
```

修改 szbaijie 用户密码，回车后按提示输入两次密码：

```shell
passwd szbaijie
```

设置 root 权限的密码命令为 szbaijie：

```shell
passwd root
```

**注：在 root 用户下给 szbaijie 增加 sudo 用户权限。vi 进入/etc/sudoers/中在 root 一行下面加入内容：szbaijie ALL=(ALL:ALL) ALL**

### 五、解决问题

#### 1、 登录警告

当开发板 root 登陆操作系统界面时，会出现警告，修改如下：

```shell
vi /root/.profile
```

　然后将 **mesg n** 替换为 **tty - s && mesg n**

#### 2、ping失败的问题

```shell
vi /etc/group
```

　在文件最后加入：

```shell
inet:x:3003:root
net_raw:x:3004:root
```

#### 3、xshell问题

xshell 中 vim 进入后 xshell 只显示一半与乱码的问题

```shell
 vim /root/.bashrc
```

　在文件最后加入

```shell
export TERM=xterm
```

#### 4、删除自带设备

原有的设备文件不可写，所以编译打包会出错，打包不了，所以需要删除。如果 dev 下为空，不用删除了。如果不为空，则将黄色部分全部删除，命令如下：

```shell
cd ubuntucore/rootfs/dev/
rm -rf full null ptmx random tty urandom zero
```

#### 5、用户权限

退出虚拟根文件系统都要执行修改下用户权限，命令如下：

```shell
chown -R 用户名.用户名 rootfs
```

### 六、设置启动脚本与服务

首先，创建system文件夹。

```
mkdir rootfs/usr/lib/systemd/system/cd rootfs/usr/lib/systemd/system/
```

然后建立firstboot.sh、firstboot.service和rcs.service和rcS

#### 1、firstboot.sh

```shell
#!/bin/sh
mkfs.ext4 /dev/mmcblk0p1
chmod +s /usr/bin/sudo
chmod a+s /bin/su
chown -R szbaijie:szbaijie /home/szbaijie/
chown lightdm:lightdm -R /var/lib/lightdm
chown colord:colord -R /var/lib/colord/

rm -rf /etc/systemd/system/multi-user.target.wants/firstboot.service
exit 0
```

#### 2、firstboot.service

```shell
[Unit]
Description=fristboot-Service
Before=rcS.service
[Service]
Type=forking
ExecStart=/usr/lib/systemd/system/firstboot.sh start
[Install]
WantedBy=multi-user.target
```

#### 3、rcS

```shell
#!/bin/sh
mkdir /data
mount -t ext4 /dev/mmcblk0p1 /data
alias ll='ls -l'

insmod /lib/modules/`uname -r`/usbnet.ko
insmod /lib/modules/`uname -r`/asix.ko
insmod /lib/modules/`uname -r`/sunxi-keyboard.ko
insmod /lib/modules/`uname -r`/disp.ko
insmod /lib/modules/`uname -r`/8723bs.ko
insmod /lib/modules/`uname -r`/videobuf2-dma-contig.ko
insmod /lib/modules/`uname -r`/vfe_io.ko
insmod /lib/modules/`uname -r`/gc2145.ko
insmod /lib/modules/`uname -r`/gc0312.ko
insmod /lib/modules/`uname -r`/vfe_v4l2.ko
insmod /lib/modules/`uname -r`/gt9xxnew_ts.ko
dhclient eth0 &
```

#### 4、rcs.service

```shell
[Unit]

Description=rcS-Service
After=rcS.service
[Service]
Type=forking
ExecStart=/usr/lib/systemd/system/rcS start [Install] WantedBy=multi-user.target
```

#### 5、使能服务

进入虚拟根文件系统，执行以下命令。

````shell
systemctl enable firstboot.service
````

使能后会在/etc/systemd/system/multi-user.target.wants/ firstboot.service 指向我们建立的 firstboot.service（指向下图）。同理，**rcs.service**服务也是这样。

#### 6、设置目录权限

设置完后，退出虚拟根文件系统并设置目录拥有者（每次退出都要操作这个）。

````shell
./ch-mount.sh -u rootfs/
sudo chown -R jzzh.jzzh rootfs/
````

最后，将 rootfs 用 tar 命令压缩打包为保存好（接下来就是安装xfce桌面环境了）。

```shell
sudo tar -zcvf ubuntu_core_18.04.tar.gz rootfs/　
```

### 七、安装桌面环境xfce

#### 1、安装lightdm

进入虚拟根文件系统中，然后执行以下命令：

```shell
./ch-mount.sh -m rootfs/
apt-get install lightdm
```

如果出现以下错误：

````
Couldn' t create temporary file tmp/apt.conf.06WTRH for passing config to apt-key
````

执行以下命令可消除错误：

```shell
chmod 777 rootfs/tmp/
```

在安装 lightdm（桌面显示管理器）时，会出现时区、地区等选择，选择对应位置就行。

#### 2、安装xfce

安装完 lightdm 后，就是安装桌面环境xfce。

```shell
apt-get install --no-install-recommends xubuntu-desktop –y
```

#### 3、gnome-termial问题

设置 root 用户下打不开 gnome-terminal 的问题：

```shell
vi /etc/default/locale
```

在里面添加：LANG=zh_CN.UTF-8
然后执行下列命令，root 用户就能启动 gnome-terminal 终端了

```shell
locale-gen zh_CN.UTF-8
```

最后，退出虚拟根文件系统。

### 八、打包

具体的打包过程参考 HelperA64 参考手册的第八章，其他平台按照其他平台方式打包。

有时候我们打包会失败，查到原因如下

```shell
du: cannot read directory 'rootfs/var/lib/snapd/void' : Permission denied
```

然后给 void 权限就可以成功打包了。

```shell
chmod 777 rootfs/var/lib/snapd/void/
```

### 九、替换源

以 root 身份打开 sources.list，将 ：http://ports.ubuntu.com/ 全部替换为中科大的源：**http://mirrors.ustc.edu.cn/ubuntu-ports/。**

```shell
cd ubuntucore/
vi rootfs/etc/apt/sources.list
```

**vi 进入 sources.list 时，命令行模式下输入下面命令一键替换源：**

```shell
:%s/ports.ubuntu.com/mirrors.ustc.edu.cn/g
```

### 十、总结

在制作的过程中会遇到麻烦和挫折，多去查资料和操作就能成功，在此基础上可以搞出更多的功能。
