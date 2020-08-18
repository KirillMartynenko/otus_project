# Выпускной проект курса OTUS Администратор Linux
#### поток 2020-01

## Тема: Отказоустойчивый почтовый сервер на базе postfix + dovecot
#### Создание проекта основано на знаниях, полученных в ходе обучения на курсе.

### Использованные opensource-технологии
* DRBD
* Pacemaker
* Corosync
* Percona XtraDB Cluster (Mysql + Galera)
* Keepalived
* Nginx
* Fastcgi



### Условия, в которых создавался стенд:
* Vagrant v.2.2.7
* ansible v.2.9.7
* VirtualBox v.5.2.36

### Минимальные требования к железу для развертывания
* Intel Core i5
* RAM 16 GB
* Подключение к сети Internet 20 Mbit/s

### Рекомендуемые требования к железу для развертывания
* Intel Core i7
* RAM 16 GB
* Подключение к сети Internet 100 Mbit/s



### В качестве операционной системы на всех хостах стенда используется Centos 7
По умолчанию отключены SELinux и firewalld


### Схема стенда
![schema](otus-project.png)


### Описание стенда

Кластер построен из 8-ми хостов. Условно их можно объединить в следующие группы:

* Фронтэнд

* Бэкэнд

* База данных

##### Хосты

| Хост | IP-address |
|------|------------|
| ngx01: | 192.168.11.111 |
| ngx02: | 192.168.11.112 |
| postfix01: | 192.168.11.113 |
| postfix02: | 192.168.11.114 |
| postfix03: | 192.168.11.115 |
| pxc1: | 192.168.11.120 |
| pxc2: | 192.168.11.121 |
| pxc3: | 192.168.11.122 |

##### Виртуальные ip-адреса

| IP-address | Хосты |
|------------|-------|
| 192.168.11.100 | ngx0\[12\] |
| 192.168.11.110 | postfix0\[1-3\] |
| 192.168.11.116 | postfix0\[1-3\] |



#### Фронтэнд

Состоит из двух нод, на каждой работает Nginx, который посредством fastcgi обеспечивает работу web-приложения Roundcube.
Roundcube обращается к базе данных для аутентификации пользователей и к postfix/dovecot для работы с почтовыми сообщениями.

Для обеспечения отказоустойчивости, на хостах поднят виртуальный ip-адрес силами keepalived в режиме active/backup.


#### Бэкэнд

Самая нагруженная часть кластера. Состоит из 3-х нод, для обеспечения кворума.
Здесь так же в режиме active/backup подняты следующие сервисы:

* Drbd
: используется для хранения содержимого почтовых ящиков

* Postfix
: MTA + LDA

* Dovecot
: MDA

* Proxysql
: Проксирует запросы к БД на мастер-ноду кластера Xtradb

* Web-приложение postfixadmin (nginx + fastcgi)
: технический фронтэнд, для управления доменами и почтовыми ящиками.


Отказоусточивость обеспечивается связкой pacemaker + corosync.
Доступность данных в случае отработки фейловера обеспечивает drbd.

##### Древовидная иерархия ресурсов pacemaker:

```
pacemaker
└── DrbdData
    ├── proxysql_ip
    │   └── proxysql
    └── virtual_ip
        ├── dovecot
        └── postfix
```

Таким образом, ключевой ресурс - DrbdData, а все остальные зависят от него.
В случае отработки фейловера, первым на рабочую ноду кластера переезжает drbd (поднимается сервис и монтируется блочное устройство).
Следом за drbd перезжают виртуальные ip:
- virtual_ip для входящих соединений к postfix, dovecot и postfixadmin
- proxysql_ip для входящих соединений к proxysql
В соответствии с тем как это отражено в древовидной структуре, вслед за вирутальными ip на выбранную кворумом ноду переезжают оставшиеся зависимые ресурсы - postfix, dovecot и proxysql.

Для предотвращения сплит-брейна данных на блочном устройстве используется fencing - на каждой ноде поднят фенсинг-агент, предотвращающий работу с распределенным блочным устройством, в случае "выпадения" ноды из кластера.



#### База данных

