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

Строкой "/srv/share      192.168.50.11(rw)" мы указываем, что хотим экспортировать дирректорию /srv/share и сделать её доступной для импорта клиенту 192.168.50.11 с возможностью записи (rw). \

Далее применяем наши изменения и проверяем.
```

[root@nfss ~]# exportfs -a
[root@nfss ~]# exportfs -s
/srv/share  192.168.50.11(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```



