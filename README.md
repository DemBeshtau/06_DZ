# Управление пакетами. Дистрибьюция софта. #
1. Создать свой RPM пакет.<br/>
2. Создать свой репозитарий и разместить там ранее собранный RPM.<br/>
3. Реализовать вышеуказанное в Vagrant, либо развернуть через NGINX и дать ссылку на репозиторий.<br/>
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads);<br/>
&ensp;&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14 <br/>
&ensp;&ensp;и образа CentOS 8 версии 2011.0.
### Ход решения ###
### 1. Сборка пакета NGINX с поддержкой openssl ###
&ensp;&ensp;Установка ПО необходимого для сборки пакетов и загрузка исходных кодов программ<br/> 
производится во время запуска образа операционной системы.<br/>
1.1. Установка SRPM пакета NGINX, после которой создаётся дерево каталогов для сборки rpmbuild:<br/>
```shell
[root@rpmtest ~]# rpm -i nginx-1.20.2-1.el7.ngx.src.rpm 
warning: nginx-1.20.2-1.el7.ngx.src.rpm: Header V4 RSA/SHA1 Signature, key ID 7bd9bf62: NOKEY
[root@rpmtest ~]# ll
total 17832
-rw-r--r--.  1 root root 11924330 Apr 14 16:43 OpenSSL_1_1_1-stable.zip
-rw-------.  1 root root     5207 Dec  4  2020 anaconda-ks.cfg
-rw-r--r--.  1 root root  1082461 Apr 14 16:43 nginx-1.20.2-1.el7.ngx.src.rpm
drwxr-xr-x. 19 root root     4096 Sep 11  2023 openssl-OpenSSL_1_1_1-stable
-rw-------.  1 root root     5006 Dec  4  2020 original-ks.cfg
-rw-r--r--.  1 root root  5222976 Apr 14 16:43 percona-orchestrator-3.2.6-2.el8.x86_64.rpm
drwxr-xr-x.  4 root root       34 Apr 14 16:44 rpmbuild
```
1.2. Разрешение зависимостей пакета NGINX:<br/>
```shell
[root@rpmtest ~]# yum-builddep rpmbuild/SPECS/nginx.spec
...
```
1.3. Редактирование spec-файла NGINX для того, что бы соответствующий пакет собирался с<br/> 
поддержкой SSL. Для этого необходимо в секцию ./configure файла nginx.spec, добавить опцию<br/>
сборки --with-openssl=/root/openssl-OpenSSL_1_1_1-stable, в которой содержится путь к исходным<br/>
кодам OpenSSL.<br/>

1.4. Сборка RPM пакета NGINX:
```shell
[root@rpmtest ~]# rpmbuild -bb rpmbuild/SPECS/nginx.spec
...
[root@rpmtest ~]# ll rpmbuild/RPMS/x86_64/
total 4864
-rw-r--r--. 1 root root 2444220 Apr 14 17:11 nginx-1.20.2-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 2533336 Apr 14 17:11 nginx-debuginfo-1.20.2-1.el8.ngx.x86_64.rpm
```
1.5. Установка пакета NGINX:
```shell
[root@rpmtest ~]# yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm
...
Installed:
  nginx-1:1.20.2-1.el8.ngx.x86_64                                                                                                        

Complete!
```
1.6. Запуск сервиса NGINX и проверка его статуса:
```shell
[root@rpmtest ~]# systemctl start nginx
[root@rpmtest ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-04-14 17:18:15 UTC; 14s ago
     Docs: http://nginx.org/en/docs/
  Process: 46855 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 46856 (nginx)
    Tasks: 5 (limit: 12419)
   Memory: 5.1M
   CGroup: /system.slice/nginx.service
           ├─46856 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           ├─46857 nginx: worker process
           ├─46858 nginx: worker process
           ├─46859 nginx: worker process
           └─46860 nginx: worker process

Apr 14 17:18:15 rpmtest systemd[1]: Starting nginx - high performance web server...
Apr 14 17:18:15 rpmtest systemd[1]: Started nginx - high performance web server.
```
1.7. Создание и наполнение своего репозитория:
```shell
[root@rpmtest ~]# mkdir /usr/share/nginx/html/repo
[root@rpmtest ~]# cp rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm /usr/share/nginx/html/repo
[root@rpmtest ~]# cp percona-orchestrator-3.2.6-2.el8.x86_64.rpm /usr/share/nginx/html/repo
[root@rpmtest ~]# ll /usr/share/nginx/html/repo
total 7492
-rw-r--r--. 1 root root 2444220 Apr 14 17:22 nginx-1.20.2-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 5222976 Apr 14 17:22 percona-orchestrator-3.2.6-2.el8.x86_64.rpm
```
1.8. Инициализация нового репозитория:
```shell
[root@rpmtest ~]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 2 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```
1.9. Настройка в NGINX доступа к листингу каталога репозитория. В location / файла<br/>
/etc/nginx/conf.d/default.conf необходимо добавить дерективу autoindex on. После чего<br/>
проверить корректность конфигурационного файла и перезапустить сервис NGINX.
```shell
[root@rpmtest ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@rpmtest ~]# nginx -s reload

```





### 4. Скрипт конфигурирования сервера ###
```shell
#!/bin/bash

sudo -i
# установка репозитария zfs
yum install -y http://download.zfsonlinux.org/epel/zfs-release.el7_8.noarch.rpm
# импорт gpg ключей
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux
# установка необходимого для ПО
yum install -y epel-release kernel-devel wget
# для того, чтобы при установке пакета zfs собирался соответствующий модуль ядра
# необходимо создать новую ссылку /lib/modules/`uname -r`/build на каталог сборки
rm -f /lib/modules/`uname -r`/build
ln -s /usr/src/kernels/* /lib/modules/`uname -r`/build
# Смена zfs репозитария 
yum-config-manager --disable zfs
yum-config-manager --enable zfs-kmod
# установка zfs
yum install -y zfs
# загрузка модуля ядра zfs
modprobe zfs
# добавление модуля ядра zfs в автозагрузку
sudo echo "zfs" >> /etc/modules-load.d/zfs.conf
```
