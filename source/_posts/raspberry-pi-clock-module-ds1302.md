---
title: '树莓派时钟模块 DS1302'
date: 2017-08-23 12:02:29
tags:
- 树莓派
- WiringPi
- DS1302
---

## 操作思路

1. 先将树莓派的系统时间通过 [NTP] [01] 网络对时，使系统时间正确。
2. 首次连接 [RTC] [02] 模块时，将系统时间写入 RTC 模块。
3. 树莓派断电后 RTC 模块使用自带的纽扣电池保持时间在走。
4. 树莓派启动时，通过脚本从 RTC 模块中读取出里面存储的时间，写回到树莓派系统中。

## 硬件连接

![DS1302 模块] [IMG01]

DS1302 模块一共有 5 个外部接口，接线方法如下：

| DS1302 模块 | 树莓派 GPIO |
| :-------: | :------: |
|    VCC    |   3.3V   |
|    GND    |  Ground  |
|    CLK    |   SCLK   |
|    DAT    |   SDA0   |
|    RST    |   CE0    |

> **注意：**
> _网上大部分教程只告诉你要按照上面的方法连接 RTC 模块和树莓派，却没有告诉你一个非常关键的事情：**在 RTC 模块的 VCC 和 DAT 这两个针脚之间还要接一个[上拉电阻] [03]（10K~30K）！**如果没有这个上拉电阻的话，你会发现用程序从 RTC 模块读出的时间非常不稳定，各种错乱。_

## 编写代码

安装 WiringPi 不表，[官网教程] [04]写的很详细。

### 一、修改示例源码：

```bash
cd ~/bin/
cp ~/src/WiringPi/examples/ds1302.c .
nano ds1302.c
```

1. 修改端口设置：

```cpp
// for Raspberry Pi Model B
ds1302setup   (0, 1, 2) ; => ds1302setup   (14, 8, 10) ;
// for Raspberry Pi Model B+
ds1302setup   (0, 1, 2) ; => ds1302setup   (14, 30, 10) ;
```

2. 修改时间设置函数（将获取 UTC 时间改为本地时间）：

```cpp
static int setDSclock(void)
{
  // struct tm t ;
  struct tm* t = NULL;
  time_t now ;
  int clock [8] ;

  printf ("Setting the clock in the DS1302 from Linux time... ") ;

  now = time (NULL) ;
  // gmtime_r (&now, &t) ;
  t = localtime(&now) ;

  // MUST change t.tm_xxx to t->tm_xxx
  clock [ 0] = dToBcd (t->tm_sec) ;             // seconds
  clock [ 1] = dToBcd (t->tm_min) ;             // mins
  clock [ 2] = dToBcd (t->tm_hour) ;            // hours
  clock [ 3] = dToBcd (t->tm_mday) ;            // date
  clock [ 4] = dToBcd (t->tm_mon + 1) ;         // months 0-11 --> 1-12
  clock [ 5] = dToBcd (t->tm_wday + 1) ;        // weekdays (sun 0)
  clock [ 6] = dToBcd (t->tm_year - 100) ;      // years
  clock [ 7] = 0 ;                              // W-Protect off
```

### 二、编译：

```bash
# for Raspberry Pi Model B
gcc ds1302.c -o ds1302-rpi-b -lwiringPi -lwiringPiDev
# for Raspberry Pi Model B+
gcc ds1302.c -o ds1302-rpi-b-plus -lwiringPi -lwiringPiDev
```

### 三、配置系统环境：

在使用 RTC 程序之前，首先要配置树莓派系统的时区，否则模块将无法正常使用。
编辑 `/etc/rc.conf` 文件，添加如下内容：

```bash
LOCALE="en_US.UTF-8"
DAEMON_LOCALE="no"
HARDWARECLOCK="localtime"
TIMEZONE="Asia/Shanghai"
```

然后执行 `ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime` 并网络同步一次系统时间（Raspbian 系统默认配置了自动同步服务，在此不详述）。

### 四、运行：

1. 测试模块：`$ ./ds1302-rpi-b -rtest`

  如果一切正常，程序会提示 `“DS1302 RAM TEST: OK”`。
  失败的话查看线连接的对不对，没有上拉电阻也会报错。

2. 同步网络时间：`ntpdate -d cn.pool.ntp.org`

3. 将系统时间写入 DS1302 模块：`$ ./ds1302-rpi-b -sdsc`

4. 读取 RTC 模块存储的时间，看是否正确：

  ```bash
  $ ./ds1302-rpi-b

  # 输出如下：

  0:   13:41:34  8/11/2015
  1:   13:41:34  8/11/2015
  2:   13:41:34  8/11/2015
  3:   13:41:35  8/11/2015
  ...
  ```

5. 开机自动同步时间：

  编辑 `/erc/rc.local` 文件，添加以下命令：

  ```bash
  # Setting the Linux Clock from the DS1302
  /home/pi/bin/ds1302-rpi-b slc
  ```

## 参考教程：

* [[原创] 为树莓派添加 DS1302 实时时钟（硬件时钟）/ Add a DS1302 RTC for RPi] [05]
* [（16）给树莓派B+ 安装一个实时时钟芯片DS1302] [06]
* [树莓派使用DS1302实现实时时钟功能] [07]
* [Raspberry Pi Real Time Clock Module DS1302] [08]


[01]: http://www.ntp.org/ntpfaq/NTP-s-def.htm "What is NTP?"
[02]: https://en.wikipedia.org/wiki/Real-time_clock "Real-time clock - Wikipedia"
[03]: https://www.zhihu.com/question/23167435 "能不能通俗的解释一下「上拉电阻/下拉电阻」的原理？ - 知乎"
[04]: http://wiringpi.com/download-and-install/ "Raspberry Pi | Wiring | Download & Install | Wiring Pi"
[05]: https://www.codelast.com/原创-为树莓派添加-ds1302-实时时钟（硬件时钟）/ "[原创] 为树莓派添加 DS1302 实时时钟（硬件时钟）/ Add a DS1302 RTC for RPi"
[06]: http://blog.csdn.net/hustsselbj/article/details/46050031 "（16）给树莓派B+ 安装一个实时时钟芯片DS1302"
[07]: http://blog.lxx1.com/1995 "树莓派使用DS1302实现实时时钟功能"
[08]: http://www.hobbytronics.co.uk/raspberry-pi-real-time-clock "Raspberry Pi Real Time Clock Module DS1302"

[IMG01]: https://raw.githubusercontent.com/codelast/raspberry-pi/master/real-time-clock/demo/ds1302_2.jpg "DS1302 模块"