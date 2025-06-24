# Домашнее задание Стенд ZFS    

### Введение    

1. Введение    
ZFS(Zettabyte File System) — файловая система, созданная компанией Sun Microsystems в 2004-2005 годах для ОС Solaris. Эта файловая система поддерживает большие объёмы данных, объединяет в себе концепции файловой системы, RAID-массивов, менеджера логических дисков и принципы легковесных файловых систем.    

ZFS продолжает активно развиваться. К примеру проект FreeNAS использует возможности ZFS для реализации ОС для управления SAN/NAS хранилищ.    

Из-за лицензионных ограничений, поддержка ZFS в GNU/Linux ограничена. По умолчанию ZFS отсутствует в ядре linux.      

Основное преимущество ZFS — это её полный контроль над физическими носителями, логическими томами и постоянное поддержание консистенции файловой системы. Оперируя на разных уровнях абстракции данных, ZFS способна обеспечить высокую скорость доступа к ним, контроль их целостности, а также минимизацию фрагментации данных. ZFS гибко настраивается, позволяет в процессе работы изменять объём дискового пространства и задавать разный размер блоков данных для разных применений, обеспечивает параллельность выполнения операций чтения-записи    

2. Цели домашнего задания    
Научится самостоятельно устанавливать ZFS, настраивать пулы, изучить основные возможности ZFS.      

3. Описание домашнего задания     
Определить алгоритм с наилучшим сжатием:     
Определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);     
создать 4 файловых системы на каждой применить свой алгоритм сжатия;     
для сжатия использовать либо текстовый файл, либо группу файлов.    
Определить настройки пула.     
С помощью команды zfs import собрать pool ZFS.     
Командами zfs определить настройки:     
    - размер хранилища;    
    - тип pool;    
    - значение recordsize;     
    - какое сжатие используется;     
    - какая контрольная сумма используется.     
    
Работа со снапшотами:     
скопировать файл из удаленной директории;    
восстановить файл локально. zfs receive;    
найти зашифрованное сообщение в файле secret_message.     

### Выполнение. Подготовка стенда.    

Использовал Vagrantfile из методички. Cоздал в заранее подготовленном каталоге /home/guared/zfs     
Установку необходимого софта прописал в скрипт ZFS.sh Запуск скрипта внес в конфигурацию Vagrantfile     
После успешного развёртывания ВМ, подключился по ssh и приступил к выполнению команд.     

Посмотрел список всех дисков, которые есть в виртуальной машине     

```shell
[root@guared vagrant]# lsblk
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

1. Создал mirror pool из дисков

```shell
[root@guared vagrant]# zpool create otus1 mirror /dev/sdb /dev/sdc

Создал ещё 3 пула:

[root@guared vagrant]# zpool create otus2 mirror /dev/sdd /dev/sde
[root@guared vagrant]# zpool create otus3 mirror /dev/sdf /dev/sdg
[root@guared vagrant]# zpool create otus4 mirror /dev/sdh /dev/sdi
```     

Посмотрел информацию о пулах:

```shell
[root@guared vagrant]# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M   104K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M   106K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```     

2. Создал файловые системы с разным сжатием

```shell
[root@guared vagrant]# zfs set compression=lzjb otus1
[root@guared vagrant]# zfs set compression=lz4 otus2
[root@guared vagrant]# zfs set compression=gzip-9 otus3
[root@guared vagrant]# zfs set compression=zle otus4

[root@guared ~]# zfs create -o compression=off mypool/dir_off
[root@guared ~]# zfs create -o compression=lzjb mypool/dir_lzjb
[root@guared ~]# zfs create -o compression=gzip mypool/dir_gzip
[root@guared ~]# zfs create -o compression=zle mypool/dir_zle
```     

Проверил, что все файловые системы имеют разные методы сжатия:

```shell
[root@guared vagrant]# zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```     

3. Скачал для теста ядро linux и распаковал его в разные файловые системы     

```shell
[root@guared ~]# wget -q https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.6.14.tar.xz
[root@guared ~]# tar -xf linux-5.6.14.tar.xz -C /mypool/dir_off
```     
4. Сжатие файлов будет работать только с файлами, которые были добавлены после включение настройки сжатия,
поэтому скачал один и тот же текстовый файл во все пулы:     