Как уже было упомянуто выше, в качестве хранилища аутентификационных данных аккаунтов пользователей используется Percona XtraDB Cluster в связке с Galera.
Одна нода работает в режиме ведущего и принимает запросы на запись (INSERT/UPDATE), оставшиеся две ноды ведомые и принимают запросы на чтение (SELECT). Роли согласовываются с proxysql, который знает какую роль выполняет та или иная нода и отправляет подходящие запросы на нужные ноды.

В случае падения одной из нод, отработка фейловера и дальнейшее включение упавшей ноды в кластер (по результату возвращения хоста в функциональное состояние) происходит в автоматическом режиме. Определение новой ведущей ноды в моменте фейловера обеспечивается кворумом.

В случае "выпадения" из кластера более одной ноды, происходит разрушение кластера, что потребует человеческого вмешательства и проведения работ по его восстановлению, а именно потребуется так называемый bootstrap ведущей ноды при выключенных ведомых, затем включение ведомых нод и затем включение ведущей ноды в обычном режиме - подробное описание операции есть по ссылке приведенной внизу.



### Развертывание стенда
Производится путем последовательного выполнения следующих команд:

```
git clone https://github.com/zradeg/otus_project

cd otus_project

vagrant up

ansible-playbook playbook.yml
```

Дождавшись окончания команд, необходимо произвести настройку почтовых систем через postfixadmin: завести аккаунт суперадминистратора, домен(ы) и почтовые ящики.


#### Заведение аккаунта суперадминистратора

Заходим на страницу настройки 
http://192.168.11.113/setup.php

При загрузке страницы должен отработать скрипт обновления таблиц базы. В случае, если внизу будет сообщение

```
DEBUG INFORMATION:
Invalid query: Percona-XtraDB-Cluster prohibits use of DML command on a table (postfix.log) without an explicit primary key with pxc_strict_mode = ENFORCING or MASTER
```

Необходимо будет выполнить таск:
```
ansible-playbook playbook.yml -vv --tags=pxc_strict_mode
```

После чего перезагрузить страницу.

В появившихся после обновления полях вводим superpassword (по умолчанию postfix2020), логин администратора почтовой системы (в формате email-адреса) и его пароль:

```
Setup password: postfix2020
Администратор: admin@postfix.loc (например)
Пароль: произвольный
```

После чего попадаем в консоль управления системой, где есть возможность создавать домены, почтовые ящики и алиасы в них и производить их настройку.

В первую очередь нужен домен - создаем его в соответствующем разделе. Для примера я создал домен postfix.loc
И сразу же создаем новый почтовый ящик (для примера admin@postfix.loc)

После этого почтовая система одного ящика готова, можно авторизовываться в roundcube:

http://192.168.11.110
На открывшейся странице авторизовываемся, используя реквизиты только что созданного ящика, после чего можно отправлять и принимать письма.

В качестве проверки можно произвести отправку письма через телнет:

```
Trying 192.168.11.113...
Connected to 192.168.11.113.
Escape character is '^]'.
220 mail.postfix.loc ESMTP Postfix
EHLO mail.postfix.loc
250-mail.postfix.loc
250-PIPELINING
250-SIZE 20000000
250-STARTTLS
250-AUTH PLAIN LOGIN
250-AUTH=PLAIN LOGIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN
mail from:<admin@postfix.loc>
250 2.1.0 Ok
rcpt to:<admin@postfix.loc>
250 2.1.5 Ok
DATA
354 End data with <CR><LF>.<CR><LF>
From: Admin <admin@postfix.loc>
To: Admin <admin@postfix.loc>
Subject: Test!!!
Content-Type: text/plain   

Test!!!
Bye
.
250 2.0.0 Ok: queued as 59453DF8E3B
quit
```


### В процессе подготовки стенда были использованы следующие материалы:
https://www.linbit.com/drbd-user-guide/drbd-guide-9_0-en/

https://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html-single/Clusters_from_Scratch/index.html

https://www.percona.com/doc/percona-xtradb-cluster/LATEST/index.html

https://github.com/roundcube/roundcubemail/wiki

https://postfix-configuration.readthedocs.io/en/latest/postfixadmin/

https://serveradmin.ru/nastroyka-postfix-dovecot-centos-8/
