# oracle

## 1.安装及初始化

### 1.1 安装

版本：oracle11g

- 傻瓜式安装
- 安装中关闭防火墙
- 第一次用超级管理员登录


### 1.2 初始化

- 建立表空间

`create tablespace phs logging datafile 'D:\app\ckzh1\oradata\phs\phs.dbf' size 4096M autoextend on next 128m extent management local;` 

- 设置及授权用户

`create user phs identified by phs123 default tablespace phs;`
`grant dba,connect,resource to phs;`

> 两个用户 phs tcs 密码 phs123 tcs123

- 同步数据

同步数据`cmd：imp phs/phs123@phs file=D:\docs\dev\phs.dmp full=y`

plsql ，导入sql数据：`@‪D:\docs\众阳\TCS.sql`

## 2.xxx

select userenv('language') from dual;//查询字符集



select a.*, a.rowid from bd_system_config a where a.config_name = 'tjFlag'
-- 健康体检表
select * from hd_personal_health;
-- 黑名单表
select * from bd_ip_banned



select * from HD_PERSONAL_INFO 

select a.*, a.rowid from bd_system_config a where a.config_name like '%心电%'

select a.*, a.rowid from bd_organize_institution a where a.organname like '%卫计局%' and organcode = '370124R30001';

select * from bd_organize_employee where name = 'super'

select * from bd_dd_info where parentid = 'DM01-02'

SELECT * FROM SYS.ALL_SYNONYMS t WHERE t.owner in ('PHS');
CK_ASSESSHEALTH
CK_REGISTINFO
HD_HEALTH_EQUIPMENT_DATA
HD_XD_RESULT

## 3.Oracle配置、连接的bug

### ORA-12541: TNS:无监听程序

OracleOraDb11g_home1TNSListener自动关闭

https://www.cnblogs.com/yx007/p/6732012.html（`netca`命令打开）

https://blog.csdn.net/qq997404392/article/details/73296429

**广建哥解决思路**

1.追踪listener日志：`D:\app\ckzh1\diag\tnslsnr\DESKTOP-PBPORH7\listener\trace`

2.刚开始出现的错误：`SID_DESC`属性错误

3.根据如下配置工具重新配置**监听程序及端口名**

![image-20200504132041676](D:\docs\文档\img\bug1.png)

4.根据`.bak`重新配置的文件 => 比对listener.ora，复制粘贴

<img src="D:\docs\文档\img\bug2.png" alt="image-20200504132423523" style="zoom: 67%;" />

重新配置无效

5.上述配置工具，删除配置再重启，依然无效

6.不知从哪个简书上复制的内容，改变了listener.ora中的`SID_DESC`属性，重启，plsql连接ok

### ORA-12514: TNS:监听程序当前无法识别连接描述符中请求的服务

navicat无法连接，plsql正常连接

![image-20200504133719075](D:\docs\文档\img\bug3.png)

