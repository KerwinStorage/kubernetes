😬前言：我们在VMware创建新的虚拟机时会发现没有全屏的现象这时我们就需要安装VMware tools来解决这个问题

## 1. 文件位置

  在VMware安装的本地文件夹下面有一个`Linux`的镜像文件需要挂载到虚拟机上面

![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220618152240002-1210215265.png)

## 2. 挂载到虚拟机上

![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220618152359694-285639354.png)

![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220618152828011-449644368.png)

## 3. copy到虚拟机桌面

\1. 将压缩包copy到Desktop

![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220618152549376-858468235.png)

\2. 解压并执行命令

```bash
sudo su
cd Desktop
cd vmware-tools-distrib
./vmware-install.pl
```

第一次询问就选`yes`然后后面一直回车，直到出现下面这个画面就可以全屏了。💯💯💯

![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220618153700345-116056648.png)

如果没有更新

```bash
sudo apt-get install open-vm-tools-desktop
# 重启
```

 