# project-otus


Построение отказоустойчивого кластера на базе wordpress, MySQL, HAProxy
________________________________________________________________________________________
Требования:

В итоге в проект должны быть включены:

— как минимум 2 узла с СУБД; 

— минимум 2 узла с веб-серверами; 

— настройка межсетевого экрана (запрещено всё, что не разрешено); 

— скрипты резервного копирования; 

— центральный сервер сбора логов (Rsyslog/Journald/ELK). 

Для реализации кластера можно использовать такие технологии, как pacemaker+corosync/hearbeat, HAproxy, VRRP, CEPH, ConsulDNS, кластерные решения для СУБД. 
________________________________________________________________________________________


Краткое описание:

 - Wordpress 

 - apache и php - на серверах приложения

 - rsyslog - сбор логов

 - bacula - для бэкапов

 - haproxy - балансировка нагрузки

 - keepalived

 - innodb mysql СУБД

 - firewalld на хостах

Схема проекта

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/schema1.png)

Развернуть кластер

```
vagrant up
```

Для отображения фронта прописать в свой /etc/hosts IP:

```
192.168.10.80 https://wp-project.org/
```

____________________________________________________________________________________________________________________
Итого 

- во время презентации полетел кластер innodb mysql 

- логи rsyslog по серверам приложения неинформативные чуть более, чем полностью.



1) Исправляем проблему с кластером 

помогли статьи 
[Working with InnoDB Cluster](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/mysql-innodb-cluster-working-with-cluster.html)
[MySQL Shell API - Cluster Class Reference](https://dev.mysql.com/doc/dev/mysqlsh-api-javascript/8.0/classmysqlsh_1_1dba_1_1_cluster.html)

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/1.png)

Нода sqlnode2 в статусе MISSING, поэтому кластер не в том состоянии, что нас устроит -
 
```
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures. 1 member is not active", 
```

vm для sqlnode2 активна, поэтому пробую через mysqlsh подключиться к ноде sqlnode2 - не прокатывает.

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/2.png)

Снова подключаюсь к корректной ноде, выполняю команды
```
var cluster=dba.getCluster(); cluster.status()
cluster.rescan()
```

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/3.png)

Команда ```cluster.rescan()``` находит, что нода *sqlnode2* не является частью кластера, удаляем её.
При запросе статуса кластера видим, что сейчас в него входят только 2 ноды - *sqlnode1*, *sqlnode3*.

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/4.png)

На vm *sqlnode2* перезапустила сервис mysql

```
systemctl restart mysqld.service
```

После рестарта - на vm с запущенным mysqlsh продолжила

```
var cluster=dba.getCluster(); cluster.addInstance('cluster@sqlnode2:3306')
cluster.rescan()
```
Добавили ноду *sqlnode2* в кластер

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/5.png)

Проверяем состояние кластера - ``` var cluster=dba.getCluster(); cluster.status()``` - все 3 ноды в статусе ONLINE, 
статус кластера 
```
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
```

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/6.png)

2) Логи
