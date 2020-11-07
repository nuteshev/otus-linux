# Основное задание

Скрипт для создания рейда включён в Vagrantfile. Конфигурационный файл для сборки рейда при запуске: 
```
[vagrant@otuslinux ~]$ cat /etc/mdadm/mdadm.conf
DEVICE partitions
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 spares=1 name=otuslinux:0 UUID=5995ec12:80b8d3a4:50ffee69:254a74b1
```

## Дополнительное задание

1. Добавляем в ОС дополнительный диск /dev/sdb
```
[vagrant@otuslinux ~]$ lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0   40G  0 disk
└─sda1      8:1    0   40G  0 part  /
sdb         8:16   0   40G  0 disk
sdc         8:32   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdd         8:48   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sde         8:64   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdf         8:80   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdg         8:96   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
```

2. Создаём на /dev/sdb раздел типа fd:
```
sfdisk -d /dev/sda | sfdisk /dev/sdb 
echo -e "t\nfd\nw" | fdisk /dev/sdb
```

3. Создаём RAID1 из одного диска /dev/sdb:
```
mdadm --create /dev/md1 --level=1 --raid-devices=2 missing /dev/sdb1 --metadata=0.90
mkfs.xfs /dev/md1
mdadm --detail --scan | grep md1 >> /etc/mdadm/mdadm.conf
```

4. Копируем содержимое первого диска на второй: 
```
mount /dev/md1 /mnt
rsync -auHxv --exclude=/proc/* --exclude=/sys/* --exclude=/mnt/* /* /mnt
```

5. Заменяем в /mnt/etc/fstab первую строчку на 
```
/dev/md1 /                       xfs     defaults        0 0
```

6. Создаём на новом диске новый initrd-образ: 
```
mount --bind /proc /mnt/proc && mount --bind /dev /mnt/dev && mount --bind /sys /mnt/sys && mount --bind /run /mnt/run && chroot /mnt/
dracut -f
exit 
```

7. Генерим новые настройки GRUB:
```
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-install /dev/sda
grub2-install /dev/sdb
```

8. Перезагружаемся, в меню GRUB появляется опция загрузки с /dev/md1, редактируем её, а именно, в строке, начинающейся c linux, добавляем в конце три опции: 
```
rd.auto rd.auto=1 selinux=0
```
и загружаемся. 

9. 
Добавляем первый диск в рейд
```
echo -e "t\nfd\nw" | fdisk /dev/sda
mdadm --add /dev/md1 /dev/sda1
``` 
и ждём окончания синхронизации. 

10. Открываем файл /etc/default/grub и в строке GRUB_CMDLINE_LINUX убираем из кавычек опцию elevator=noop, зато добавляем опции rd.auto rd.auto=1. Повторяем генерацию настроек GRUB из пункта 7.

11. Машина успешно загружается. Вывод lsblk:

```
[vagrant@otuslinux ~]$ lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0   40G  0 disk
└─sda1      8:1    0   40G  0 part
  └─md1     9:1    0   40G  0 raid1 /
sdb         8:16   0   40G  0 disk
└─sdb1      8:17   0   40G  0 part
  └─md1     9:1    0   40G  0 raid1 /
sdc         8:32   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdd         8:48   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sde         8:64   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdf         8:80   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdg         8:96   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
``` 

