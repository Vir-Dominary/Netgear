# Netgear WNAP320接入点固件分析和漏洞利用

## 实验要求

选取参考链接（或自行搜索）页面中任意一个路由器、摄像头或者其它智能设备的固件漏洞，对目标固件进行逆向工程和漏洞分析，剖析漏洞机理，找到漏洞利用方法，编写漏洞利用代码，展示漏洞利用效果，简述漏洞防护方法。具体要求：
1、全程记录过程，形成文档，文件名中标明漏洞的CVE编号。
2、提交文件，压缩包中包含固件、源代码等。
3、可借鉴他人思路和方法，坚决杜绝全文拷贝。

## 实验准备
 + AttifyOS 3.0 虚拟环境
 + 所用固件型号：Netgear WNAP 320
 + CVE编号：2016-1555

注：推荐在kali环境下进行本次实验，Attify不带有python中的fat功能，需要自行安装。我选择attify是因为想试试徐老师推荐的虚拟机型号。但总体来说差别不大，只需要安装一个包即可

## 实验流程

### 1.安装并解压实验所需的固件

固件下载链接：[FirmWare Version 2.0.3](https://www.downloads.netgear.com/files/GDC/WNAP320/WNAP320%20Firmware%20Version%202.0.3.zip)

也可以使用如下命令进行下载
```bash
wget https://www.downloads.netgear.com/files/GDC/WNAP320/WNAP320%20Firmware%20Version%202.0.3.zip
```
![下载与解压](img%5C%E4%B8%8B%E8%BD%BD%E4%B8%8E%E8%A7%A3%E5%8E%8B.jpg)

### 2.按如下指令对固件进行解压和分析

```bash
unzip WNAP320\\ Firmware\\ Version\\ 2.0.3.zip
###按tab可以直接补全
tar -xvf WNA320_V2.0.3_firmware.tar
```
解压出的结果被其他命令覆盖了TvT

ls查看现有的文件和文件夹，其中的绿字部分即为存储了固件信息的文件
![解压结果](img%5C%E8%A7%A3%E5%8E%8B%E7%BB%93%E6%9E%9C.jpg)

使用binwalk进行查看，可以看到版本信息等
```bash
binwalk rootfs.squahfs
binwalk -e rootfs.squahfs
```
![information of files](img%5C%E6%96%87%E4%BB%B6%E4%BF%A1%E6%81%AF.jpg)

```
###进入rootfs.squahfs.extracted文件包
cd _rootfs.squahfs.extracted/
###查看文件夹里的内容
ls
###进入子文件夹
cd squashfs-root/
cd home
cd www
ls
```
可以看到login.php与boardDataWW.php两个关键文件，用sublime打开，查看源码
![文件](img%5C%E5%85%B3%E9%94%AE%E6%96%87%E4%BB%B6.jpg)
![源码信息](img%5C%E6%96%87%E4%BB%B6%E7%9A%84%E6%BA%90%E7%A0%81%E4%BF%A1%E6%81%AF.jpg)

在login.php里可以看到'usemane==admin','pasword==str[1]'等信息

在boardDataWW.php里可以看到'wr_mfg_data -m.$_REQUEST['macAddress']." -c ".$_REQUEST['reginfo'],$dummy,$res'。这是针对本次的溢出攻击的重要信息，结合前文的账号密码，通过栈溢出攻击获取password文件

### 3.对固件进行攻击，找到password文件

需要下载fat包
```bash
git clone https://github.com/attify/firmware-analysis-toolkit
cd firmware-analysis-toolkit
./setup.sh
###数据组模块
sudo apt-get install postgresql
sudo -u postgres createuser -P firmadyne，用`firmadyne`的密码
sudo -u postgres createdb -O firmadyne firmware
sudo -u postgres psql -d firmware < ./firmadyne/database/schema
echo "ALTER USER firmadyne PASSWORD 'firmadyne'" | sudo -u postgres psql
###设置fat.config，某些业务要有sudo权限
[DEFAULT]
sudo_password=attify123
firmadyne_path=/home/attify/firmadyne
```

将rootfs.squahfs文件复制到工作目录下，运行fat
![fat](img%5Cfat%E5%AF%B9%E6%96%87%E4%BB%B6%E8%BF%9B%E8%A1%8C%E8%A7%A3%E6%9E%90.jpg)

相当于模拟了固件的工作场景，network一看找到ip地址192.168.0.100，浏览器访问即可
![访问](img%5C%E8%AE%BF%E9%97%AE%E5%9B%BA%E4%BB%B6.jpg)

账号密码输入'admin，password'

这个登录界面显示的url是192.168.0.100/login.php，实际上将后面的login改成别的内容，可以实现对目标的访问。例如前文提到的boardDataWW.php
![boardDataWW](img%5CboardDataWW.php.jpg)
并不是所有文件都可以这样访问。例如passwd，这显然是无法直接查看的。
![passwd](img%5Cpasswd.html.jpg)

在记录数据的boardDataWW.php网页的mac地址栏里输入IP，可以查看这个IP地址对应的网络报文
![IP地址对应的网络报文](img%5C1111111%E5%AF%B9%E5%BA%94%E7%9A%84%E7%BD%91%E7%BB%9C%E4%BF%A1%E6%81%AF.jpg)

直接修改此处macaddress后的11111111，加上'(cp/etc/passwd/passwd.html'
![注入](img%5C%E6%B3%A8%E5%85%A5.jpg)
此时访问passwd，即可成功查看密码信息
![success](img%5C%E6%88%90%E5%8A%9F.jpg)

## 参考文献
[Wiki 96.mk](https://wiki.96.mk/)

[p1Kk's World!](https://p1kk.github.io/)

[阿里云社区](https://xz.aliyun.com/)

[fat安装与原理说明](https://blog.csdn.net/weixin_44932880/article/details/110845743)