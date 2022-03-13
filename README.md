# Практические навыки работы с ZFS
## Подготовка виртуальной машины
Загрузил локально заранее подгтовленный box для Vagrant (из урока LVM), разметил диски. Запустился и получил следующее:
```
[vagrant@zfs ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
```
LVM разделы остались с предыдущего урока, пришлось убить машину, удалить бокс и залить новый, снова запуститься:
```
sudo vagrant box remove centos7
sudo vagrant box add centos7 CentOS-7-x86_64-Vagrant-2004_01.VirtualBox.box
sudo vagrant up
sudo vagrant ssh
```
Теперь как планировалось:
```
[vagrant@zfs ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk
sdc      8:32   0  512M  0 disk
sdd      8:48   0  512M  0 disk
sde      8:64   0  512M  0 disk
sdf      8:80   0  512M  0 disk
sdg      8:96   0  512M  0 disk
sdh      8:112  0  512M  0 disk
sdi      8:128  0  512M  0 disk
```
### Подготовка zfs, добавление скрипта для первоначальной установки
```
yum install -y http://download.zfsonlinux.org/epel/zfs-release.el7_8.noarch.rpm
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux
yum install -y epel-release kernel-devel zfs
yum-config-manager --disable zfs
yum-config-manager --enable zfs-kmod
yum install -y zfs
modprobe zfs
yum install -y wget
```
Настройка успешно завершена. Данный скрипт заворачиваем в файл sh и прописываем в Vagrantfile через параметр
```
в секции VM name:
:provision => "installzfs.sh",

В секции VM resources config:
box.vm.provision "shell", path: boxconfig[:provision]
```
## 1. Определение алгоритма с наилучшим сжатием
Создаем 4 пула каждый по два диска в режиме RAID 1, и смотри информацию о них (zpool status показывает информацию о каждом диске):
```
[root@zfs ~]# zpool create otus1 mirror /dev/sdb /dev/sdc
[root@zfs ~]# zpool create otus2 mirror /dev/sdd /dev/sde
[root@zfs ~]# zpool create otus3 mirror /dev/sdf /dev/sdg
[root@zfs ~]# zpool create otus4 mirror /dev/sdh /dev/sdi
[root@zfs ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
Добавим разные алгоритмы сжатия в каждую файловую систему и проверяем:
```
[root@zfs ~]# zfs set compression=lzjb otus1
[root@zfs ~]# zfs set compression=lz4 otus2
[root@zfs ~]# zfs set compression=gzip-9 otus3
[root@zfs ~]# zfs set compression=zle otus4
[root@zfs ~]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
Скачаем один и тот же текстовый файл во все пулы и проверим:
```
[root@zfs ~]# for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
[root@zfs ~]# ls -l /otus*
/otus1:
total 22016
-rw-r--r--. 1 root root 40792737 Mar  2 09:00 pg2600.converter.log

/otus2:
total 17970
-rw-r--r--. 1 root root 40792737 Mar  2 09:00 pg2600.converter.log

/otus3:
total 10948
-rw-r--r--. 1 root root 40792737 Mar  2 09:00 pg2600.converter.log

/otus4:
total 39865
-rw-r--r--. 1 root root 40792737 Mar  2 09:00 pg2600.converter.log
```
Проверим более детально информацию о сжатии:
```
[root@zfs ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.6M   330M     21.5M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.0M   313M     39.0M  /otus4
[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                  -
otus2  compressratio         2.22x                  -
otus3  compressratio         3.64x                  -
otus4  compressratio         1.00x                  -
```
По пулу otus3 очевидно (по скорости и размеру файла после сжатия) что gzip-9 самый эффективный алгоритм по сжатию

