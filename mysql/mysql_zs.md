## 大小写
* SQL 对大小写不敏感
但小心cmd时可能会这样
* Linux下mysql默认区分大小写
* Windows下mysql默认不区分大小写
## 启动
linux:
sudo /etc/init.d/mysql start
sudo systemctl start mysql
sudo service mysql start（wsl只能这样启动）

sudo service mysql status（查看状态）
sudo systemctl status mysql

sudo mysql -uroot -p
### 密码设置

查看初始密码策略
`SHOW VARIABLES LIKE 'validate_password%';` 
```
#设置密码安全等级为LOW
set global validate_password_policy=LOW;
 
#设置最短密码长度为3
set global validate_password_length=3;
 
#设置必须包含大小写字数量符为0
set global validate_password_mixed_case_count = 0;
 
#设置必须包含特殊字符数量为0
set global validate_password_special_char_count = 0;
 
#设值必须包含数字的数量为0
set global validate_password_number_count = 0;
 
#修改mysql密码为123
alter user root@localhost identified by '123';
 
#授权root用户可以在任意IP使用密码123登录
grant all privileges on *.* to 'root'@'%' identified by '123' with grant option;
 
#刷新权限
flush privileges;
```

windows:
管理员身份运行cmd  
net start mysql
mysql -u root -p
（本机密码sha）

## 远程连接mysql
连接账户的主机需要在账户创建时授权，并且需要具有对某些表的访问权限。


grep 'temporary password' /var/log/mysqld.log （查询自动生成的密码）
## 事务
mysql每条语句都是一个事务，事务自动提交。
[here](https://blog.csdn.net/wang_luwei/article/details/119619105)


## 别人总结的黑马mysql笔记
[here](https://dhc.pythonanywhere.com/article/public/1/)

## mvcc
[here](https://juejin.cn/post/7016165148020703246)

## MySQL三大日志：binlog、redo log和undo log
[here](https://zhuanlan.zhihu.com/p/190886874)

## view


## 当前读和快照读
[here](https://blog.csdn.net/weixin_44844089/article/details/115532014#:~:text=%E5%BF%AB%E7%85%A7%E8%AF%BB%EF%BC%8C%E9%A1%BE%E5%90%8D%E6%80%9D%E4%B9%89,%E4%BF%9D%E8%AF%81%E8%AF%BB%E5%86%99%E4%B8%8D%E5%86%B2%E7%AA%81%E3%80%82)

## mysql小技巧
* select一行显示不下，后面加`\G`列显示


## MySQL原理

### 联合索引Btree
[here](https://blog.csdn.net/wtopps/article/details/114558152)

[范围索引失效](https://blog.csdn.net/yuruixin_china/article/details/105145049)

* 在MySQL中，联合索引是一种包含多个列的索引。当你在一个表上创建了一个联合索引，并且查询涉及到该索引的多个列时，MySQL可以使用这个索引来优化查询性能。
* 范围查询包括 >= 和 > 操作符，它们在使用联合索引时有一些区别：

* = 操作符：

* ``>=`` 表示大于或等于的条件，包括等于。
当你执行一个范围查询，例如 WHERE col1 >= value1 AND col2 >= value2，如果你的联合索引覆盖了 col1 和 col2，MySQL 可能会使用这个联合索引来优化查询。
* `>` 操作符：

* `>` 表示大于的条件，不包括等于。
当你执行一个范围查询，例如 WHERE col1 > value1 AND col2 > value2，如果你的联合索引覆盖了 col1 和 col2，MySQL 可能仍然会使用这个联合索引来优化查询，但它不会考虑等于条件。
* 总结：大与不知到范围完全无法利用范围索引，**>=有了边界条件可以利用第范围索引进行优化**。

### 索引跳表扫描
[here](https://cloud.tencent.com/developer/article/1437313)