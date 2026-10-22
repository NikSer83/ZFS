# Домашнее задание на тему "Работа с ФС ZFS"

1. Определить алгоритм с наилучшим сжатием:
- определить, какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
- создать 4 файловых системы, на каждой применить свой алгоритм сжатия;
- для сжатия использовать либо текстовый файл, либо группу файлов.
2. Определить настройки пула.  
    С помощью команды zfs import собрать pool ZFS.  
    Командами zfs определить настройки:
- размер хранилища;
- тип pool;
- значение recordsize;
- какое сжатие используется;
- какая контрольная сумма используется.
3. Работа со снапшотами:
- скопировать файл из удаленной директории;
- восстановить файл локально. zfs receive;
- найти зашифрованное сообщение в файле secret_message.

Для выполнения работы буду использовать новую ВМ Ubuntu server 24.04 с добавлением 8 дисков по 512Mb

### 1. Определить алгоритм с наилучшим сжатием

После запуска сразу "заsudoшусь" и проверю наличие дисков: 
```
root@linuxzfs:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0 11,5G  0 lvm  /
sdb                         8:16   0  512M  0 disk 
sdc                         8:32   0  512M  0 disk 
sdd                         8:48   0  512M  0 disk 
sde                         8:64   0  512M  0 disk 
sdf                         8:80   0  512M  0 disk 
sdg                         8:96   0  512M  0 disk 
sdh                         8:112  0  512M  0 disk 
sdi                         8:128  0  512M  0 disk 
```
Устанавливаю пакет утилит для **ZFS**:
```
root@linuxzfs:~# apt install zfsutils-linux
```
Создаю 4 пула по 2 диска в режиме RAID1:
```
root@linuxzfs:~# zpool create zfs_test_1 mirror /dev/sd{b,c}
root@linuxzfs:~# zpool create zfs_test_2 mirror /dev/sd{d,e}
root@linuxzfs:~# zpool create zfs_test_3 mirror /dev/sd{f,g}
root@linuxzfs:~# zpool create zfs_test_4 mirror /dev/sd{h,i}
```
Выведу информацию о пулах:
```
root@linuxzfs:~# zpool status
  pool: zfs_test_1
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	zfs_test_1  ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: zfs_test_2
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	zfs_test_2  ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: zfs_test_3
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	zfs_test_3  ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: zfs_test_4
 state: ONLINE
config:

	NAME        STATE     READ WRITE CKSUM
	zfs_test_4  ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors
root@linuxzfs:~# zpool list
NAME         SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zfs_test_1   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
zfs_test_2   480M   117K   480M        -         -     0%     0%  1.00x    ONLINE  -
zfs_test_3   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
zfs_test_4   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
Добавлю для каждого пула свой алгоритм сжатия:
```
root@linuxzfs:~# zfs set compression=lzjb zfs_test_1
root@linuxzfs:~# zfs set compression=lz4 zfs_test_2
root@linuxzfs:~# zfs set compression=gzip-9 zfs_test_3
root@linuxzfs:~# zfs set compression=zle zfs_test_4
```
Проверю, что алгоритмы приминились:
```
root@linuxzfs:~# zfs get all | grep compression
zfs_test_1  compression           lzjb                       local
zfs_test_2  compression           lz4                        local
zfs_test_3  compression           gzip-9                     local
zfs_test_4  compression           zle                        local
```
Для проверки сжатия, скачаю одинаковый текстовый файл во вск пулы:
```
root@linuxzfs:~# for i in {1..4}; do wget -P /zfs_test_$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
--2026-10-19 19:19:30--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://www.gutenberg.org/cache/epub/2600/pg2600.converter.log [following]
--2026-10-19 19:19:31--  https://www.gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving www.gutenberg.org (www.gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to www.gutenberg.org (www.gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41273476 (39M) [text/plain]
Saving to: ‘/zfs_test_1/pg2600.converter.log’
```
Проверю наличие скачанного файла в пулах:
```
root@linuxzfs:~# ls -l /zfs_test_*
/zfs_test_1:
total 22134
-rw-r--r-- 1 root root 41273476 окт  2 07:31 pg2600.converter.log