## 2. Определение настроек пула
Скачиваем архив, разархивируем его и проверим, возможно ли импортировать данный каталог в пул (zpool status - информация о составе импортированного
пула)
:
```
[root@zfs ~]# wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
...
2022-03-13 20:44:29 (7.92 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]
[root@zfs ~]# zpool import -d zpoolexport/^C
[root@zfs ~]# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
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
Меняем имя пула во время импорта и смотрим все параметры пула (zfs get all otus если хотим получить параметры FS):
```
[root@zfs ~]# zpool import -d zpoolexport/ otus newotus
[root@zfs ~]# zfs get all newotus
NAME     PROPERTY              VALUE                  SOURCE
newotus  type                  filesystem             -
newotus  creation              Fri May 15  4:00 2020  -
newotus  used                  2.04M                  -
newotus  available             350M                   -
newotus  referenced            24K                    -
newotus  compressratio         1.00x                  -
newotus  mounted               yes                    -
newotus  quota                 none                   default
newotus  reservation           none                   default
newotus  recordsize            128K                   local
newotus  mountpoint            /newotus               default
newotus  sharenfs              off                    default
newotus  checksum              sha256                 local
newotus  compression           zle                    local
newotus  atime                 on                     default
newotus  devices               on                     default
newotus  exec                  on                     default
newotus  setuid                on                     default
newotus  readonly              off                    default
newotus  zoned                 off                    default
newotus  snapdir               hidden                 default
newotus  aclinherit            restricted             default
newotus  createtxg             1                      -
newotus  canmount              on                     default
newotus  xattr                 on                     default
newotus  copies                1                      default
newotus  version               5                      -
newotus  utf8only              off                    -
newotus  normalization         none                   -
newotus  casesensitivity       sensitive              -
newotus  vscan                 off                    default
newotus  nbmand                off                    default
newotus  sharesmb              off                    default
newotus  refquota              none                   default
newotus  refreservation        none                   default
newotus  guid                  14592242904030363272   -
newotus  primarycache          all                    default
newotus  secondarycache        all                    default
newotus  usedbysnapshots       0B                     -
newotus  usedbydataset         24K                    -
newotus  usedbychildren        2.01M                  -
newotus  usedbyrefreservation  0B                     -
newotus  logbias               latency                default
newotus  objsetid              54                     -
newotus  dedup                 off                    default
newotus  mlslabel              none                   default
newotus  sync                  standard               default
newotus  dnodesize             legacy                 default
newotus  refcompressratio      1.00x                  -
newotus  written               24K                    -
newotus  logicalused           1020K                  -
newotus  logicalreferenced     12K                    -
newotus  volmode               default                default
newotus  filesystem_limit      none                   default
newotus  snapshot_limit        none                   default
newotus  filesystem_count      none                   default
newotus  snapshot_count        none                   default
newotus  snapdev               hidden                 default
newotus  acltype               off                    default
newotus  context               none                   default
newotus  fscontext             none                   default
newotus  defcontext            none                   default
newotus  rootcontext           none                   default
newotus  relatime              off                    default
newotus  redundant_metadata    all                    default
newotus  overlay               off                    default
newotus  encryption            off                    default
newotus  keylocation           none                   default
newotus  keyformat             none                   default
newotus  pbkdf2iters           0                      default
newotus  special_small_blocks  0                      default
```
Просмотр параметров.
Размер:
```
[root@zfs ~]# zfs get available newotus
NAME     PROPERTY   VALUE  SOURCE
newotus  available  350M   -
```
Параметры чтения/записи:
```
[root@zfs ~]# zfs get readonly newotus
NAME     PROPERTY  VALUE   SOURCE
newotus  readonly  off     default
```
Значение recordsize:
```
[root@zfs ~]# zfs get recordsize newotus
NAME     PROPERTY    VALUE    SOURCE
newotus  recordsize  128K     local
```
Тип сжатия (или параметр отключения):
```
[root@zfs ~]# zfs get compression newotus
NAME     PROPERTY     VALUE     SOURCE
newotus  compression  zle       local
```
Тип контрольной суммы:
```
[root@zfs ~]# zfs get checksum newotus
NAME     PROPERTY  VALUE      SOURCE
newotus  checksum  sha256     local
```
## 3. Работа со снапшотом, поиск сообщения от преподавателя
Скачаем файл, указанный в задании и восстановим файловую систему из снапшота:
```
[root@zfs ~]# wget -O otus_task2.file --no-check-certificate 'https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download'
...
2022-03-13 21:21:34 (7.58 MB/s) - ‘otus_task2.file’ saved [5432736/5432736]
[root@zfs ~]# zfs receive newotus/test@today < otus_task2.file
```
Далее, ищем в каталоге /newotus/test файл с именем "secret_message" и смотрим его содержимое:
```
[root@zfs ~]# find /newotus/test -name "secret_message"
/newotus/test/task1/file_mess/secret_message
[root@zfs ~]# cat /newotus/test/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome
```
Тут мы видим ссылку на GitHub

## 4. Самостоятельная работа, изучение дополнительных команд
Документация: man zpool, man zfs
Создаем пул, Смотрим информацию, Уничтожаем пул:
```
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk
sdc      8:32   0  512M  0 disk
sdd      8:48   0  512M  0 disk
sde      8:64   0  512M  0 disk
sdf      8:80   0  512M  0 disk
sdg      8:96   0  512M  0 disk
sdh      8:112  0  512M  0 disk
sdi      8:128  0  512M  0 disk
[root@zfs ~]# zpool create myzfs /dev/sdb /dev/sdc /dev/sde
[root@zfs ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
myzfs  1.41G    93K  1.41G        -         -     0%     0%  1.00x    ONLINE  -
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          sdb       ONLINE       0     0     0
          sdc       ONLINE       0     0     0
          sde       ONLINE       0     0     0

errors: No known data errors
[root@zfs ~]# zpool destroy myzfs
[root@zfs ~]# zpool list
no pools available
```
Создать зеркалированный пул, Отключить устройство от зеркалированного пула, Подключить устройство к пулу. Если пул раньше не был зеркальным, он превращается в зеркальный. Если он уже был зеркальным, он превращается в тройное зеркало.
```
[root@zfs ~]# zpool create myzfs mirror /dev/sdb /dev/sde
[root@zfs ~]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
myzfs   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors
[root@zfs ~]# zpool detach myzfs /dev/sde
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          sdb       ONLINE       0     0     0

errors: No known data errors
[root@zfs ~]# zpool attach myzfs /dev/sdb /dev/sde
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: ONLINE
  scan: resilvered 268K in 0 days 00:00:00 with 0 errors on Sun Mar 13 22:27:13 2022
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors
```
Попробовать удалить устройство из пула. Поскольку это зеркало, нужно использовать "zpool detach".
```
[root@zfs ~]# zpool remove myzfs /dev/sde
cannot remove /dev/sde: operation not supported on this type of pool
[root@zfs ~]# zpool detach myzfs /dev/sde
```
Добавить запасное устройство горячей замены (hot spare) к пулу. Удалить запасное устройство горячей замены из пула
```
[root@zfs ~]# zpool add myzfs spare /dev/sde
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: ONLINE
  scan: resilvered 268K in 0 days 00:00:00 with 0 errors on Sun Mar 13 22:27:13 2022
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          sdb       ONLINE       0     0     0
        spares
          sde       AVAIL

errors: No known data errors
[root@zfs ~]# zpool remove myzfs /dev/sde
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: ONLINE
  scan: resilvered 268K in 0 days 00:00:00 with 0 errors on Sun Mar 13 22:27:13 2022
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          sdb       ONLINE       0     0     0

errors: No known data errors
```
Вывести указанное устройство из эксплуатации (offline). После этого попыток писать и читать это устройство не будет до тех пор, пока оно не будет переведено в online. Если использовать ключ -t, устройство будет переведено в offline временно. После перезагрузки устройство опять будет в работе (online).
```
[root@zfs ~]# zpool offline myzfs /dev/sde
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: resilvered 304K in 0 days 00:00:00 with 0 errors on Sun Mar 13 22:32:00 2022
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       DEGRADED     0     0     0
          mirror-0  DEGRADED     0     0     0
            sdb     ONLINE       0     0     0
            sde     OFFLINE      0     0     0

errors: No known data errors
[root@zfs ~]# zpool online myzfs /dev/sde
```
Заменить один диск в пуле другим (например, при сбое диска):
```
[root@zfs ~]# zpool replace myzfs /dev/sdb /dev/sdc
[root@zfs ~]# zpool status -v
  pool: myzfs
 state: ONLINE
  scan: resilvered 332K in 0 days 00:00:01 with 0 errors on Sun Mar 13 22:34:56 2022
config:

        NAME        STATE     READ WRITE CKSUM
        myzfs       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdc     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors
```
Создание ещё одной файловой системы. Обратите внимание, что обе файловые системы как будто бы имеют 352M свободных, ибо квоты не установлены. Каждая может расти до тех пор, пока не заполнит пул.
Далее зарезервируем 200M для одной файловой системы, что гарантирует что остальные не заполнят всё место
```
[root@zfs ~]# zfs create myzfs/yar
[root@zfs ~]# zfs list
NAME        USED  AVAIL     REFER  MOUNTPOINT
myzfs       159K   352M       24K  /myzfs
myzfs/yar    24K   352M       24K  /myzfs/yar
[root@zfs ~]# zfs set reservation=200m myzfs/yar
[root@zfs ~]# zfs list -o reservation
RESERV
  none
  200M
[root@zfs ~]# zfs list
NAME        USED  AVAIL     REFER  MOUNTPOINT
myzfs       200M   152M     25.5K  /myzfs
myzfs/yar    24K   352M       24K  /myzfs/yar
```
Включить сжатие и проверить, что оно включилось
```
[root@zfs ~]# zfs set compression=on myzfs/yar
[root@zfs ~]# zfs list -o compression
COMPRESS
     off
      on
```
Создать снапшот (snapshot, снимок) test. Откатиться на него.:
```
[root@zfs ~]# zfs snapshot myzfs/yar@test
[root@zfs ~]# zfs list -t snapshot
NAME             USED  AVAIL     REFER  MOUNTPOINT
myzfs/yar@test     0B      -       24K  -
[root@zfs ~]# zfs snapshot myzfs@mytest
[root@zfs ~]# zfs list -t snapshot
NAME             USED  AVAIL     REFER  MOUNTPOINT
myzfs@mytest       0B      -     25.5K  -
myzfs/yar@test     0B      -       24K  -
[root@zfs ~]# zfs rollback myzfs@mytest
```
Смонтировать/Размонтировать файловую систему.
```
[root@zfs ~]# zfs umount myzfs/yar
[root@zfs ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        237M     0  237M   0% /dev
tmpfs           244M     0  244M   0% /dev/shm
tmpfs           244M  4.6M  239M   2% /run
tmpfs           244M     0  244M   0% /sys/fs/cgroup
/dev/sda1        40G  3.2G   37G   8% /
tmpfs            49M     0   49M   0% /run/user/1000
myzfs           152M  128K  152M   1% /myzfs
tmpfs            49M     0   49M   0% /run/user/0
[root@zfs ~]# zfs mount myzfs/yar
[root@zfs ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        237M     0  237M   0% /dev
tmpfs           244M     0  244M   0% /dev/shm
tmpfs           244M  4.6M  239M   2% /run
tmpfs           244M     0  244M   0% /sys/fs/cgroup
/dev/sda1        40G  3.2G   37G   8% /
tmpfs            49M     0   49M   0% /run/user/1000
myzfs           152M  128K  152M   1% /myzfs
tmpfs            49M     0   49M   0% /run/user/0
myzfs/yar       352M  128K  352M   1% /myzfs/yar
```
Посмотреть историю команд для всех пулов. Можно ограничить историю одним пулом, для этого надо указать его имя в командной строке. После того как пул уничтожен, его история теряется.
```
[root@zfs ~]# zpool history
History for 'myzfs':
2022-03-13.22:25:21 zpool create myzfs mirror /dev/sdb /dev/sde
2022-03-13.22:26:10 zpool detach myzfs /dev/sde
2022-03-13.22:27:13 zpool attach myzfs /dev/sdb /dev/sde
2022-03-13.22:29:16 zpool detach myzfs /dev/sde
2022-03-13.22:30:24 zpool add myzfs spare /dev/sde
2022-03-13.22:31:23 zpool remove myzfs /dev/sde
2022-03-13.22:32:00 zpool attach myzfs /dev/sdb /dev/sde
2022-03-13.22:33:06 zpool offline myzfs /dev/sde
2022-03-13.22:33:29 zpool online myzfs /dev/sde
2022-03-13.22:34:56 zpool replace myzfs /dev/sdb /dev/sdc
2022-03-13.22:39:24 zfs create myzfs/yar
2022-03-13.22:40:25 zfs set reservation=200m myzfs/yar
2022-03-13.22:43:44 zfs set compression=on myzfs/yar
2022-03-13.22:45:08 zfs snapshot myzfs/yar@test
2022-03-13.22:45:52 zfs snapshot myzfs@mytest
2022-03-13.22:46:36 zfs rollback myzfs@mytest
```
