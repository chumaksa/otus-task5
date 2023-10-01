# otus-task5

# Vagrant стенд для NFS

* vagrant up должен поднимать 2 виртуалки: сервер и клиент;
* на сервере должна быть расшарена директория;
* на клиента она должна автоматически монтироваться при старте (fstab или autofs);
* в шаре должна быть папка upload с правами на запись;
* требования для NFS: NFSv3 по UDP, включенный firewall.

## Vagrant up должен поднимать 2 виртуалки: сервер и клиент;

### Решение

Создаём файл Vagrant следующего содержания:
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  config.vm.box_version = "2004.01"

  config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 1
  end

  config.vm.define "nfss" do |nfss|
    nfss.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "net1"
    nfss.vm.hostname = "nfss"
  end

  config.vm.define "nfsc" do |nfsc|
    nfsc.vm.network "private_network", ip: "192.168.50.11", virtualbox__intnet: "net1"
    nfsc.vm.hostname = "nfsc"
  end

end
```

После выполнения команды vagrant up у нас получится две машины;
```

Current machine states:

nfss                      running (virtualbox)
nfsc                      running (virtualbox)
```

Виртуальная машина с именем nfss будет сервером, а nfsc - клиентом.

## На сервере должна быть расшарена директория

### Решение

В нашем дистрибутиве уже установлен nfs сервер, нам остаётся только запустить его.
```

[root@nfss ~]# systemctl enable nfs --now
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.

[root@nfss ~]# systemctl status nfs-server.service
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
   Active: active (exited) since Sat 2023-09-30 18:40:42 UTC; 32s ago
  Process: 3287 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
  Process: 3271 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 3270 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 3271 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service

Sep 30 18:40:42 nfss systemd[1]: Starting NFS server and services...
Sep 30 18:40:42 nfss systemd[1]: Started NFS server and services.
```

Далее создаём директорию которую будем расшаривать - /srv/share/. По заданию нам нужно будет отдельно изменить права на вложенную папку upload, создадим её сразу.
```

[root@nfss ~]# mkdir -p /srv/share/upload/
```

Изменим владельца папки чтобы к ней можно было получить доступ.
```

[root@nfss ~]# chown -R nfsnobody:nfsnobody /srv/share
```

Для того чтобы экспортировать нашу категорию нам нужно отредактирвать файл /etc/exports
```

[root@nfss ~]# vi /etc/exports
[root@nfss ~]# cat /etc/exports
/srv/share      192.168.50.11(rw)
```

Строкой "/srv/share      192.168.50.11(rw)" мы указываем, что хотим экспортировать дирректорию /srv/share и сделать её доступной для импорта клиенту 192.168.50.11 с возможностью записи (rw).

Далее применяем наши изменения и проверяем.
```

[root@nfss ~]# exportfs -a
[root@nfss ~]# exportfs -s
/srv/share  192.168.50.11(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```

## На клиента она должна автоматически монтироваться при старте (fstab или autofs).

### Решение

Заходи на машину клиент по ssh, создаём дирректорию для точки монтирования и монтируем с сервера расшаренную папку.
```

[root@nfsc ~]# mkdir -p /mnt/share/
[root@nfsc ~]# mount -o rw,hard,intr,bg 192.168.50.10:/srv/share/ /mnt/share/
```

Опция rw - монитрование фаловой системы для чтения-записи; \
hard - если сервер отключился, операции, которые пытаются получить к нему доступ, блокируются до тех пор, пока работоспособность сервера не восстановится; \
intr - позволяет прерывать заблокированные операции с клавиатуры; \
bg - если сервер не отвечает, переводим операцию монтирования в фоновый режим и продолжаем другие операции монтирования.

Проверим, что всё смонтировалось и с какими опциями:
```

[root@nfsc ~]# mount | grep /mnt/share
192.168.50.10:/srv/share on /mnt/share type nfs4 (rw,relatime,vers=4.1,rsize=32768,wsize=32768,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.50.11,local_lock=none,addr=192.168.50.10)
```

Для автоматического монтирования будем использовать службу autofs. Установим необходимый пакет.
```

[root@nfsc ~]# yum install -y autofs
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.yandex.ru
 * extras: mirror.yandex.ru
 * updates: mirror.corbina.net
base                                                                                                                                                                                                                                                    | 3.6 kB  00:00:00
extras                                                                                                                                                                                                                                                  | 2.9 kB  00:00:00
updates                                                                                                                                                                                                                                                 | 2.9 kB  00:00:00
(1/4): base/7/x86_64/group_gz                                                                                                                                                                                                                           | 153 kB  00:00:00
(2/4): extras/7/x86_64/primary_db                                                                                                                                                                                                                       | 250 kB  00:00:01
(3/4): base/7/x86_64/primary_db                                                                                                                                                                                                                         | 6.1 MB  00:00:06
(4/4): updates/7/x86_64/primary_db                                                                                                                                                                                                                      |  23 MB  00:00:08
Resolving Dependencies
--> Running transaction check
---> Package autofs.x86_64 1:5.0.7-116.el7_9.1 will be installed
--> Processing Dependency: libhesiod.so.0()(64bit) for package: 1:autofs-5.0.7-116.el7_9.1.x86_64
--> Running transaction check
---> Package hesiod.x86_64 0:3.2.1-3.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================== Package                 Arch                    Version                                 Repository                Size
========================================================================================================================Installing:
 autofs                  x86_64                  1:5.0.7-116.el7_9.1                     updates                  834 k
Installing for dependencies:
 hesiod                  x86_64                  3.2.1-3.el7                             base                      30 k

Transaction Summary
========================================================================================================================Install  1 Package (+1 Dependent package)

Total download size: 864 k
Installed size: 5.2 M
Downloading packages:
warning: /var/cache/yum/x86_64/7/base/packages/hesiod-3.2.1-3.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for hesiod-3.2.1-3.el7.x86_64.rpm is not installed
(1/2): hesiod-3.2.1-3.el7.x86_64.rpm                                                             |  30 kB  00:00:00
Public key for autofs-5.0.7-116.el7_9.1.x86_64.rpm is not installed===========        ]  0.0 B/s | 672 kB  --:--:-- ETA
(2/2): autofs-5.0.7-116.el7_9.1.x86_64.rpm                                                       | 834 kB  00:00:00
------------------------------------------------------------------------------------------------------------------------Total                                                                                   918 kB/s | 864 kB  00:00:00
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-8.2003.0.el7.centos.x86_64 (@anaconda)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : hesiod-3.2.1-3.el7.x86_64                                                                            1/2
  Installing : 1:autofs-5.0.7-116.el7_9.1.x86_64                                                                    2/2
  Verifying  : hesiod-3.2.1-3.el7.x86_64                                                                            1/2
  Verifying  : 1:autofs-5.0.7-116.el7_9.1.x86_64                                                                    2/2

