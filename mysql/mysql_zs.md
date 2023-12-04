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


windows:
管理员身份运行cmd  
net start mysql
mysql -u root -p
（本机密码sha）


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