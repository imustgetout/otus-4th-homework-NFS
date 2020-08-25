Установка  и настрока NFS сервер\клиент CentOS7

Серверная часть
устанавливаем компоненты NFS:
nfs-utils - NFS utilities and supporting clients and daemons for the kernel NFS server.
nfs-utils-lib - Network File System Support Library
yum install nfs-utils nfs-utils -y
Создаем директорию, которую будем экспортировать:
mkdir -p /var/super_shara/upload
Ключ -p создает все отсутствующие каталоги в пути. Экспортировать будем каталог /var/super_shara, но право записи дадим только в каталог /var/super_shara/upload.
Меняем владельца /var/super_shara/upload на анонимного пользователя nfsnobody:
chown nfsnobody /var/super_shara/upload.
В параметрах /etc/exportfs укажем ключ all_squash - все будут писать в каталог /var/super_shara/upload от имени пользователя nfsnobody.
Включаем и запускаем сервисы:
sudo systemctl enable rpcbind
sudo systemctl enable nfs-server
sudo  systemctl enable nfs-lock
sudo systemctl start rpcbind
sudo systemctl start nfs-server
sudo systemctl start nfs-lock

Вносим изменения в etc/exports:
echo "/var/super_shara 192.168.50.11(rw,sync,root_squash,all_squash)" >> /etc/exports
Перечитываем конфиг:
exportfs -a
Настройка firewall:
Запустим firewall:
systemctl start firewalld.service
Для NFS3 нужно открыть порты: 111(RPC)(Для NFS4 не нужно),2049(NFS), и mountd(произвольный порт, можно назначить постоянный в /etc/sysconfig/nfs)
firewall-cmd --zone=public --add-service=nfs3 --permanent
firewall-cmd --zone=public --add-service=rpc-bind --permanent
firewall-cmd --zone=public --add-service=mountd --permanent
firewall-cmd --reload



Клиентская часть

Также устанавливаем компоненты  NFS:
yum install nfs-utils nfs-utils -y
создаем каталог, в который будем монтировать:
mkdir /mnt/mega-shara
Монтируем:
mount -t nfs -o vers=3,proto=udp 192.168.50.10:/var/super_shara /mnt/mega-shara
Дабавляем запись в /etc/fstab:
echo "192.168.50.10: /var/super_shara/  /mnt/mega-shara/        nfs     rw,sync,hard,intr       0 0" >>/etc/fstab

Использование autofilesystem  вместо /etc/fstab:
Устанавливаем пакет autosf:
yum install autofs -y
Редактируем  Master map file (/etc/auto.master):
Добавляем строку:
/mnt	/etc/auto.mega	--timeout=180
Формат записи: точка куда монтируем	map файл	опции(опции не обязательны, тут timeout - количество секунд простоя после которых устройство будет отмонтировано)
Создаем map file(auto.mega) и записываем туда строку:
mega-shara	-fstype=nfs,rw,soft,intr	192.168.50.10:/var/super_shara/
Перестартуем autofs:
service autofs restart

Использование systemd automount:
yum install nfs-utils nfs-utils-lib autofs -y
mkdir /mnt/mega-shara
Перед созданием unit нужно будет преобразовать путь к точке монтирования в \x2descape -стиль.
systemd-escape \mnt\mega-shara. Это имя используем для именования юнита.
Генерим юнит:
cat << EOF | sudo tee '/etc/systemd/system/mnt-mega\x2dshara.mount'
[Unit]
Description=Mount NFS share
After=network-online.service
Wants=network-online.service

[Mount]
What=192.168.50.10:/var/super_shara
Where=/mnt/mega-shara
Options=rw,sync,hard,intr
Type=nfs
TimeoutSec=60

[Install]
WantedBy=multi-user.target
EOF

Прописываем использование 3-й версии nfs в клиенте:
echo 'Defaultvers=3'>>/etc/nfsmount.conf
Перестартуем system manager:
systemctl daemon-reload
Включаем монтирование:
systemctl enable 'mnt-mega\x2dshara.mount'
Монтируем:
systemctl start 'mnt-mega\x2dshara.mount'


используемые источники:
http://www.bog.pp.ru/work/NFS.html
https://montazhtv.ru/firewall-otkryt-port-centos-7/
https://www.linuxtechi.com/automount-nfs-share-in-linux-using-autofs/
https://linuxconfig.org/how-to-configure-the-autofs-daemon-on-centos-7-rhel-7
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-nfs#s2-nfs-how-daemons
https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04-ru
https://blog.sleeplessbeastie.eu/2019/09/23/how-to-mount-nfs-share-using-systemd/
https://wiki.it-kb.ru/unix-linux/centos/linux-how-to-setup-nfs-server-with-share-and-nfs-client-in-centos-7-2

