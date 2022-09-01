# CentOS 7 安装 MySQL 5.7 

## 配置 yum 源

在 [https://dev.mysql.com/downloads/repo/yum/](https://links.jianshu.com/go?to=https%3A%2F%2Fdev.mysql.com%2Fdownloads%2Frepo%2Fyum%2F) 找到 yum 源 rpm 安装包

![img](https://upload-images.jianshu.io/upload_images/1458376-6c3dece1d8bd0650.png?imageMogr2/auto-orient/strip|imageView2/2/w/1193/format/webp)

安装 mysql 源

```csharp
# 下载
shell> wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
# 安装 mysql 源
shell> yum localinstall mysql57-community-release-el7-11.noarch.rpm
```

用下面的命令检查 mysql 源是否安装成功

```bash
shell> yum repolist enabled | grep "mysql.*-community.*"
```

## 安装 MySQL

使用 yum install 命令安装

```undefined
shell> yum install -y mysql-community-server
```

问题：

Failing package is: mysql-community-libs-compat-5.7.37-1.el7.x86_64

原因是Mysql的GPG升级了，需要重新获取
使用以下命令即可

```
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
```



## 启动 MySQL 服务

在 CentOS 7 下，新的启动/关闭服务的命令是 `systemctl start|stop`

```undefined
shell> systemctl start mysqld
```

用 `systemctl status` 查看 MySQL 状态

```undefined
shell> systemctl status mysqld
```



## 设置开机启动

```bash
shell> systemctl enable mysqld
# 重载所有修改过的配置文件
shell> systemctl daemon-reload
```



## 修改 root 本地账户密码

mysql 安装完成之后，生成的默认密码在 `/var/log/mysqld.log` 文件中。使用 grep 命令找到日志中的密码。

```bash
shell> grep 'temporary password' /var/log/mysqld.log
```

![img](https:////upload-images.jianshu.io/upload_images/1458376-6694dca4f9eb39a3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1137/format/webp)

查看临时密码

首次通过初始密码登录后，使用以下命令修改密码

```bash
shell> mysql -uroot -p
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!'; 
```

或者

```bash
mysql> set password for 'root'@'localhost'=password('MyNewPass4!'); 
```

以后通过 update set 语句修改密码

```bash
mysql> use mysql;
mysql> update user set password=PASSWORD('MyNewPass5!') where user='root';
mysql> flush privileges;
```

> 注意：mysql 5.7 默认安装了密码安全检查插件（validate_password），默认密码检查策略要求密码必须包含：大小写字母、数字和特殊符号，并且长度不能少于8位。否则会提示 ERROR 1819 (HY000): Your password does not satisfy the current policy requirements 错误。查看 [MySQL官网密码详细策略](https://links.jianshu.com/go?to=https%3A%2F%2Fdev.mysql.com%2Fdoc%2Frefman%2F5.7%2Fen%2Fvalidate-password-options-variables.html%23sysvar_validate_password_policy)



# Rocky 8 安装 MySQL 5.7 

### 卸载原有的 MySQL

1、查看已安装的 MySQL

```sh
[root@kaiwen ~]# rpm -qa |grep mysql
mysql-common-8.0.26-1.module+el8.4.0+652+6de068a7.x86_64
mysql-8.0.26-1.module+el8.4.0+652+6de068a7.x86_64
```

2、卸载已经安装的 MySQL

```sh
[root@kaiwen ~]# rpm -e mysql-8.0.26-1.module+el8.4.0+652+6de068a7.x86_64
[root@kaiwen ~]# 
[root@kaiwen ~]# rpm -e mysql-common-8.0.26-1.module+el8.4.0+652+6de068a7.x86_64
```



### 安装 MySQL 5.7

1、下载mysql的rpm

```sh
wget http://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
```

2、安装MySQL的rpm

```sh
rpm -ivh mysql80-community-release-el7-3.noarch.rpm
```

这时候会在 /etc/yum.repos.d/目录下生成两个文件
mysql-community.repo和mysql-community-source.repo

修改repo文件（安装5.7，不要8.0）

```sh
cd /etc/yum.repos.d/
vim mysql-community.repo
```

将 [mysql57-community] 下的 enabled 设置为1，表示打开5.7
将 [mysql80-community] 下的 enabled 设置为0，表示关闭8.0

修改完保存退出

3、安装Mysql

```sh
yum -y install mysql-community-server
```

问题1

yum install mysql-community-server 出现以下错误

```
Error:GPG check FAILED
```

原因

这由于源key错误导致的dnf或者[yum](https://so.csdn.net/so/search?q=yum&spm=1001.2101.3001.7020)（软件包管理器）安装软件失败。

解决

添加 --nogpgcheck 选项就能部分解决此问题。

```sh
yum -y install mysql-community-server --nogpgcheck
```

问题2

rocky8 安装mysql57 运行 yum install mysql-community-server 出现以下错误:

```sh
No match for argument: mysql-community-server
Error: Unable to find a match: mysql-community-server
```

原因

基于RHEL8和Oracle Linux 8的基于EL8的系统包括默认情况下启用的MySQL模块。 除非禁用此模块，否则它将屏蔽MySQL存储库提供的软件包。

解决

执行如下命令即可

```sh
yum module disable mysql
```

问题3

rocky8 安装mysql57 运行 yum install mysql-community-server 出现以下错误:

```sh
Error: Transaction test error:
  file /etc/my.cnf from install of mysql-community-server-5.7.39-1.el7.x86_64 conflicts with file from package mariadb-connector-c-config-3.1.11-2.el8_3.noarch
```

原因

 mysql-community-server-5.7.39-1.el7.x86_64  和 mariadb-connector-c-config-3.1.11-2.el8_3.noarch 冲突

解决

执行命令，卸载 mariadb-connector-c-config.noarch

```sh
yum remove mariadb-connector-c-config.noarch
```

4、验证 MySQL 版本

```sh
[root@kaiwen yum.repos.d]# mysql --version
mysql  Ver 14.14 Distrib 5.7.39, for Linux (x86_64) using  EditLine wrapper
```

### 启动并查看状态

```sh
[root@kaiwen yum.repos.d]# systemctl start mysqld.service 
[root@kaiwen yum.repos.d]# systemctl status mysqld.service 
```

### 设置开机启动

```sh
systemctl enable mysqld
systemctl daemon-reload
```

### 修改默认密码

1、查看默认密码

mysql安装完成之后，在/var/log/mysqld.log文件中给root生成了一个默认密码。通过下面的方式找到root默认密码，然后登录mysql进行修改：

```sh
[root@kaiwen yum.repos.d]# grep 'temporary password' /var/log/mysqld.log
2022-07-27T06:19:48.916621Z 1 [Note] A temporary password is generated for root@localhost: ?!Y;DfY4jCpx
```

注意冒号后面的那个空格不需要。

登陆之后会发现很多功能都不能用，只有修改密码才能进行正常操作。

2、修改密码策略

```mysql
mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_check_user_name    | OFF    |
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
```

validate_password_policy取值

```
0 or LOW     只验证长度
1 or MEDIUM  验证长度、数字、大小写、特殊字符
2 or STRONG  验证长度、数字、大小写、特殊字符、字典文件
```

MySQL 默认密码策略为 MEDIUM，所以你更改密码必须满足：数字、小写字母、大写字母 、特殊字符、长度至少8位。

如需设置简单密码则需更改密码策略，否则可跳过此步。

进入 mysql，

```sh
[root@kaiwen yum.repos.d]# mysql -uroot -p
Enter password:输入默认密码
```

修改策略，及密码长度

```mysql
mysql> set global validate_password_policy=0; #最低策略
mysql> set global validate_password_length=1; #最短长度验证
```

查看 mysql 初始的密码策略

```mysql
mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| validate_password_check_user_name    | OFF   |
| validate_password_dictionary_file    |       |
| validate_password_length             | 4     |
| validate_password_mixed_case_count   | 1     |
| validate_password_number_count       | 1     |
| validate_password_policy             | LOW   |
| validate_password_special_char_count | 1     |
+--------------------------------------+-------+
```

3、修改密码

```mysql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';

   或者

mysql> SET PASSWORD = PASSWORD('MyNewPass4!');
```

### 