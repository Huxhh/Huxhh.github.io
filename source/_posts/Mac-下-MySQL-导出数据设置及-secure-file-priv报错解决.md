---
title: Mac 下 MySQL 导出数据设置及--secure-file-priv报错解决
date: 2019-09-29 16:52:15
header-img: C5DJ0zrM/006y8m-N6ly1g7gi3do6tuj30ls0ban2d.jpg
tags: MySQL
---

在把MySQL中的数据导入到一个文件当中的时候遇到了一些问题，在此记录一下。

### 语句
MySQL中导出数据的语句是 **SELECT ... INTO OUTFILE** ，举例来说：

```
select * from test_info   
 
into outfile '/tmp/test.csv'   
 
fields terminated by ','        --字段间以,号分隔
 
optionally enclosed by '"'      --字段用"号括起
 
escaped by '"'   　　　　　　     --字段中使用的转义符为"
 
lines terminated by '\r\n';　　  --行以\r\n结束
```
命令不是特别复杂，本文的重点也不在此。重点在于如果没有提前修改配置的话，在运行此命令时会出问题。

### 修改配置
首先遇到的问题是，在我使用的MySQL 5.7中报出了以下错误：

```
ERROR 1290 (HY000): The MySQL server is running with the --secure-file-priv option so it cannot execute this statement
```

经过查询得知是需要修改--secure-file-priv参数，这一参数可以使用语句 show variables like '%secure%'; 查询，如果结果为NULL，则需要更改，如下图所示:
![1](https://tva1.sinaimg.cn/large/006y8mN6ly1g7ghqnzk79j31o60to1kx.jpg)

修改改参数需要到MySQL的配置文件中修改，mac中MySQL的配置文件为 `/private/etc/my.cnf`

打开此文件，在 `[musqld]` 下面加上 `secure_file_priv='/tmp/path/'` 即可，如下图所示：
![2](https://tva1.sinaimg.cn/large/006y8mN6ly1g7ghskip44j30zm0e2qfg.jpg)

**要注意的是：Mac当中只可以把该目录设置在`/tmp/`文件夹下，设置在其他位置会导致MySQL启动报错**

但此时直接导出文件仍会报错:
```
ERROR 1 (HY000): Can't create/write to file 'test.csv' (Errcode: 13 - Permission denied)
```

这里的原因是MySQL是使用_mysql 用户来写文件的，而该用户对该文件夹没有操作权限，因此需要执行以下语句：
```
chown _mysql:_mysql /private/tmp/mysql_output/
```

最后重启MySQL，重新执行命令发现执行成功。