```shell
[root@guared vagrant]# for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
--2024-11-27 07:50:19--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: '/otus1/pg2600.converter.log'

100%[==========================================================================================================================================================>] 41,098,441  1.51MB/s   in 30s

2024-11-27 07:50:49 (1.30 MB/s) - '/otus1/pg2600.converter.log' saved [41098441/41098441]

--2024-11-27 07:50:49--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: '/otus2/pg2600.converter.log'

100%[==========================================================================================================================================================>] 41,098,441  1.44MB/s   in 27s

2024-11-27 07:51:17 (1.45 MB/s) - '/otus2/pg2600.converter.log' saved [41098441/41098441]

--2024-11-27 07:51:17--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: '/otus3/pg2600.converter.log'

100%[==========================================================================================================================================================>] 41,098,441  2.27MB/s   in 29s

2024-11-27 07:51:47 (1.37 MB/s) - '/otus3/pg2600.converter.log' saved [41098441/41098441]

--2024-11-27 07:51:47--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: '/otus4/pg2600.converter.log'

100%[==========================================================================================================================================================>] 41,098,441  1.43MB/s   in 25s

2024-11-27 07:52:13 (1.54 MB/s) - '/otus4/pg2600.converter.log' saved [41098441/41098441]

Проверил, что файл был скачан во все пулы:
[root@guared vagrant]# ls -l /otus*
/otus1:
total 22090
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log

/otus2:
total 18003
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log

/otus3:
total 10964
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log

/otus4:
total 40164
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log
```     

Уже на этом этапе видно, что самый оптимальный метод сжатия используется в пуле otus3.    
Проверил, сколько места занимает один и тот же файл в разных пулах и проверил степень сжатия файлов:     

```shell
[root@guared vagrant]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.7M   330M     21.6M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.3M   313M     39.2M  /otus4

[root@guared vagrant]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                  -
otus2  compressratio         2.23x                  -
otus3  compressratio         3.66x                  -
otus4  compressratio         1.00x                  -
```     

Таким образом, получается, что алгоритм gzip-9 самый эффективный по сжатию.     

5. Определение настроек пула.     
Скачиваю архив в домашний каталог:     

```shell
[root@guared vagrant]# wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
--2024-11-27 07:59:07--  https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 142.250.185.97, 2a00:1450:4001:828::2001
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|142.250.185.97|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/octet-stream]
Saving to: 'archive.tar.gz'

100%[==========================================================================================================================================================>] 7,275,140   1.18MB/s   in 6.1s

2024-11-27 07:59:22 (1.14 MB/s) - 'archive.tar.gz' saved [7275140/7275140]
```     

Разархивировал его:    

```shell
[root@guared vagrant]# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```     

Проверил, возможно ли импортировать данный каталог в пул:     

```shell
[root@guared vagrant]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        otus                                 ONLINE
          mirror-0                           ONLINE
            /home/vagrant/zpoolexport/filea  ONLINE
            /home/vagrant/zpoolexport/fileb  ONLINE
```    

Данный вывод показывает имя пула, тип raid и его состав.      
Сделал импорт данного пула в ОС:     

```shell
[root@guared vagrant]# zpool import -d zpoolexport/ otus
[root@guared vagrant]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

        NAME                                 STATE     READ WRITE CKSUM
        otus                                 ONLINE       0     0     0
          mirror-0                           ONLINE       0     0     0
            /home/vagrant/zpoolexport/filea  ONLINE       0     0     0
            /home/vagrant/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
```      

Команда zpool status выдает информацию о составе импортированного пула.     

6. Далее нужно определить настройки: zpool get all otus      
Запрос сразу всех параметром файловой системы: zfs get all otus      

```shell
[root@guared vagrant]# zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               off                    default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
```      

C помощью команды grep можно уточнить конкретный параметр, например:     
Размер:     

```shell
[root@guared vagrant]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M

Тип:
[root@guared vagrant]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default

Значение recordsize:
[root@guared ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local

Тип сжатия (или параметр отключения):
[root@guared ~]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local

Тип контрольной суммы:
[root@guared ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```     

7. Работа со снапшотом, поиск сообщения от преподавателя     
Скачал файл, указанный в задании:     

```shell
[root@guared ~]# wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
[1] 2891
[root@guared ~]# --2024-11-29 07:23:03--  https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 142.250.186.33, 2a00:1450:4001:830::2001
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|142.250.186.33|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5432736 (5.2M) [application/octet-stream]
Saving to: 'otus_task2.file'

100%[==========================================================================================================================================================>] 5,432,736   3.32MB/s   in 1.6s

2024-11-29 07:23:12 (3.32 MB/s) - 'otus_task2.file' saved [5432736/5432736]


[1]+  Done                    wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI
```     
Восстановил файловую систему из снапшота:
```shell
[root@guared ~]# zfs receive otus/test@today < otus_task2.file
```     
Далее, выполнил поиск в каталоге /otus/test файл с именем “secret_message”:
```shell
[root@guared ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
```     
Посмотрел содержимое найденного файла:
```shell
[root@guared ~]# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
```     

____________________________
end
