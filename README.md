# DHCP-PXE
1. Устанавливаем dnsmasq , что бы клиенты могли получать ip адреса от DHCP- сервера и  так же установим TFTP-сервер (получим файл pxelinux.0).
2. Отключаем firewall:
```
root@pxeserver:~# systemctl stop ufw
root@pxeserver:~# systemctl disable ufw
root@pxeserver:~# systemctl status ufw
```
3. Установка утилиты dnsmasq:
```
 apt update && apt install dnsmasq -y
```
4. файла /etc/dnsmasq.d/pxe.conf: выглядит так:
```
root@pxeserver:~# nano /etc/dnsmasq.d/pxe.conf
...
root@pxeserver:~# cat /etc/dnsmasq.d/pxe.conf
interface=enp0s8
bind-interfaces
dhcp-range=enp0s8, 10.0.0.100, 10.0.0.120
dhcp-boot=pxelinux.0
enable-tftp
tftp-root=/srv/tftp/amd64
```
5. Создаем директорию в нее скачиваем дистрибутив для установги его по сети:
```
root@pxeserver:~# mkdir /srv/tftp
root@pxeserver:~# wget https://mirror.yandex.ru/ubuntu-releases/24.04/ubuntu-24.04-netboot-amd64.tar.gz
root@pxeserver:~# tar -xzvf ubuntu-24.04-netboot-amd64.tar.gz -C /srv/ftp
root@pxeserver:~# ll /srv/tftp/amd64
total 85268
drwxr-xr-x 4 root root     4096 Apr 23 09:46 ./
drwxr-xr-x 3 root root     4096 Apr 23 09:46 ../
-rw-r--r-- 1 root root   966664 Apr  4 12:39 bootx64.efi
drwxr-xr-x 2 root root     4096 Apr 23 09:46 grub/
-rw-r--r-- 1 root root  2340744 Apr  4 10:24 grubx64.efi
-rw-r--r-- 1 root root 68889068 Apr 23 09:46 initrd
-rw-r--r-- 1 root root   118676 Apr  8 16:20 ldlinux.c32
-rw-r--r-- 1 root root 14928264 Apr 23 09:46 linux
-rw-r--r-- 1 root root    42392 Apr  8 16:20 pxelinux.0
drwxr-xr-x 2 root root     4096 Jul 14 15:14 pxelinux.cfg/
```
6. Устанавливаем apache2, что бы передача файлов была по протоколу HTTP:
```
apt install apache2
```
7. Скачиваем Ubuntu 24.04
```
root@pxeserver:~# mkdir /srv/images && cd /srv/images
root@pxeserver:/srv/images# wget https://mirror.yandex.ru/ubuntu-releases/24.04/ubuntu-24.04-live-server-amd64.iso   
```
8. Страница для доступа к серверу TFTP:
```
root@pxeserver:~# nano /etc/apache2/sites-available/ks-server.conf
...
root@pxeserver:~# cat /etc/apache2/sites-available/ks-server.conf
<VirtualHost 10.0.0.20:80>
DocumentRoot /
	<Directory /srv/ks>
		Options Indexes MultiViews
		AllowOverride All
		Require all granted
	</Directory>
	<Directory /srv/images>
		Options Indexes MultiViews
		AllowOverride All
		Require all granted	
	</Directory>
</VirtualHost>
```
9. Активирование конфигурации ks-server.conf в apache:
```
root@pxeserver:/etc/apache2/sites-available/# a2ensite ks-server.conf
```
10. файла конфигурации загрузчика PXE /srv/tftp/amd64/pxelinux.cfg/default:
```
root@pxeserver:~# cat /srv/tftp/amd64/pxelinux.cfg/default
DEFAULT install
LABEL install
  KERNEL linux
  INITRD initrd
  APPEND root=/dev/ram0 ramdisk_size=3000000 ip=dhcp iso-url=http://10.0.0.20/srv/images/ubuntu-24.04-live-server-amd64.iso autoinstall ds=nocloud-net;s=http://10.0.0.20/srv/ks
```
11. Правим файл, что бы Ubuntu автоматически установилась
```
root@pxeserver:~# mkdir /srv/ks
root@pxeserver:~# nano /srv/ks/user-data
...
root@pxeserver:~# cat /srv/ks/user-data
#cloud-config
autoinstall:
  apt:
    disable_components: []
    geoip: true
    preserve_sources_list: false
    primary:
      - arches:
          - amd64
          - i386
        uri: http://us.archive.ubuntu.com/ubuntu
      - arches:
          - default
        uri: http://ports.ubuntu.com/ubuntu-ports
  drivers:
    install: false
  identity:
    hostname: linux
    password: $6$sJgo6Hg5zXBwkkI8$btrEoWAb5FxKhajagWR49XM4EAOfO/Dr5bMrLOkGe3KkMYdsh7T3MU5mYwY2TIMJpVKckAwnZFs2ltUJ1abOZ.
    realname: otus
    username: otus
  kernel:
    package: linux-generic
  keyboard:
    layout: us
    toggle: null
    variant: ''
  locale: en_US.UTF-8
  network:
    version: 2
    ethernets:
      enp0s3:
        dhcp4: true
      enp0s8:
        dhcp4: true
  ssh:
    allow-pw: true
    authorized-keys: []
    install-server: true
  updates: security
  version: 1
``` 
12. Перезапуск сервисов:
```
root@pxeserver:~# systemctl restart dnsmasq
root@pxeserver:~# systemctl restart apache2
``` 
13. настройка окончена. После перезапуска pxeclient, сетевой интерфейс получит IP адрес и начнется установка ОС.
