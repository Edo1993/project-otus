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

В ходе проверки было выявлено, что логи, например, по входу в приложение, не пишутся.
Что было сделано для исправления:

- изменён конфиг файл wordpress [wp-config.php](https://github.com/Edo1993/project-otus/blob/master/roles/wordpress/templates/wp-config.php)
добавлены строки :

```
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
```

- изменён конфиг файл на клиентах для rsyslog - rsyslog.conf, включена отправка всех сообщений, а не только ошибочных. В playbook это внесено в виде изменений в tasks в роль [rsyslog-sender](https://github.com/Edo1993/project-otus/blob/master/roles/rsyslog-sender/tasks/main.yml)


```
*.* @192.168.10.250:514
```
- на vm с приложениями backend1 и backend2 добавлены конфиг-файлы для отправки логов приложения wordpress на удаленный сервер с логами - mon.

Для отправки логов доступа -  

```
[root@backend1 rsyslog.d]# cat /etc/rsyslog.d/wordpress_access.conf 
$ModLoad imfile

$InputFileName /var/log/httpd/access_log              # or any log file on your server
$InputFileTag backend_httpd_access:                                    # assign a unique tag so you can search on loggly easily
$InputFileSeverity info
$InputFileStateFile httpdlog1
$InputRunFileMonitor                                          # Include this so the imfile module will be able to scan the next file.

if $programname == 'httpd' then @@192.168.10.250
```

Для отправки ошибок -  

```
[root@backend1 rsyslog.d]# cat /etc/rsyslog.d/wordpress_error.conf 
$ModLoad imfile

$InputFileName /var/log/httpd/error_log              # or any log file on your server
$InputFileTag backend_httpd_error:                                    # assign a unique tag so you can search on loggly easily
$InputFileSeverity info
$InputFileStateFile httpdlog1
$InputRunFileMonitor                                          # Include this so the imfile module will be able to scan the next file.

if $programname == 'httpd' then @@192.168.10.250
```

После обновления конфигов - перезапустить сервисы 

```
systemctl restart httpd.service
systemctl restart rsyslog.service
```


3) Redis 

- так же на защите обнаружили, что если положить один из серверов приложения - не сохраняется сессия.
Исправлено: в конфиге redis нужно было закоммитить строку 
```
# bind 127.0.0.1
```
так же был открыт порт 6379 для redis на vm - backend1 / backend2 - соответствующий шаг добавлен в task для роли firewalld.


