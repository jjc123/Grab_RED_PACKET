# Grab_RED_PACKET
java配置SSM纯注解+Redis实现高并发抢红包项目

## 前言：
前段时间学习点Redis，这次结合ssm实现一个高并发抢红包的项目。
跟以前不一样的：
1. 基于java配置SSM，而并非XML.为什么要这样呢？
找了点网络上的答案：
```
在使用Spring开发时，我们经常会看到各种各样xml配置，过于繁多的xml配置显得复杂烦人。  
在Spring3之后，Spring支持使用JavaConfig来代替xml配置，  
这种方式也得到越来越多人的推荐，甚至在Spring Boot的项目中，基本上已经见不到xml的影子了。
```
2. 抢红包的记录使用Lua保存到Redis内存中，最后再异步使用批量事务插入到Mysql。
目的：Redis内存响应速度比Mysql硬盘响应速度快。
**这个项目的目的也是为了实现上面的两个内容。**
***
### 首先我们先配置数据库：
两个表：
T_RED_PACKET存红包信息
T_USER_RED_PACKET存用户抢红包信息
```
create table T_RED_PACKET(
id int(12) not null auto_increment,
user_id int(12) not null,
amount decimal(16,2) not null,
send_date timestamp not null,
total int(12) not null,
unit_amount decimal(12) not null,
stock int(12) not null,
version int(12) default 0 not null,
note varchar(256) null,
primary key clustered(id)
);
create table T_USER_RED_PACKET(
id int(12) not null auto_increment,
red_packet_id int(12) not null,
user_id int(12) not null,
amount decimal(16,2) not null,
grab_time timestamp not null,
note varchar(256) null,
primary key clustered (id) 
);
insert into T_RED_PACKET(user_id,amount,send_date,total,unit_amount,stock,note)
values(1,200000.00,now(),20000,10.00,20000,'20万元金额，2万个小红包 每个10元');
```
注意：
```
amount ：金额 要用decimal而不是double
send_date ：发红包时间
stock：剩余的红包数
primary key clustered (id) 是设置主键和聚集索引
```
**题外话：**
`Ubuntu下设置MySQL字符集为utf8`

**1.mysql配置文件地址**
/etc/mysql/my.cnf

**2.在[mysqld]在下方添加以下代码**
[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake

**3.重启mysql服务**
sudo service mysql restart

4.检测字符集是否更新成utf8.
进入mysql，mysql -u root -p,输入show variables like '%character%' 查看字符集
+--------------------------+----------------------------+
| Variable_name | Value |
+--------------------------+----------------------------+
| character_set_client | utf8 |
| character_set_connection | utf8 |
| character_set_database | utf8 |
| character_set_filesystem | binary |
| character_set_results | utf8 |
| character_set_server | utf8 |
| character_set_system | utf8 |
| character_sets_dir | /usr/share/mysql/charsets/ |

### 配置Redis：
`注意：`redis的数据即使电脑关机下次打开redis的时候还会存在

进入redis的src然后./redis-server ../redis.conf &指定配置文件启动而且是后台启动
再打开新的客户端：./redis-cli
127.0.0.1:6379> hset red_packet_2 stock 20000        （保存红包库存 也可以用来直接修改）
(integer) 0
127.0.0.1:6379> hset red_packet_2 unit_amount 10     (保存单个红包库存)
(integer) 0
**题外话：**
```
hget red_packet_2 stock   hash获取该键的值
ltrim red_packet_list_2 1 0  清空list对应键的值
RPUSH red_packet_list_2 c  list添加单个元素
LRANGE red_packet_list_2 0 -1 获取对应list的所有值
LLEN red_packet_list_2 获取list长度
```
***
