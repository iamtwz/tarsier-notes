# 在 Ubuntu 21.10 上使用 QEMU 运行 openEuler RISC-V

## 0x00 主机环境

咱在一台 Windows 笔记本上使用 Hyper-V 虚拟机创建了一台 Ubuntu 21.10 的虚拟机，最新版 Ubuntu 对 Hyper-V 的增强会话支持有点问题，需要做一点点小改动，不然默认分辨率眼睛都要瞎了

微软官方有一个 [linux-vm-tools](https://github.com/microsoft/linux-vm-tools)，但是已经是 archived 状态，逼着你去用 WSL2，不过 WSL 有时候还是没有虚拟机灵活，所以咱还是选择了虚拟机方案。

```shell
wget https://raw.githubusercontent.com/Microsoft/linux-vm-tools/master/ubuntu/18.04/install.sh
chmod +x install.sh
sudo ./install.sh
```

尽管它是 18.04 时期的脚本，但是勉强能工作，不过最后会有 `xrdp.service` 的报错，所以修改一下 `/etc/xrdp/xrdp.ini` 的下列两行。

```ini
port=vsock://-1:3389
use_vsock=false
```

关机，然后去管理员 Powershell 运行：

``` powershell
Set-VM -VMName 'YourVMName' -EnhancedSessionTransportType HvSocket
```

虚拟机开机，增强会话工作正常。

## 0x01 QEMU RISC-V 环境配置

咱使用的是 Ubuntu 21.10：

```shell
➜  ~ apt search qemu
Sorting... Done
Full Text Search... Done
...
qemu/impish-updates,impish-security 1:6.0+dfsg-2expubuntu1.2 amd64
  fast processor emulator, dummy package
...

qemu-system-mips/impish-updates,impish-security 1:6.0+dfsg-2expubuntu1.2 amd64
  QEMU full system emulation binaries (mips)

qemu-system-misc/impish-updates,impish-security 1:6.0+dfsg-2expubuntu1.2 amd64
  QEMU full system emulation binaries (miscellaneous)
...
```

这里的 `qemu-system-misc` 其实已经包括了一堆常见指令集的模拟，就像它说的 `miscellaneous`，比如 `avr`，`riscv` 之类的都是在列的，更具体的介绍可以看看[这里](https://packages.debian.org/sid/qemu-system-misc)。

所以其实只要：

```shell
sudo apt install qemu qemu-system-misc
```

一行就能配完环境。

## 0x02 启动 openEuler 虚拟机

直接去 https://repo.openeuler.org/openEuler-preview/RISC-V/Image/ 下载最新版的 openEuler RISC-V 镜像，我们需要的是 `openEuler-preview.riscv64.qcow2` 和 `fw_payload_oe_docker.elf` 。

```shell
➜  openEuler qemu-system-riscv64 --version
QEMU emulator version 6.0.0 (Debian 1:6.0+dfsg-2expubuntu1.2)
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
```

qemu 没问题之后我们直接把虚拟机跑起来：

```shell
qemu-system-riscv64 \
  -nographic -machine virt \
  -smp 8 -m 2G \
  -kernel fw_payload_oe_docker.elf \
  -drive file=openEuler-preview.riscv64.qcow2,format=qcow2,id=hd0 \
  -object rng-random,filename=/dev/urandom,id=rng0 \
  -device virtio-rng-device,rng=rng0 \
  -device virtio-blk-device,drive=hd0 \
  -device virtio-net-device,netdev=usernet \
  -netdev user,id=usernet,hostfwd=tcp::2022-:22 \
  -append 'root=/dev/vda1 rw console=ttyS0 systemd.default_timeout_start_sec=600 selinux=0 highres=off mem=2048M earlycon' \
  -bios none
```

之后 ssh 过去就好

```shell
➜  ~ ssh root@localhost -p 2022

Last login: Tue Sep  3 14:07:05 2019 from 172.18.12.1
Welcome to 5.5.19

System information as of time: 	Tue Sep  3 09:29:08 UTC 2019

System load: 	1.18
Processes: 	119
Memory used: 	3.6%
Swap used: 	0.0%
Usage On: 	11%
Users online: 	1

[root@openEuler-RISCV-rare ~]# 
```

好耶！

## 0x03 openEuler 的一些小配置

登陆进去之后显然是可以发现系统时钟是不工作的，所以需要 NTP 同步一下时间，不然依赖包管理之类的都得挂掉。

修改 `/etc/systemd/timesyncd.conf` 中的 NTP [服务器](https://www.ntppool.org/hi/zone/cn)：

```shell
NTP=0.cn.pool.ntp.org 1.cn.pool.ntp.org 2.cn.pool.ntp.org 3.cn.pool.ntp.org
```

之后跑一下 `systemd-timesyncd` ：

```shell
[root@openEuler-RISCV-rare ~]# systemctl enable systemd-timesyncd
[root@openEuler-RISCV-rare ~]# systemctl start systemd-timesyncd
[root@openEuler-RISCV-rare ~]# timedatectl
               Local time: Sat 2022-03-05 17:26:20 UTC
           Universal time: Sat 2022-03-05 17:26:20 UTC
                 RTC time: n/a
                Time zone: n/a (UTC, +0000)
```

设置一下时区：

```shell
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

系统基本就可用了！

## 0x04 编译安装 neofetch

```shell
git clone https://github.com/dylanaraps/neofetch
cd neofetch
make install
```

编译安装一把梭，非常简单。

![](.\neofetch.png)

它工作了！

## 参考：

https://isrc.iscas.ac.cn/mirror/openeuler-sig-riscv/images/README.txt

https://gitee.com/openeuler/RISC-V/blob/master/doc/tutorials/vm-qemu-oErv.md

https://wiki.251.sh/openeuler_risc-v_qemu_install

