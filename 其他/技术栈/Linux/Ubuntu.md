# Ubuntu

## 1 初始化

### 1.1 换源

#### Ubuntu18.04更换国内源：

https://www.cnblogs.com/boundless-sky/p/11576373.html

### 1.2 连接服务器

#### 1.2.1 ssh方法

`ssh username@127.0.0.1 -p 22`

#### 1.2.2 ssh shell软件



## 2 网络配置

ssh终端管理软件：SSH_Shell

开启远程连接：https://blog.csdn.net/weixin_34018202/article/details/88760631

ifconfig

## 3 常用命令

### 3.1 apt-get

​	https://blog.csdn.net/bjzhaoxiao/article/details/81224550



查看内存使用

- free -mlht

- - http://www.cnblogs.com/wayneiscoming/p/7865068.html

- top

- - https://blog.csdn.net/anxpp/article/details/52453134

进程

https://www.cnblogs.com/aipiaoborensheng/p/7676364.html

- 查看

- - ps aux|grep xxx
  - ps -a
  - ps ...等等
  - df -h 查看磁盘使用情况
  - 查看指定文件下所有文件大小 du -h /data/
  - du -sh 查看当前文件夹内所有文件的大小

- 杀死进程

- - kill -9 pid

后台运行

- xxx  &
- xxx -d
- bg 查看后台进程数
- fg   后台进程转到前台

服务

- systemctl status xxx

- systemctl start xxx

- systemctl stop xxx

- systemctl系列参考:https://www.cnblogs.com/lxjshuju/p/7183689.html

- service系列

- 加入服务启动

- 开机自启/关闭自启

- - chkconfig servicename on/off

查看端口:

- netstat -anp

查看配置文件

mysql:

mysql --help|grep 'my.cnf' 

查找文件：

find / -name dump.rdb

## 4 常用软件安装

传送文件：https://blog.csdn.net/u013828589/article/details/73605594

### 4.1 Redis

#### 修改配置文件：

1.开启远程连接：

​	https://blog.csdn.net/guo_yan_gy/article/details/82877994

2.dir

3.后台启动 yes

4.logfile "6379.log"

5.dump.rdb => dump-6379.rdb

...

#### 4.1.1 手动编译版本

https://blog.csdn.net/u010623954/article/details/80037078

##### 1.下载redis

- 上传：`scp -p 22 redis-5.0.7.tar.gz messi@172.20.10.12:/home/messi/download`
- 下载：`wget http://download.redis.io/releases/redis-x.x.x.tar.gz` 

2.配置redis.conf

3.设置后台自启动

https://blog.csdn.net/zyccode/article/details/90106396

#### 4.1.2 apt-get install 版本

- 查询redis安装目录：`whereis redis-server/rediscli/redis.conf`

- redis.conf目录：/etc/redis 

  





