[TOC]
## CentOS 7 YUM 仓库源研究
## 一.背景说明

CentOS做为国内服务器生产环境使用的较多，但系统默认自带官方安装源，管理员在安装软件包时有可能下载慢，为了解决此类问题，现总结了两种解决办法，一是利用FastestMirror插件自动选择最快的镜像，另一种是手动更换国内镜像源。手动更换国内镜像源使用现已整合好的Base和EPEL的阿里源，清华源，科大源，下载Base和EPEL的源文件后，即可连接以上国内服务器下载安装软件包。本文的目的在于研究学习 YUM 仓库源的相关过程。

## 二.FastestMirror插件自动选择
FastestMirror插件自动选择是利用安装【yum-plugin-fastestmirror】插件后，插件根据仓库源文件里指定的URL，自动获取仓库镜像列表，并测试连接速度，分析下载最快的镜像仓库，最终在安装软件包时使用这个最快的镜像地址去下载软件包。
此插件的好处是可以实现自动化管理镜像仓库地址，但有时候选择的地址好慢，需要手动调整相关参数以选取更好的镜像仓库

### 启动容器
```shell
# 启动一个CentOS7容器，不污染外部环境
docker run -it centos:7 bash
```

### 安装Base源
如果默认官方源可用则不用安装
如果默认官方源修改了或删除了，想重新安装官方源，执行以下命令
```shell
rm -rf /etc/yum.repos.d/*
rpm -Uvh --force http://mirror.centos.org/centos-7/7/os/x86_64/Packages/centos-release-7-6.1810.2.el7.centos.x86_64.rpm
```

### 验证Base源
验证Base源的目的是看【 mirrorlist 】或【 baseurl 】配置是否正确，本方案的目的是利用FastestMirror插件根据 mirrorlist 自动下载镜像仓库地址，所以 mirrorlist 值一定要正确，并注释 baseurl 以便于 YUM 工具不使用 baseurl
```shell
[root@c311b111f4e9 base]# cat /etc/yum.repos.d/CentOS-Base.repo
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.

[base]
name=CentOS-$releasever - Base
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
#注意这里配置了mirrorlist，注释了baseurl。手动更换源时使用baseurl，注释mirrorlist
#启用mirrorlist使系统自动去查找相应的镜像列表并记录到文件中 - [/var/cache/yum/x86_64/7/base/mirrorlist.txt]
#并测试得出最服务器的连接速度，并记录到文件中 - [/var/cache/yum/x86_64/7/timedhosts.txt]
#在后继的软件包安装时，取最快的服务器地址进行连接下载

#released updates
[updates]
name=CentOS-$releasever - Updates
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#获取 mirrorlist 中变量值，便于用于curl ${mirrorlist}中的queryString中
[root@bac7ada9ecaf tools]# python -c 'import yum, pprint; yb = yum.YumBase(); pprint.pprint(yb.conf.yumvar, width=1)'
Loaded plugins: fastestmirror, ovl
{'arch': 'ia32e',
 'basearch': 'x86_64',
 'contentdir': 'centos',
 'infra': 'container',
 'releasever': '7',
 'uuid': '758bc17d-260a-48a7-91d0-9473454c5985'}

#根据上面的变量值，组装 mirrorlist url 并发起查询请求
#mirrorlist url来自base.repo中的mirrorlist配置项
#mirrorlist url中的查询变量值来自上面结果
[root@bac7ada9ecaf tools]# curl "http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=container"
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/
http://mirrors.neusoft.edu.cn/centos/7.6.1810/os/x86_64/
http://mirror.jdcloud.com/centos/7.6.1810/os/x86_64/
http://ftp.sjtu.edu.cn/centos/7.6.1810/os/x86_64/
http://ap.stykers.moe/centos/7.6.1810/os/x86_64/
http://mirrors.nwsuaf.edu.cn/centos/7.6.1810/os/x86_64/
http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/os/x86_64/
http://mirrors.cn99.com/centos/7.6.1810/os/x86_64/
http://mirrors.njupt.edu.cn/centos/7.6.1810/os/x86_64/
http://mirrors.163.com/centos/7.6.1810/os/x86_64/
# 从以上结果可以看出，mirrorlist 指向配置服务器上一文件的 URL，其内容为 baseurl 所要的值
```

