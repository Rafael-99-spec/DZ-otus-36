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
После ÿтого можно запуститþ службу:
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

mysql> 
```