/zfs_test_2:
total 18025
-rw-r--r-- 1 root root 41273476 окт  2 07:31 pg2600.converter.log

/zfs_test_3:
total 10975
-rw-r--r-- 1 root root 41273476 окт  2 07:31 pg2600.converter.log

/zfs_test_4:
total 40348
-rw-r--r-- 1 root root 41273476 окт  2 07:31 pg2600.converter.log
```
Проверю занимаемое место файла в разных пулах:
```
root@linuxzfs:~# zfs list
NAME         USED  AVAIL  REFER  MOUNTPOINT
zfs_test_1  21.8M   330M  21.6M  /zfs_test_1
zfs_test_2  17.8M   334M  17.6M  /zfs_test_2
zfs_test_3  10.9M   341M  10.7M  /zfs_test_3
zfs_test_4  39.6M   312M  39.4M  /zfs_test_4
```
Посмотрю степень сжатия:
```
root@linuxzfs:~# zfs get all | grep compressratio | grep -v ref
zfs_test_1       compressratio         1.82x                         -
zfs_test_2       compressratio         2.23x                         -
zfs_test_3       compressratio         3.66x                         -
zfs_test_4       compressratio         1.00x                         -
```
Таким образом, у нас получается, что алгоритм **gzip-9** самый эффективный по сжатию.
### 2. Определение настроек пула

Скачиваю архив в домашнюю директорию:
```
root@linuxzfs:~# wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
--2026-10-21 17:00:20--  https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 142.251.142.225, 2a00:1450:4010:c1e::84
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|142.251.142.225|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6,9M) [application/octet-stream]
Saving to: ‘archive.tar.gz’

archive.tar.gz                                                                  100%[====================================================================================================================================================================================================>]   6,94M  7,12MB/s    in 1,0s    

2026-10-21 17:00:29 (7,12 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]
```
Разархивирую его:
```
root@linuxzfs:~# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb

root@linuxzfs:~# ll
total 7140
drwx------  4 root root    4096 окт 21 17:01 ./
drwxr-xr-x 28 root root    4096 окт 21 17:09 ../
-rw-r--r--  1 root root 7275140 дек  6  2023 archive.tar.gz
-rw-------  1 root root     696 окт 19 19:36 .bash_history
-rw-r--r--  1 root root    3106 апр 22  2024 .bashrc
-rw-r--r--  1 root root     161 апр 22  2024 .profile
drwx------  2 root root    4096 окт 18 19:27 .ssh/
-rw-r--r--  1 root root     171 окт 19 19:27 .wget-hsts
drwxr-xr-x  2 root root    4096 мая 15  2020 zpoolexport/
```
Проверю возможность импортировать этот каталог а пул:
```
root@linuxzfs:~# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
	(Note that they may be intentionally disabled if the
	'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
	some features will not be available without an explicit 'zpool upgrade'.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE
```
Делаю импорт пула в ОС:
```
root@linuxzfs:~# zpool import -d zpoolexport/ otus

root@linuxzfs:~# zpool status
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
	The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
	the pool may no longer be accessible by software that does not support
	the features. See zpool-features(7) for details.
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
...
```
Просмотрю все параметры файловой системы:
```
root@linuxzfs:~# zfs get all otus
NAME  PROPERTY              VALUE                      SOURCE
otus  type                  filesystem                 -
otus  creation              Пт мая 15  4:00 2020  -
otus  used                  2.04M                      -
otus  available             350M                       -
otus  referenced            24K                        -
otus  compressratio         1.00x                      -
otus  mounted               yes                        -
otus  quota                 none                       default
otus  reservation           none                       default
otus  recordsize            128K                       local
otus  mountpoint            /otus                      default
otus  sharenfs              off                        default
otus  checksum              sha256                     local
otus  compression           zle                        local
otus  atime                 on                         default
otus  devices               on                         default
otus  exec                  on                         default
otus  setuid                on                         default
otus  readonly              off                        default
otus  zoned                 off                        default
otus  snapdir               hidden                     default
otus  aclmode               discard                    default
otus  aclinherit            restricted                 default
otus  createtxg             1                          -
otus  canmount              on                         default
otus  xattr                 on                         default
otus  copies                1                          default
otus  version               5                          -
otus  utf8only              off                        -
otus  normalization         none                       -
otus  casesensitivity       sensitive                  -
otus  vscan                 off                        default
otus  nbmand                off                        default
otus  sharesmb              off                        default
otus  refquota              none                       default
otus  refreservation        none                       default
otus  guid                  14592242904030363272       -
otus  primarycache          all                        default
otus  secondarycache        all                        default
otus  usedbysnapshots       0B                         -
otus  usedbydataset         24K                        -
otus  usedbychildren        2.01M                      -
otus  usedbyrefreservation  0B                         -
otus  logbias               latency                    default
otus  objsetid              54                         -
otus  dedup                 off                        default
otus  mlslabel              none                       default
otus  sync                  standard                   default
otus  dnodesize             legacy                     default
otus  refcompressratio      1.00x                      -
otus  written               24K                        -
otus  logicalused           1020K                      -
otus  logicalreferenced     12K                        -
otus  volmode               default                    default
otus  filesystem_limit      none                       default
otus  snapshot_limit        none                       default
otus  filesystem_count      none                       default
otus  snapshot_count        none                       default
otus  snapdev               hidden                     default
otus  acltype               off                        default
otus  context               none                       default
otus  fscontext             none                       default
otus  defcontext            none                       default
otus  rootcontext           none                       default
otus  relatime              on                         default
otus  redundant_metadata    all                        default
otus  overlay               on                         default
otus  encryption            off                        default
otus  keylocation           none                       default
otus  keyformat             none                       default
otus  pbkdf2iters           0                          default
otus  special_small_blocks  0                          default
```
Также можно просматривать конкретный параметр:
```
root@linuxzfs:~# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -

root@linuxzfs:~# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default

root@linuxzfs:~# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local

root@linuxzfs:~# zfs get compression otus
NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local

root@linuxzfs:~# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```

### 3. Работа со снапшотами и поиск сообщения

Скачиваю требуемый файл:
```
root@linuxzfs:~# wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
```
Проверю его наличие:
```
root@linuxzfs:~# ll
total 12452
drwx------  4 root root    4096 окт 21 17:30 ./
drwxr-xr-x 28 root root    4096 окт 21 17:09 ../
-rw-r--r--  1 root root 7275140 дек  6  2023 archive.tar.gz
-rw-------  1 root root     696 окт 19 19:36 .bash_history
-rw-r--r--  1 root root    3106 апр 22  2024 .bashrc
-rw-r--r--  1 root root 5432736 дек  6  2023 otus_task2.file
-rw-r--r--  1 root root     161 апр 22  2024 .profile
drwx------  2 root root    4096 окт 18 19:27 .ssh/
-rw-r--r--  1 root root     171 окт 19 19:27 .wget-hsts
-rw-r--r--  1 root root    2446 окт 21 17:30 wget-log
drwxr-xr-x  2 root root    4096 мая 15  2020 zpoolexport/
```
Восстанавливаю ФС из снапшота:
```
root@linuxzfs:~# zfs receive otus/test@today < otus_task2.file
```
Нахожу файл **secret_message** и смотрю его содержимое:
```
root@linuxzfs:~# find /otus/test/ -name "secret_message"
/otus/test/task1/file_mess/secret_message
root@linuxzfs:~# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
```
Зашифрованное сообщение - https://otus.ru/lessons/linux-hl/

Ссылка на вступительное тестирование "Инфраструктура высоконагруженных систем"