### 安装EPEL源
此处安装的是EPEL官方源
```shell
rm -rf /etc/yum.repos.d/epel*
yum install epel-release
```

### 验证EPEL源
```shell
[root@60f59e160f4f yum.repos.d]# cat epel.repo
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
#baseurl=http://download.fedoraproject.org/pub/epel/7/$basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=$basearch
# 注意此处使用了 metalink，而不是 mirrorlist，更不是baseurl
# metalink 的作用见后继演示
failovermethod=priority
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

[epel-debuginfo]
name=Extra Packages for Enterprise Linux 7 - $basearch - Debug
#baseurl=http://download.fedoraproject.org/pub/epel/7/$basearch/debug
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-debug-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1

[epel-source]
name=Extra Packages for Enterprise Linux 7 - $basearch - Source
#baseurl=http://download.fedoraproject.org/pub/epel/7/SRPMS
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-source-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1

#测试epel.repo文件中metalink效果
[root@bac7ada9ecaf tools]# curl "https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=x86_64"
<?xml version="1.0" encoding="utf-8"?>
<metalink version="3.0" xmlns="http://www.metalinker.org/" type="dynamic" pubdate="Wed, 19 Jun 2019 02:15:05 GMT" generator="mirrormanager" xmlns:mm0="http://fedorahosted.org/mirrormanager">
  <files>
    <file name="repomd.xml">
      <mm0:timestamp>1560797592</mm0:timestamp>
      <size>5478</size>
      <verification>
        <hash type="md5">04c28b064e80175c259cadaa02a6d083</hash>
        <hash type="sha1">981c1da0e8b95c67b55f7e85c3cf67623620a350</hash>
        <hash type="sha256">7ad05ccec07c038ea07eaa839517b4d55d2f0d8a74d7e4d57b4c01011a4e9875</hash>
        <hash type="sha512">dcc1cd328e695931da1620f86675497c29917eec04e88eff730ab713d1f0a254c8d56b013e5fcbb99d4f67eaa28530d6dcac7b41cd3b3d72988aca044c196652</hash>
      </verification>
      <mm0:alternates>
        <mm0:alternate>
            <mm0:timestamp>1560644323</mm0:timestamp>
            <size>5476</size>
            <verification>
              <hash type="md5">7443fd012729e5552a088fdb6ba248ac</hash>
              <hash type="sha1">931911c3f0d6b4ed54eb852e6cd0063f281961c8</hash>
              <hash type="sha256">e7b6113a3e425d9468dea6602a3a8baeed39fc3e9ee5fca7e7e047d099160ed3</hash>
              <hash type="sha512">0fbffc1735f31274e8cfacece1ed36528a8f84b5805d0674014b696134c2048d3cb4880ed96b8b9adaaadd78d598505761c410f98b094c3c1fdfbd03c1b99f8a</hash>
            </verification>
        </mm0:alternate>
      </mm0:alternates>
      <resources maxconnections="1">
        <url protocol="http" type="http" location="CN" preference="100" >http://mirrors.tuna.tsinghua.edu.cn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="CN" preference="100" >rsync://mirrors.tuna.tsinghua.edu.cn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="CN" preference="100" >https://mirrors.tuna.tsinghua.edu.cn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="CN" preference="99" >rsync://mirrors.yun-idc.com/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="CN" preference="99" >http://mirrors.yun-idc.com/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="CN" preference="98" >http://mirrors.aliyun.com/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="CN" preference="97" >http://mirrors.njupt.edu.cn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="CN" preference="97" >https://mirrors.njupt.edu.cn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="CN" preference="96" >https://mirror.lzu.edu.cn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="CN" preference="96" >http://mirror.lzu.edu.cn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="JP" preference="95" >rsync://ftp.riken.jp/fedora/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="JP" preference="95" >http://ftp.riken.jp/Linux/fedora/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="JP" preference="94" >rsync://ftp.jaist.ac.jp/pub/Linux/Fedora/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="JP" preference="94" >http://ftp.jaist.ac.jp/pub/Linux/Fedora/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="TH" preference="93" >rsync://mirrors.nipa.cloud/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="TH" preference="93" >http://mirrors.nipa.cloud/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="IN" preference="92" >http://repos.del.extreme-ix.org/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="IN" preference="92" >rsync://repos.del.extreme-ix.org/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="PH" preference="91" >http://mirror.pregi.net/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="PH" preference="91" >https://mirror.pregi.net/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="JP" preference="90" >https://ftp.yz.yamagata-u.ac.jp/pub/linux/fedora-projects/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="JP" preference="90" >rsync://ftp.yz.yamagata-u.ac.jp/pub/linux/fedora-projects/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="JP" preference="90" >http://ftp.yz.yamagata-u.ac.jp/pub/linux/fedora-projects/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="BD" preference="89" >http://mirror.xeonbd.com/fedora-epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="TH" preference="88" >rsync://mirror1.ku.ac.th/fedora-epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="TH" preference="88" >http://mirror1.ku.ac.th/fedora/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="KZ" preference="87" >rsync://mirror.ps.kz/fedora-epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="KZ" preference="87" >http://mirror.ps.kz/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="KZ" preference="87" >https://mirror.ps.kz/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="MY" preference="86" >http://my.fedora.ipserverone.com/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="MY" preference="86" >rsync://my.fedora.ipserverone.com/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="ID" preference="85" >rsync://kartolo.sby.datautama.net.id/EPEL/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="ID" preference="85" >http://kartolo.sby.datautama.net.id/EPEL/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="SG" preference="84" >http://sg.fedora.ipserverone.com/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="VN" preference="83" >http://mirror.horizon.vn/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="TW" preference="82" >rsync://fedora.cs.nctu.edu.tw/fedora-epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="TW" preference="82" >http://fedora.cs.nctu.edu.tw/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="SG" preference="81" >http://download.nus.edu.sg/mirror/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="SG" preference="81" >rsync://download.nus.edu.sg/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="SG" preference="81" >https://download.nus.edu.sg/mirror/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="rsync" type="rsync" location="KZ" preference="80" >rsync://mirror.hoster.kz/fedora-epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="https" type="https" location="KZ" preference="80" >https://mirror.hoster.kz/fedora/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="KZ" preference="80" >http://mirror.hoster.kz/fedora/epel/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="IQ" preference="79" >http://epel.scopesky.iq/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="TW" preference="78" >http://mirror01.idc.hinet.net/EPEL/7/x86_64/repodata/repomd.xml</url>
        <url protocol="http" type="http" location="VN" preference="77" >http://mirror.ehost.vn/epel/7/x86_64/repodata/repomd.xml</url>
      </resources>
    </file>
  </files>
</metalink>
#从结果上来看，metalink类似于mirrorlist效果，但返回结果比 mirrorlist 更丰富，带更多控制参数
```

