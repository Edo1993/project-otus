# project-otus
wordpress
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

Развернуть кластер

```
vagrant up
```

Для отображения фронта прописать в свой /etc/hosts IP:

```
192.168.10.80 https://wp-project.org/
```

Схема проекта

![Img_alt](https://github.com/Edo1993/project-otus/blob/master/schema.png)

Краткое описание:

Wordpress 
apache и php - на серверах приложения
rsyslog - сбор логов
bacula - для бэкапов
haproxy - балансировка нагрузки
keepalived
innodb mysql СУБД
firewalld на хостах
