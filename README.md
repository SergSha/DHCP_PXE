<h3>### DHCP, PXE ###</h3>

<h4>Описание домашнего задания</h4>

<ol>
<li>Следуя шагам из документа https://docs.centos.org/en-US/8-docs/advanced-install/assembly_preparing-for-a-network-install установить и настроить загрузку по сети для дистрибутива CentOS8.<br />
В качестве шаблона воспользуйтесь репозиторием https://github.com/nixuser/virtlab/tree/main/centos_pxe.</li>
<li>Поменять установку из репозитория NFS на установку из репозитория HTTP.</li>
<li>Настройить автоматическую установку для созданного kickstart файла (*) Файл загружается по HTTP.</li>
<li>* автоматизировать процесс установки Cobbler cледуя шагам из документа https://cobbler.github.io/quickstart/.<br />
Формат сдачи ДЗ - vagrant + ansible</li>
</ol>

<h4>1. Работа с шаблоном из задания</h4>

<p>В домашней директории создадим директорию dhcp_pxe, в котором будут храниться настройки виртуальных машин:</p>

<pre>[user@localhost otus]$ mkdir ./dhcp_pxe
[user@localhost otus]$</pre>

<p>Перейдём в директорию backup:</p>

<pre>[user@localhost otus]$ cd ./dhcp_pxe/
[user@localhost dhcp_pxe]$</pre>

<p>Скачиваем файлы, указанные в домашнем задании: https://github.com/nixuser/virtlab/tree/main/centos_pxe.<br />
Рассмотрим загруженный Vagrantfile:</p>

<pre>[user@localhost dhcp_pxe]$ vi ./Vagrantfile</pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :
# export VAGRANT_EXPERIMENTAL="disks"

Vagrant.configure("2") do |config|

config.vm.define "pxeserver" do |server|
  server.vm.box = 'centos/8.4'
  server.vm.disk :disk, size: "15GB", name: "extra_storage1"

  server.vm.host_name = 'pxeserver'
  server.vm.network :private_network, 
                     ip: "10.0.0.20", 
                     virtualbox__intnet: 'pxenet'

  # server.vm.network "forwarded_port", guest: 80, host: 8081

  server.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  # ENABLE to setup PXE
  server.vm.provision "shell",
    name: "Setup PXE server",
    path: "setup_pxe.sh"
  end


# config used from this
# https://github.com/eoli3n/vagrant-pxe/blob/master/client/Vagrantfile
  config.vm.define "pxeclient" do |pxeclient|
    pxeclient.vm.box = 'centos/8.4'
    pxeclient.vm.host_name = 'pxeclient'
    pxeclient.vm.network :private_network, ip: "10.0.0.21"
    pxeclient.vm.provider :virtualbox do |vb|
      vb.memory = "2048"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize [
          'modifyvm', :id,
          '--nic1', 'intnet',
          '--intnet1', 'pxenet',
          '--nic2', 'nat',
          '--boot1', 'net',
          '--boot2', 'none',
          '--boot3', 'none',
          '--boot4', 'none'
        ]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end
      # ENABLE to fix memory issues
#     end
  end

end</pre>

<p>В некоторые строки требуется внести изменения:
● Pxeclient.vm.box = 'centos/8.4' и server.vm.box = 'centos/8.4' — на данный момент в Vagrant Box нет образа с таким именем. Нам требуется образ CentOS 8.4, воспользуемся образом bento/centos-8.4. Плюсом этого Vagrant Box является то, что по умолчанию он создаёт ОС с размером диска 60ГБ. При использовании данного образа нам не придётся полдключать дополнительный диск.<br />
● # export VAGRANT_EXPERIMENTAL="disks" и server.vm.disk :disk, size: "15GB", name: "extra_storage1" — так как нам хватает свободного места, мы можем не подключать дополнитеный диск. Если вы планируете в своём домашнем задании подключить дополнительный диск, то команда export VAGRANT_EXPERIMENTAL="disks" должна быть введена в терминале.<br />
● # server.vm.network "forwarded_port", guest: 80, host: 8081 — опция проброса порта. В нашем ДЗ мы её расскомментируем. Порт 8081 оставим таким же.<br />
● # ENABLE to setup PXE — блок настройки PXE-сервера с помощью bash-скрипта. Так как мы будем использовать Ansible для настройки хоста, данный блок нам не понадобится, поэтому мы его закомментируем. Далее можно будет добавить блок настройки хоста с помощью Ansible…<br />
● Для настройки хоста через Ansible, нам потребуется добавить дополнтельный сетевой интефейс для Pxeserver. Добавим дополнительный сетевой интефейс с адресом 192.168.50.10:</p>

<pre>server.vm.network :private_network, ip: "192.168.50.10", adapter: 3.</pre>

<p>Vagrantfile теперь выглядит следующим обазом:</p>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :
# export VAGRANT_EXPERIMENTAL="disks"

Vagrant.configure("2") do |config|

config.vm.define "pxeserver" do |server|
  server.vm.box = 'bento/centos-8.4'
  server.vm.disk :disk, size: "15GB", name: "extra_storage1"

  server.vm.host_name = 'pxeserver'
  server.vm.network :private_network, 
                     ip: "10.0.0.20", 
                     virtualbox__intnet: 'pxenet'
  server.vm.network :private_network, ip: "192.168.50.10", adapter: 3

  # server.vm.network "forwarded_port", guest: 80, host: 8081

  server.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  # ENABLE to setup PXE
#  server.vm.provision "shell",
#    name: "Setup PXE server",
#    path: "setup_pxe.sh"
#  end


# config used from this
# https://github.com/eoli3n/vagrant-pxe/blob/master/client/Vagrantfile
  config.vm.define "pxeclient" do |pxeclient|
    pxeclient.vm.box = 'bento/centos-8.4'
    pxeclient.vm.host_name = 'pxeclient'
    pxeclient.vm.network :private_network, ip: "10.0.0.21"
    pxeclient.vm.provider :virtualbox do |vb|
      vb.memory = "2048"
      vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
      vb.customize [
          'modifyvm', :id,
          '--nic1', 'intnet',
          '--intnet1', 'pxenet',
          '--nic2', 'nat',
          '--boot1', 'net',
          '--boot2', 'none',
          '--boot3', 'none',
          '--boot4', 'none'
        ]
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    end
      # ENABLE to fix memory issues
#     end
  end

end</pre>

<p>После внесения всех изменений запускаем наш стенд с помощью команды vagrant up:</p>

<pre>[user@localhost dhcp_pxe]$ vagrant up</pre>

<p>Данный Vagrantfile развернет нам 2 хоста: pxeserver и pxeclient:</p>

<pre>[user@localhost dhcp_pxe]$ vagrant status
Current machine states:

pxeserver                 running (virtualbox)
pxeclient                 running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost netarchitecture]$</pre>

<p>Выполнение команды закончится с ошибкой, так как на Pxeclient настроена загрузка по сети.</p>

<p>Теперь мы приступаем к настройке Pxe-сервера.<br />
Для настроки хоста с помощью Ansible создадим структуру директорий и файлов в отдельной папке ansible.</p>

<pre>[user@localhost dhcp_pxe]$ mkdir ./ansible && cd ./ansible
[user@localhost dhcp_pxe]$ mkdir ./roles && cd ./roles
[user@localhost dhcp_pxe]$ ansible-galaxy init dhcp_pxe
</pre>

