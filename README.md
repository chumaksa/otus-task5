# otus-task5

# Vagrant стенд для NFS

* vagrant up должен поднимать 2 виртуалки: сервер и клиент;
* на сервер должна быть расшарена директория;
* на клиента она должна автоматически монтироваться при старте (fstab или autofs);
* в шаре должна быть папка upload с правами на запись;
* требования для NFS: NFSv3 по UDP, включенный firewall.

## vagrant up должен поднимать 2 виртуалки: сервер и клиент;

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
