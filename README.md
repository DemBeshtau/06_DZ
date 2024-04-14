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

1.1. Просмотр всех имеющихся дисков виртуальной машины:
```shell
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk 
```
1.2. Подготовка пулов из двух дисков в режиме RAID 1:
```shell
[root@zfs ~]# zpool create tank1 /dev/sdb /dev/sdc
[root@zfs ~]# zpool create tank2 /dev/sdd /dev/sde
[root@zfs ~]# zpool create tank3 /dev/sdf /dev/sdg
[root@zfs ~]# zpool create tank4 /dev/sdh /dev/sdi
```
1.3. Просмотр списка созданных пулов:
```shell
[root@zfs ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank1   960M   108K   960M        -         -     0%     0%  1.00x    ONLINE  -
tank2   960M   108K   960M        -         -     0%     0%  1.00x    ONLINE  -
tank3   960M   108K   960M        -         -     0%     0%  1.00x    ONLINE  -
tank4   960M   122K   960M        -         -     0%     0%  1.00x    ONLINE  -
```
1.4. Включение разных алгоритмов сжатия в каждую из файловых систем:
```shell
[root@zfs ~]# zfs set compression=lzjb tank1
[root@zfs ~]# zfs set compression=lz4 tank2
[root@zfs ~]# zfs set compression=gzip-9 tank3
[root@zfs ~]# zfs set compression=zle tank4
```
1.5. Проверка наличия алгоритмов сжатия на файловых системах:
```shell
[root@zfs ~]# zfs get all | grep compression
tank1  compression           lzjb                   local
tank2  compression           lz4                    local
tank3  compression           gzip-9                 local
tank4  compression           zle                    local
```
1.6. Заполнение пулов однообразной информацией:
```shell
[root@zfs ~]# for i in {1..4}; do wget -P /tank$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
...
```
1.7. Проверка заполнения пулов информацией:
```shell
[root@zfs ~]# ll /tank/
/tank1:
total 22076
-rw-r--r--. 1 root root 41034307 Apr  2 07:54 pg2600.converter.log

/tank2:
total 17999
-rw-r--r--. 1 root root 41034307 Apr  2 07:54 pg2600.converter.log

/tank3:
total 10961
-rw-r--r--. 1 root root 41034307 Apr  2 07:54 pg2600.converter.log

/tank4:
total 40102
-rw-r--r--. 1 root root 41034307 Apr  2 07:54 pg2600.converter.log
```
1.8. Вывод информации о пулах с целью определения эффективности сжатия:
```shell
[root@zfs ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
tank1  23.5M   808M     23.3M  /tank1
tank2  17.8M   814M     17.6M  /tank2
tank3  10.9M   821M     10.7M  /tank3
tank4  39.3M   793M     39.2M  /tank4.
```
1.9. Вывод информации о степени сжатия файлов на пулах:
```shell
[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
tank1  compressratio         1.90x                  -
tank2  compressratio         2.22x                  -
tank3  compressratio         3.65x                  -
tank4  compressratio         1.00x                  -
```
&ensp;&ensp;По итогам работы видно, что самым эффективным алгоритмом по степени сжатия является gzip-9.<br/>
### 2. Определение настроек пула ###
2.1. Получение и разархивирование архива для работы:
```shell
[root@zfs ~]# wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
[root@zfs ~]# ll
total 7124
-rw-------. 1 root root    5570 Apr 30  2020 anaconda-ks.cfg
-rw-r--r--. 1 root root 7275140 Dec  6 15:49 archive.tar.gz
-rw-------. 1 root root    5300 Apr 30  2020 original-ks.cfg
[root@zfs ~]# tar -xzvf archive.tar.gz 
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```
2.2. Проверка возможности импортирования содержимого архива в пул:
```shell
[root@zfs ~]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
```
2.3. Осуществление импорта скачанного пула и просмотр его состояния:
```shell
[root@zfs ~]# zpool import -d zpoolexport/ otus
[root@zfs ~]# zpool status otus
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0*
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

[root@zfs ~]# zfs getavailable otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -

[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default

[root@zfs ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local

[root@zfs ~]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local

[root@zfs ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```
### 3. Работа со снапшотом, поиск сообщения от преподавателя ###
3.1. Получение файла для работы:
```shell
wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
[root@zfs ~]# ll
total 12432
-rw-------. 1 root root    5570 Apr 30  2020 anaconda-ks.cfg
-rw-r--r--. 1 root root 7275140 Dec  6 15:49 archive.tar.gz
-rw-------. 1 root root    5300 Apr 30  2020 original-ks.cfg
-rw-r--r--. 1 root root 5432736 Dec  6 15:22 otus_task2.file
drwxr-xr-x. 2 root root      32 May 15  2020 zpoolexport
````
3.2. Восстановление файловой системы из снапшота:
```shell
[root@zfs ~]# zfs receive otus/test@today < otus_task2.file 
```
3.3. Поиск заданного файла по имени и просмотр его содержимого:
```shell
[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message

[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
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
