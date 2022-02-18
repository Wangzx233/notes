# MariaDB默认字符集踩坑

## 问题

存储中文进入数据库时会全部变成`?`

## 原因

查看mariaDB官方文档发现

> 在MariaDB中，默认的字符集[character set](https://mariadb.com/kb/en/data-types-character-sets-and-collations/)为latin1，默认的排序规则为latin1_swedish_ci(但不同的发行版可能会不同，例如[Differences in MariaDB in Debian](https://mariadb.com/kb/en/differences-in-mariadb-in-debian/))。

## 解决方案

### 设置数据库字符集

可以在创建数据库时就设置数据库的字符集，不过需要注意Mariadb和mysql设置字符集方法略有不同

MySQL是在括号里如下：

```mysql
CREATE TABLE tablename (
    ......
 DEFAULT charset = utf8);
```

 

Mariadb 是在括号外如下：

```mariadb
CREATE TABLE tablename (
    ......
) DEFAULT charset = utf8;
```



也可以修改数据库字符集

`alter database DBdata character set utf8;`

### 更改默认字符集

要改变默认的字符集latin1为UTF-8，需要在配置文件my.cnf中进行如下设置：

```
[client]
...
default-character-set=utf8
...
[mysql]
...
default-character-set=utf8
...
[mysqld]
...
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8
...
```

注意，选项`default-character-set`是一个客户端选项，而非服务端选项。

我的配置文件是在/etc/my.cnf

其中有一行`!includedir /etc/my.cnf.d`

所以还是修改`my.cnf.d`目录下的

修改`/etc/my.cnf.d/cilent.cnf`里的[client]

修改`/etc/my.cnf.d/mysql-cilent.cnf`里的[mysql]

修改`/etc/my.cnf.d/server.cnf`里的[mysqld]

重启一下`systemctl restart mariadb.service`

检查一下

`SHOW VARIABLES LIKE 'character_set_%';`



但已经创建的数据库需要手动再改一下字符集

`alter database DBdata character set utf8;`