Installed:
  autofs.x86_64 1:5.0.7-116.el7_9.1

Dependency Installed:
  hesiod.x86_64 0:3.2.1-3.el7

Complete!
```

Для работы autofs нам нужно создать таблицу прямых ссылок и заполнить её
```

[root@nfsc ~]# vi /etc/auto.direct
[vagrant@nfsc ~]$ cat /etc/auto.direct
/mnt/share/     -rw,hard,intr,bg 192.168.50.10:/srv/share/
```

Далее запусаем демон autofs
```

[root@nfsc ~]# systemctl enable --now autofs
Created symlink from /etc/systemd/system/multi-user.target.wants/autofs.service to /usr/lib/systemd/system/autofs.service.
[root@nfsc ~]# systemctl status autofs
● autofs.service - Automounts filesystems on demand
   Loaded: loaded (/usr/lib/systemd/system/autofs.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2023-10-01 09:41:14 UTC; 7s ago
 Main PID: 1250 (automount)
   CGroup: /system.slice/autofs.service
           └─1250 /usr/sbin/automount --systemd-service --dont-check-daemon

Oct 01 09:41:14 nfsc systemd[1]: Starting Automounts filesystems on demand...
Oct 01 09:41:14 nfsc systemd[1]: Started Automounts filesystems on demand.
```

Теперь наша папка монтируется автоматически, даже после перезагрузки
```

[root@nfsc ~]# mount | grep /mnt/share
/etc/auto.direct on /mnt/share type autofs (rw,relatime,fd=17,pgrp=1250,timeout=300,minproto=5,maxproto=5,direct,pipe_ino=18399)
```

## В шаре должна быть папка upload с правами на запись.

### Решение

Раннее мы уже добавляли папку upload на сервере. Сейчас изменим ей права.
```

[root@nfss ~]# chmod 0777 /srv/share/upload
```

## Требования для NFS: NFSv3 по UDP, включенный firewall.

### Решение
В нашем случае использует 4-ая версия NFS и протокол tcp. Изменим это в соответствии с требованиями задачи, путём редактирования файла /etc/auto.direct
```

[root@nfsc ~]# vi /etc/auto.direct
[root@nfsc ~]# cat /etc/auto.direct
/mnt/share/     -rw,hard,intr,bg,nfsvers=3,proto=udp    192.168.50.10:/srv/share/
[root@nfsc ~]# mount | grep /mnt/share
/etc/auto.direct on /mnt/share type autofs (rw,relatime,fd=5,pgrp=1342,timeout=300,minproto=5,maxproto=5,direct,pipe_ino=19434)
192.168.50.10:/srv/share/ on /mnt/share type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)
```

Видим, что используется vers=3 и proto=udp. Изменения внесены успешно. Осталось включить и настроить firewall.

Проверяем на сервере запущен ли firewall
```

[root@nfss ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

Видим, что firewall сейчас выключен. Включем его и проверим статус ещё раз
```

[root@nfss ~]# systemctl enable firewalld.service --now
[root@nfss ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2023-09-30 23:08:04 UTC; 6s ago
     Docs: man:firewalld(1)
 Main PID: 22763 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─22763 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Sep 30 23:08:04 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
Sep 30 23:08:04 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
Sep 30 23:08:04 nfss firewalld[22763]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. Please consider disabling it now.
```

Разрешаем доступ к сервисам NFS
```

[root@nfss ~]# firewall-cmd --add-service="nfs3" --add-service="rpc-bind" --add-service="mountd" --permanent
success
[root@nfss ~]# firewall-cmd --reload
success
```

Теперь проверяем firewall на клиенте.
```

[root@nfsc ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```

Firewall выключен. Запустим его.
```

[root@nfsc ~]# systemctl enable firewalld.service --now
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
[root@nfsc ~]# systemctl status firewalld.service
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2023-10-01 11:37:30 UTC; 4s ago
     Docs: man:firewalld(1)
 Main PID: 1476 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─1476 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid

Oct 01 11:37:30 nfsc systemd[1]: Starting firewalld - dynamic firewall daemon...
Oct 01 11:37:30 nfsc systemd[1]: Started firewalld - dynamic firewall daemon.
Oct 01 11:37:30 nfsc firewalld[1476]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. Please consider disabling it now.
```