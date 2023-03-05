# CentOS

## yum常用命令

```shell
yum install softwarename  #安装
yum remove softwarename #卸载软件
yum list softwarename #查看软件源中是否有此软件
yum list all #列出所有软件名称
yum list installed #列出已经安装的软件名称
yum list available #列出可以用yum安装的软件
yum clean all #清空yum缓存
yum search softwareinfo #根据软件信息搜索软件名字（如，使用search web搜索web浏览器）
yum whatprovides filename #在yum源中查找包含filename文件的软件包（如，whatprovides rm搜索汉含rm的软件，命令实质上是文件）
yum update #更新软件，会存在未知问题，一般不对服务器升降级
yum history #查看系统软件改变历史
yum reinstall softwarename #重新安装
yum info softwarename #查看软件信息
yum groups list #查看软件组信息
yum groups info softwarename #查看软件组内包含的软件
yum groups install softwarename #安装组件
yum groups remove softwarename #卸载组件
yum clean all #清理缓存
```

## 实用命令

### 进程类

https://www.cnblogs.com/aipiaoborensheng/p/7676364.html

#### 查看进程

- `ps aux|grep xxx`

- `ps -a`

-  查看磁盘使用情况：`df -h`

- 查看指定文件下所有文件大小： `du -h /data/`

- 查看当前文件夹内所有文件的大小：du -sh 

#### 杀死进程

​		`kill -9 pid`

