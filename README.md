# ДЗ №36 Mysql

```
Развернуть базу из дампа и настроить репликацию
Цель: В результате выполнения ДЗ студент развернет базу из дампа и настроит репликацию.
В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp
Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:
| bookmaker |
| competition |
| market |
| odds |
| outcome |

* Настроить GTID репликацию

варианты которые принимаются к сдаче
- рабочий вагрантафайл
- скрины или логи SHOW TABLES
* конфиги
* пример в логе изменения строки и появления строки на реплике
```

### Практика
Клонируем нужный репозиторий на локальный компьютер.
Установим для начала mysql на master - 
```
wget https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
md5sum mysql57-community-release-el7-9.noarch.rpm
rpm -ivh mysql57-community-release-el7-9.noarch.rpm
yum install mysql-server -y
```
Установливаем percona-server-57:
```
yum install http://www.percona.com/downloads/percona-release/redhat/0.1-6/percona-release-0.1-6.noarch.rpm -y
yum install Percona-Server-server-57 -y
```
Копируем конфиги из /vagrant/conf.d в /etc/my.cnf.d/
```
[root@master vagrant]# cp /vagrant/conf/conf.d/* /etc/my.cnf.d/ 
```
После этого можно запустить службу:
```
[root@master vagrant]# systemctl start mysqld
```
При установке Percona автоматически генерирует пароль для пользователя root и кладет его в файл /var/log/mysqld.log:
```
[root@master vagrant]# cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
FODssV<.=3Sa
```
Подключаемся к mysql и меняем пароль для доступа к полному функционалу:
```
[root@master vagrant]# mysql -uroot -p'FODssV<.=3Sa'
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.32

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> ALTER USER USER() IDENTIFIED BY 'P@ssw0rd';
Query OK, 0 rows affected (0.00 sec)

mysql> 
```
Следует обратить внимание, что атрибут server-id на мастер-сервере должен обязательно
отличаться от server-id слейв-сервера. Проверить какая переменная установлена в текущий
момент можно следующим образом:
```
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)
```
Убеждаемся что GTID включен:
```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
```
Создадим тестовую базу bet и загрузим в нее дамп и проверим:
```
mysql> CREATE DATABASE bet;
Query OK, 1 row affected (0.00 sec)

mysql> exit
[root@master vagrant]# mysql -uroot -p -D bet < /vagrant/bet.dmp
mysql> USE bet;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> SHOW TABLES;
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
7 rows in set (0.00 sec)
```
Создадим пользователя для репликации и даем ему права на эту самую репликацию:
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT user,host FROM mysql.user where user='repl';
+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0.00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
Query OK, 0 rows affected, 1 warning (0.00 sec)
```
Дампим базу для последующего залива на слейв и игнорируем таблицу по заданию:
```
[root@master vagrant]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
Enter password: 
```
На этом настройка Master-а завершена. Файл дампа нужно залить на слейв.

Так же точно копируем конфиги из /vagrant/conf.d в /etc/my.cnf.d/
```
[root@slave vagrant]# cp /vagrant/conf/conf.d/* /etc/my.cnf.d/
```
Правим server-id = 2
```
mysql> SET GLOBAL server_id=2;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           2 |
+-------------+
1 row in set (0.00 sec)
```
Раскомментируем в /etc/my.cnf.d/05-binlog.cnf строки:
```
replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event
```
Таким образом указываем таблицы которые будут игнорироваться при репликации
Заливаем дамп мастера(сделал через общую nfs-пакпку) и убеждаемся что база есть и она без лишних таблиц:
```
mysql>  SHOW DATABASES LIKE 'bet';
+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
1 row in set (0.01 sec)
mysql> SHOW TABLES;
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| market           |
| odds             |
| outcome          |
+------------------+
```
Запускаем слейв базу
```
mysql> CHANGE MASTER TO MASTER_HOST = "192.168.11.150", MASTER_PORT = 3306,
MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2018", MASTER_AUTO_POSITION = 1;
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
 Slave_IO_State: Waiting for master to send event
 Master_Host: 192.168.11.150
 Master_User: repl
 Master_Port: 3306
 Connect_Retry: 60
 Master_Log_File: mysql-bin.000001
 Read_Master_Log_Pos: 313
 Relay_Log_File: mysql2-relay-bin.000002
 Relay_Log_Pos: 526
 Relay_Master_Log_File: mysql-bin.000001
 Slave_IO_Running: Yes
 Slave_SQL_Running: Yes
 mysql> CHANGE MASTER TO MASTER_HOST = "192.168.11.150", MASTER_PORT = 3306,
MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2018", MASTER_AUTO_POSITION = 1;
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
 Slave_IO_State: Waiting for master to send event
 Master_Host: 192.168.11.150
 Master_User: repl
 Master_Port: 3306
 Connect_Retry: 60
 Master_Log_File: mysql-bin.000001
 Read_Master_Log_Pos: 313
 Relay_Log_File: mysql2-relay-bin.000002
 Relay_Log_Pos: 526
 Relay_Master_Log_File: mysql-bin.000001
 Slave_IO_Running: Yes
 Slave_SQL_Running: Yes
 ```
 Диагностика репликации на мастере
 ```
 mysql> USE bet;
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
4 rows in set (0.00 sec)

mysql> 
```
И тоже самое видим в слейве
```
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
4 rows in set (0.00 sec)

mysql> 
```
