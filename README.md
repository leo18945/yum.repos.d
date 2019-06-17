# CentOS 7更换国内源

# 使用方法

```shell
# 启动一个CentOS7容器，换源动作在容器中进行，不污染外部环境
docker run -it centos:7 bash

# 打开YUM缓存，打开后会把相关的软件包存放在类似以下包下 
# 【  /var/cache/yum/$basearch/$releasever/base/packages】
# 如：/var/cache/yum/x86_64/7/base/packages/logrotate-3.8.6-17.el7.x86_64.rpm
# 如：/var/cache/yum/x86_64/7/epel/packages/redis-3.2.12-2.el7.x86_64.rpm
sed -i 's/keepcache=0/keepcache=1/g' /etc/yum.conf

# 进到YUM源目录
cd /etc/yum.repos.d/

# 删除原有的官方源
rm -rf *

# 获取Base源，此文件中合并了阿里源，清华源，科大源
curl https://raw.githubusercontent.com/leo18945/yum.repos.d/master/centos7-base-all.repo -o centos7-base-all.repo

# 查看并验证所取得到文件内容是否正确
cat centos7-base-all.repo

# 获取EPEL源，此文件中合并了阿里源，清华源，科大源
curl https://raw.githubusercontent.com/leo18945/yum.repos.d/master/centos7-epel-all.repo -o centos7-epel-all.repo

# 查看并验证所取得到文件内容是否正确
cat centos7-epel-all.repo

# 安装redis，会用到Base和EPEL源，验证前面的两个源配置是否正确
yum install redis

# 查看各个源的速度快慢
cat /var/cache/yum/x86_64/7/timedhosts.txt

# 查找所有YUM安装过的软件包的原始下载地址，便于清楚的了解到一个软件包究竟从中个URL中下载，便于问题排查，此种方法仅限于CentOS7及以下版本
find / -name origin_url
/var/lib/yum/yumdb/r/523ab344de37420aa091a581e652edb442c68e14-redis-3.2.12-2.el7-x86_64/origin_url

# 查看某个软件包的具体下载地址
find / -name origin_url | xargs cat
http://mirrors.aliyun.com/epel/7/x86_64/Packages/j/jemalloc-3.6.0-1.el7.x86_64.rpm
https://mirrors.tuna.tsinghua.edu.cn/epel/7/x86_64/Packages/r/redis-3.2.12-2.el7.x86_64.rpm
http://mirrors.aliyun.com/centos/7/os/x86_64/Packages/logrotate-3.8.6-17.el7.x86_64.rpm
```