### 安装FastestMirror插件
FastestMirror插件用于自动选择速度最快的仓库镜像下载软件包
CentOS7 Docker镜像默认已安装此插件，如果没有请安装
关于此插件安装请参考-[https://wiki.centos.org/PackageManagement/Yum/FastestMirror]
```shell
yum install yum-plugin-fastestmirror
```

### 配置FastestMirror插件
关于此插件配置请参考-[https://wiki.centos.org/PackageManagement/Yum/FastestMirror]
```shell
# 打开FastestMirror插件的调试开关
sed -i 's/verbose=0/verbose=1/g' /etc/yum/pluginconf.d/fastestmirror.conf

# 查看并确认相关配置信息
cat /etc/yum/pluginconf.d/fastestmirror.conf
[main]
enabled=1  #启用此插件
verbose=1  #打开调试开关，输出镜像连接测试过程
always_print_best_host = true
socket_timeout=3
#  Relative paths are relative to the cachedir (and so works for users as well
# as root).
hostfilepath=timedhosts.txt #镜像测试速度记录文件
maxhostfileage=10
maxthreads=15
#exclude=.gov, facebook #不包括哪些网址，因考考虑到类gov镜像下载慢
#include_only=.nl,.de,.uk,.ie
```

### 打开YUM缓存
打开YUM缓存后，在使用YUM命令时会把相关的软件包存放在以下路径，目的是为了后继查看软件包究竟从哪下载，还可用于验证 yum clean all 时是否清理这些软件包 
```shell
# 【/var/cache/yum/$basearch/$releasever/base/packages】
# 如：/var/cache/yum/x86_64/7/base/packages/logrotate-3.8.6-17.el7.x86_64.rpm
# 如：/var/cache/yum/x86_64/7/epel/packages/redis-3.2.12-2.el7.x86_64.rpm
sed -i 's/keepcache=0/keepcache=1/g' /etc/yum.conf
```

### 安装软件包
为了测试前面镜像仓库源配置是否正确，能否正确安装软件，此例安装 mysql
安装 mysql 时系统要下载安装Base和EPEL两个仓库中的软件包和依赖，正好测试两个仓库源的配置是否正确
```shell
yum install mysql
```

### 查看镜像仓库列表
【 mirrorlist.txt 】文件里存放的是 FastestMirror 插件根据 .repo 文件中 mirrorlist 参数指定的URL下载元数据解析并得到可用的镜像仓库URL，为后继的速度测试和选出最快的镜像下载做准备
```shell
[root@60f59e160f4f yum.repos.d]# cat /var/cache/yum/x86_64/7/base/mirrorlist.txt
http://mirrors.neusoft.edu.cn/centos/7.6.1810/os/x86_64/
http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/os/x86_64/
http://mirror.lzu.edu.cn/centos/7.6.1810/os/x86_64/
http://ap.stykers.moe/centos/7.6.1810/os/x86_64/
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/
http://ftp.sjtu.edu.cn/centos/7.6.1810/os/x86_64/
http://centos.ustc.edu.cn/centos/7.6.1810/os/x86_64/
http://mirror.jdcloud.com/centos/7.6.1810/os/x86_64/
http://mirrors.njupt.edu.cn/centos/7.6.1810/os/x86_64/
http://mirrors.163.com/centos/7.6.1810/os/x86_64/
```
此处【 mirrorlist.txt 】中的内容应该对应【curl "http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=container"】和【curl "https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=x86_64"】的返回值之和

### 查看镜像仓库连接速度
【 timedhosts.txt 】是 FastestMirror 插件根据 【mirrorlist.txt】里的URL挨个测试连接速度，再在此文件中记录每个URL的连接速度值，以便于安装软件包时挑选最快的镜像仓库做参考
```shell
[root@60f59e160f4f yum.repos.d]# cat /var/cache/yum/x86_64/7/timedhosts.txt
ap.stykers.moe 0.0704910755157
mirror.bit.edu.cn 0.0865077972412
mirrors.njupt.edu.cn 0.114202976227
centos.ustc.edu.cn 0.0678670406342
mirrors.tuna.tsinghua.edu.cn 0.108617067337
mirrors.huaweicloud.com 0.0630021095276
ftp.sjtu.edu.cn 0.119927883148
mirrors.cqu.edu.cn 0.101330041885
mirror.lzu.edu.cn 0.140116930008
mirrors.neusoft.edu.cn 0.115367889404
mirror.jdcloud.com 0.0815250873566
mirrors.163.com 0.0608839988708
mirrors.aliyun.com 0.0417621135712
```

### 验证
查看已安装的软件包真实的下载地址，对比【镜像仓库连接速度表】，试问 YUM 是挑选最快的镜像仓库地址去下载的吗？
```shell
#查找所有 YUM 安装过的软件包的下载地址，便于清楚的了解到一个软件包究竟从哪个URL下载，便于问题排查，此种方法仅限于CentOS7及以下版本
[root@60f59e160f4f yum.repos.d]# find / -name origin_url | xargs -I{} sh -c "cat {}; echo ''"
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/groff-base-1.22.2-8.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/extras/x86_64/Packages/epel-release-7-11.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-podlators-2.5.1-3.el7.noarch.rpm
http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/os/x86_64/Packages/perl-Time-Local-1.2300-2.el7.noarch.rpm
http://mirrors.163.com/centos/7.6.1810/os/x86_64/Packages/perl-Socket-2.010-4.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Carp-1.26-244.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Filter-1.49-3.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Getopt-Long-2.40-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/updates/x86_64/Packages/perl-5.16.3-294.el7_6.x86_64.rpm
http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/os/x86_64/Packages/perl-Scalar-List-Utils-1.27-248.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-File-Temp-0.23.01-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Encode-2.51-7.el7.x86_64.rpm
http://mirrors.163.com/centos/7.6.1810/os/x86_64/Packages/perl-constant-1.27-2.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Time-HiRes-1.9725-3.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Exporter-5.68-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-PathTools-3.40-5.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-HTTP-Tiny-0.033-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-threads-shared-1.43-6.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-parent-0.225-244.el7.noarch.rpm
http://mirrors.163.com/centos/7.6.1810/os/x86_64/Packages/perl-Text-ParseWords-3.29-4.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-threads-1.87-4.el7.x86_64.rpm
http://mirrors.163.com/centos/7.6.1810/updates/x86_64/Packages/perl-Pod-Escapes-1.04-294.el7_6.noarch.rpm
http://mirrors.163.com/centos/7.6.1810/os/x86_64/Packages/perl-Pod-Usage-1.63-3.el7.noarch.rpm
http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/updates/x86_64/Packages/perl-macros-5.16.3-294.el7_6.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Storable-2.45-3.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Pod-Perldoc-3.20-4.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-Pod-Simple-3.28-4.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7.6.1810/os/x86_64/Packages/perl-File-Path-2.09-2.el7.noarch.rpm
http://mirrors.163.com/centos/7.6.1810/updates/x86_64/Packages/perl-libs-5.16.3-294.el7_6.x86_64.rpm
http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/os/x86_64/Packages/mariadb-libs-5.5.60-1.el7_5.x86_64.rpm
http://mirrors.163.com/centos/7.6.1810/os/x86_64/Packages/mariadb-5.5.60-1.el7_5.x86_64.rpm
```
从以上结果可以看出
- 各镜像下载次数对比 aliyun：20/31，163：7/31，tsinghua：4/31
- 此例多处使用阿里云镜像仓库下载【 mirrors.aliyun.com 】，对比镜像仓库连接速度表 【 timedhosts.txt 】得知，【 mirrors.aliyun.com 0.0417621135712 】是最快的选择

## 三.手动配置镜像仓库地址
手动配置是指在仓库源文件中指定固定的仓库路径，YUM安装工具根据源文件中的路径直接去下载软件包，如果baseurl中配置有多个镜像地址，则会挑选最快的镜像仓库下载。
此种方法不同于FastestMirror插件（镜像仓库地址由mirrorlist指定的服务器统一维护），镜像仓库地址需要用户自行维护。
### 启动容器
```shell
# 启动一个CentOS7容器，不污染外部环境
docker run -it centos:7 bash
```

### 安装Base源
安装Base源为从网上下载相应的仓库源文件，或自行编辑现有的仓库源文件。此处给出的实例是基于本人修改合并了阿里源，清华源，科大源的统一中国源
```shell
#如果有官方源或是其它Base源，请先删除，再安装，不能同一种源有多个仓库源文件并且仓库名称还相同
rm -rf /etc/yum.repos.d/xxx*
curl https://raw.githubusercontent.com/leo18945/yum.repos.d/master/centos7-base-all.repo -o /etc/yum.repos.d/centos7-base-all.repo
```

### 验证Base源
```shell
[root@60f59e160f4f yum.repos.d]# cat /etc/yum.repos.d/centos7-base-all.repo
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.

[base]
name=CentOS-$releasever - Base - mirrors.aliyun.com
failovermethod=priority
# 注意，此处启用了baseurl，即镜像仓库地址列表，而不是mirrorlist
baseurl=http://mirrors.aliyun.com/centos/$releasever/os/$basearch/
            https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch/
            https://mirrors.ustc.edu.cn/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/updates/$basearch/
            https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/updates/$basearch/
            https://mirrors.ustc.edu.cn/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/extras/$basearch/
            https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/extras/$basearch/
            https://mirrors.ustc.edu.cn/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/centosplus/$basearch/
            https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/centosplus/$basearch/
            https://mirrors.ustc.edu.cn/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7

#contrib - packages by Centos Users
[contrib]
name=CentOS-$releasever - Contrib - mirrors.aliyun.com
failovermethod=priority
baseurl=http://mirrors.aliyun.com/centos/$releasever/contrib/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://mirrors.aliyun.com/centos/RPM-GPG-KEY-CentOS-7
[root@60f59e160f4f yum.repos.d] #
```

### 安装EPEL源
安装EPEL源为从网上下载相应的仓库源文件，或自行编辑现有的仓库源文件。此处给出的实例是基于本人修改合并了阿里源，清华源，科大源的统一中国源
```shell
#如果有官方源或是其它EPEL源，请先删除，再安装，不能同一种源有多个仓库源文件并且仓库名称还相同
rm -rf /etc/yum.repos.d/xxx*
curl https://raw.githubusercontent.com/leo18945/yum.repos.d/master/centos7-epel-all.repo -o /etc/yum.repos.d/centos7-epel-all.repo
```

### 验证EPEL源
```shell
[root@60f59e160f4f yum.repos.d]# cat /etc/yum.repos.d/centos7-epel-all.repo
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
# 注意，此处启用了baseurl，即镜像仓库地址列表，而不是metalink
baseurl=http://mirrors.aliyun.com/epel/7/$basearch
            https://mirrors.tuna.tsinghua.edu.cn/epel/7/$basearch
            https://mirrors.ustc.edu.cn/epel/7/$basearch
failovermethod=priority
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

[epel-debuginfo]
name=Extra Packages for Enterprise Linux 7 - $basearch - Debug
baseurl=http://mirrors.aliyun.com/epel/7/$basearch/debug
            https://mirrors.tuna.tsinghua.edu.cn/epel/7/$basearch/debug
            https://mirrors.ustc.edu.cn/epel/7/$basearch/debug
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=0

[epel-source]
name=Extra Packages for Enterprise Linux 7 - $basearch - Source
baseurl=http://mirrors.aliyun.com/epel/7/SRPMS
            https://mirrors.tuna.tsinghua.edu.cn/epel/7/SRPMS
            https://mirrors.ustc.edu.cn/epel/7/SRPMS
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=0
[root@60f59e160f4f yum.repos.d]#
```

### 打开YUM缓存
打开YUM缓存后，在使用YUM命令时会把相关的软件包存放在以下路径，目的是为了后继查看软件包究竟从哪下载，还可用于验证 yum clean all 时是否清理这些软件包 
```shell
# 【/var/cache/yum/$basearch/$releasever/base/packages】
# 如：/var/cache/yum/x86_64/7/base/packages/logrotate-3.8.6-17.el7.x86_64.rpm
# 如：/var/cache/yum/x86_64/7/epel/packages/redis-3.2.12-2.el7.x86_64.rpm
sed -i 's/keepcache=0/keepcache=1/g' /etc/yum.conf
```

### 安装软件包
为了测试前面镜像仓库源配置是否正确，能否正确安装软件，此例安装 mysql
安装 mysql 时系统要下载安装Base和EPEL两个仓库中的软件包和依赖，正好测试两个仓库源的配置是否正确
```shell
yum install mysql
```

### 查看镜像仓库列表
【 mirrorlist.txt 】文件里存放的是 FastestMirror 插件根据 .repo 文件中 mirrorlist 参数指定的URL下载元数据解析并得到可用的镜像仓库URL，为后继的速度测试和选出最快的镜像下载做准备
```shell
[root@bac7ada9ecaf yum.repos.d]# cat /var/cache/yum/x86_64/7/base/mirrorlist.txt
cat: /var/cache/yum/x86_64/7/base/mirrorlist.txt: No such file or directory
```
由于仓库源文件【.repo】中没有配置mirrorlist，所以镜你仓库列表文件不存在，即没有使用这一功能

### 查看镜像仓库连接速度
【 timedhosts.txt 】是 FastestMirror 插件根据 【.repo】里的 baseurl 挨个测试连接速度，再在此文件中记录每个URL的连接速度值，以便于安装软件包时挑选最快的镜像仓库做参考
```shell
[root@bac7ada9ecaf 7]# cat /var/cache/yum/x86_64/7/timedhosts.txt
mirrors.tuna.tsinghua.edu.cn 0.0670750141144
mirrors.aliyun.com 0.0400879383087
mirrors.ustc.edu.cn 0.0682208538055
```

### 验证
查看已安装的软件包真实的下载地址，对比【镜像仓库连接速度表】，试问 YUM 是挑选最快的镜像仓库地址去下载的吗？
```shell
#查找所有 YUM 安装过的软件包的下载地址，便于清楚的了解到一个软件包究竟从哪个URL下载，便于问题排查，此种方法仅限于CentOS7及以下版本
[root@bac7ada9ecaf 7]# find / -name origin_url | xargs -I{} sh -c "cat {}; echo ''"
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/groff-base-1.22.2-8.el7.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-podlators-2.5.1-3.el7.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Time-Local-1.2300-2.el7.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Socket-2.010-4.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-Carp-1.26-244.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-Filter-1.49-3.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-Getopt-Long-2.40-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7/updates/x86_64/Packages/perl-5.16.3-294.el7_6.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Scalar-List-Utils-1.27-248.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-File-Temp-0.23.01-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-Encode-2.51-7.el7.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-constant-1.27-2.el7.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Time-HiRes-1.9725-3.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-Exporter-5.68-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-PathTools-3.40-5.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-HTTP-Tiny-0.033-3.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-threads-shared-1.43-6.el7.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-parent-0.225-244.el7.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Text-ParseWords-3.29-4.el7.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-threads-1.87-4.el7.x86_64.rpm
https://mirrors.ustc.edu.cn/centos/7/updates/x86_64/Packages/perl-Pod-Escapes-1.04-294.el7_6.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Pod-Usage-1.63-3.el7.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/updates/x86_64/Packages/perl-macros-5.16.3-294.el7_6.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Storable-2.45-3.el7.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Pod-Perldoc-3.20-4.el7.noarch.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/perl-Pod-Simple-3.28-4.el7.noarch.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/perl-File-Path-2.09-2.el7.noarch.rpm
https://mirrors.ustc.edu.cn/centos/7/updates/x86_64/Packages/perl-libs-5.16.3-294.el7_6.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/centos/7/os/x86_64/Packages/mariadb-libs-5.5.60-1.el7_5.x86_64.rpm
https://mirrors.ustc.edu.cn/centos/7/os/x86_64/Packages/mariadb-5.5.60-1.el7_5.x86_64.rpm
```
- 从以上结果可以看出，下载所使用的 url 都是 baseurl 中指定的并没有超出指定范围，也没有自行去其它地方查询获取更多的镜像仓库列表。
- 且多个 url 交替使用，并不存在更偏向于使用哪个最快的，因为指定的是 baseurl，所以系统认为 baseurl 中的服务器都是平等一样的，如果有多个交替使用下载就行啦。