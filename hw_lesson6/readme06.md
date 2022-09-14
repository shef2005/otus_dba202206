## Выполнение домашнего задания по теме "Установка PostgreSQL"
## 1 вариант:

### Создаем экземпляр VM в CGP
![VMCreated](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/1/VMCreated.PNG)

### Установка PostgreSQL 14
1. Для установки PostgreSQL последней версии необходимо выполнить следующую команду:
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14 && sudo apt install unzip
```
2. После установки убеждаемся, что кластер запустился при помощи `pg_lsclusters` . Если кластер запущен, получаем примерно следующий вывод:
```
shef2005@postgres-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

### Создание тестовой таблицы и внесение данных
1. Подключаемся к базе данных `postgres` при помощи команды `sudo -i -u postgres psql`.
2. Создаем таблицу `test` и наполнякем ее произвольными данными:
``` 
create table test(c1 text);
insert into test values('1');
insert into test values('2');
```
3. Проверяем содержимое таблицы:
```
postgres=# select * from test;
 c1
----
 1
 2
(2 rows)
```

### Создание внешнего диска и добавление к существующему экземпляру ВМ
1. Останавливаем наш кластер PostgreSQL при помощи команды `sudo -i -u postgres pg_ctlcluster 14 main stop`:
```
shef2005@postgres-2:~$ sudo -i -u postgres pg_ctlcluster 13 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@13-main
```
2. Убеждаемся, что кластер остановлен:
```
shef2005@postgres-2:~$ sudo -i -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 down   postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-14-main.log
```
3. Создаем новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB

![DiskCreated](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/1/DiskCreated.PNG)

4. Добавляем диск на виртуальную машину:

![DiskAdded](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/1/DiskAdded.PNG)

5. На самой виртуальной машине проверяем, подтянулся ли наш диск при помощи команды `lsblk`:
```
shef2005@postgres-2:~$ lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0    31M  1 loop /snap/snapd/9721
loop1     7:1    0  55.4M  1 loop /snap/core18/1932
loop2     7:2    0  67.8M  1 loop /snap/lxd/18150
loop3     7:3    0 128.8M  1 loop /snap/google-cloud-sdk/159
sda       8:0    0    10G  0 disk
├─sda1    8:1    0   9.9G  0 part /
├─sda14   8:14   0     4M  0 part
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0    10G  0 disk
```
Как видим, наш диск добавлен с DEVICE_ID sdb.

### Создание файловой системы на новом диске, монтирование новой директории

1. Форматируем диск с помощью инструмента `mkfs`, выполняя команду `sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb`:
```
shef2005@postgres-2:~$ sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: e2c1085f-1b4a-4be9-9118-0e998cb4edfd
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
2. Создаем директорию для данных при помощи команды `sudo mkdir -p /mnt/data/postgres`.
3. Монтируем новую директорию на добавленный диск - `sudo mount -o discard,defaults /dev/sdb /mnt/data/postgres`.
4. Проверяем наш новый диск:
```
shef2005@postgres-2:~$ lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0    31M  1 loop /snap/snapd/9721
loop1     7:1    0  55.4M  1 loop /snap/core18/1932
loop2     7:2    0  67.8M  1 loop /snap/lxd/18150
loop3     7:3    0 128.8M  1 loop /snap/google-cloud-sdk/159
sda       8:0    0    10G  0 disk
├─sda1    8:1    0   9.9G  0 part /
├─sda14   8:14   0     4M  0 part
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0    10G  0 disk /mnt/data/postgres
```
Новая директория примонтирована на sdb.

5. Добавляем права на запись в нашу новую директорию - `sudo chmod a+w /mnt/data/postgres`.
6. Теперь необходимо добавить правило в fstab, чтобы после перезапуска вм наш диск оставался примонтированным. При помощи `blkid` узнаем UUID для нашего диска:
```
shef2005@postgres-2:~$ sudo blkid /dev/sdb
/dev/sdb: UUID="e2c1085f-1b4a-4be9-9118-0e998cb4edfd" TYPE="ext4"
```
7. Открываем `/etc/fstab` через `vim` и добавляем запись в правило:

```
LABEL=cloudimg-rootfs   /        ext4   defaults        0 0
LABEL=UEFI      /boot/efi       vfat    defaults        0 0
UUID=e2c1085f-1b4a-4be9-9118-0e998cb4edfd /mnt/data/postgres ext4 discard,defaults,nofail 0 2
``` 
Для применения правила выполняем `sudo mount -a`.

### Перенос базы данных в новую директорию

1. Назначаем пользователя `postgres` владельцем новой директории - `sudo chown -R postgres:postgres /mnt/data/postgres`.
2. Переносим все данные PostgreSQL на новый сервер - `sudo mv /var/lib/postgresql/13 /mnt/data/postgres`.
3. Пробуем запустить кластер main:
```
shef2005@postgres-2:~$ sudo -i -u postgres pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist
```
Как видим, мы получаем ошибку, что кластер либо непоступен, либо вообще не существует.

4. Для запуска кластера необходимо изменить параметр `data_directory` файла `/etc/postgresql/13/main/postgresql.conf`:
```
sudo vim /etc/postgresql/14/main/postgresql.conf
---
data_directory = '/mnt/data/postgres/13/main'          # use data in another directory
                                        # (change requires restart)
hba_file = '/etc/postgresql/14/main/pg_hba.conf'        # host-based authentication file
                                        # (change requires restart)
ident_file = '/etc/postgresql/13/main/pg_ident.conf'    # ident configuration file
                                        # (change requires restart)
---
```
5. Пробуем запустить кластер еще раз:
```
shef2005@postgres-2:~$ sudo -i -u postgres pg_ctlcluster 14 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@13-main
```
6. Проверяем статус:
```
shef2005@postgres-2:~$ sudo -i -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory             Log file
13  main    5432 online postgres /mnt/data/postgres/14/main /var/log/postgresql/postgresql-13-main.log
```
Как видим, кластер успешно запущен.
7. Чтобы убедиться, что наши данные никуда не пропали, зайдем в базу данных `postgres` и проверим нашу таблицу `test`:
```
shef2005@postgres-2:~$ sudo -i -u postgres psql
psql (13.1 (Ubuntu 14.5.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
 2
(2 rows)
```
Наши данные не пропали. 

## Задание со звездочкой

1. Чтобы наш кластер работал как прежде, вернем все как было:
```
- sudo -i -u postgres pg_ctlcluster 14 main stop
- sudo vim /etc/postgresql/13/main/postgresql.conf
---
data_directory = '/mnt/data/postgres/14/main'          # use data in another directory
                                        # (change requires restart)
hba_file = '/etc/postgresql/14/main/pg_hba.conf'        # host-based authentication file
                                        # (change requires restart)
ident_file = '/etc/postgresql/14/main/pg_ident.conf'    # ident configuration file
                                        # (change requires restart)
---                                        
- sudo chown -R postgres:postgres /var/lib/postgresql
- sudo -i -u postgres pg_ctlcluster 14 main start
```
2. Отключаем наш диск от виртуальной машины:

![DiskRemoved](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/1/DiskRemoved.PNG)

3. Создаем новый экземпляр виртуальной машины и устанавливаем PostgreSQL 14 (см. пункты "Установка PostgreSQL 14"):

![VM2Created](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/1/VM2Created.PNG)
```
shef2005@postgres-2-2:~$ sudo -i -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
```
3. Добавляем наш диск с данными прошлой вм на новую машину:

![DiskAdded2](https://github.com/apovyshev/PostgreSQL/blob/main/02.PostgresSetup/1/DiskAdded2.PNG)
```
shef2005@postgres-2-2:~$ lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0     7:0    0    31M  1 loop /snap/snapd/9721
loop1     7:1    0  55.4M  1 loop /snap/core18/1932
loop2     7:2    0  67.8M  1 loop /snap/lxd/18150
loop3     7:3    0 128.8M  1 loop /snap/google-cloud-sdk/159
sda       8:0    0    10G  0 disk
├─sda1    8:1    0   9.9G  0 part /
├─sda14   8:14   0     4M  0 part
└─sda15   8:15   0   106M  0 part /boot/efi
sdb       8:16   0    10G  0 disk
```
4. Останавливаем наш кластер - `sudo -i -u postgres pg_ctlcluster 14 main stop`.
5. Удаляем все содержимое данных PostgreSQL - `sudo rm -rf  /var/lib/postgresql/*`.
6. Проверяем, что в папке /var/lib/postgresql/ ничего нет:
```
shef2005@postgres-2-2:/var/lib/postgresql$ ll
total 8
drwxr-xr-x  2 postgres postgres 4096 Nov 13 10:55 ./
drwxr-xr-x 41 root     root     4096 Nov 13 10:50 ../
```
7. Монтируем наш диск в домашнюю директорию PostgreSQL - `sudo mount -o discard,defaults /dev/sdb /var/lib/postgresql/`.
8. Снова проверяем содержимое папки:
```
shef2005@postgres-2-2:/var/lib/postgresql$ ll
total 28
drwxrwxrwx  4 postgres postgres  4096 Nov 13 10:19 ./
drwxr-xr-x 41 root     root      4096 Nov 13 10:50 ../
drwxr-xr-x  3 postgres postgres  4096 Nov 13 09:27 13/
drwx------  2 postgres postgres 16384 Nov 13 10:00 lost+found/
```
Как видим, подтянулись все данные из диска.

9. Чтобы проверить целостность наших таблиц, заходим в базу и смотрим данные:
```
shef2005@postgres-2-2:/var/lib/postgresql$ sudo -i -u postgres pg_ctlcluster 14 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@13-main
shef2005@postgres-2-2:/var/lib/postgresql$ sudo -i -u postgres psql
psql (13.1 (Ubuntu 13.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
 2
(2 rows)

postgres=#
```
10. Чтобы диск не отвалился после перезагрузки, создаем правило в fstab (см. пункты 6,7 раздела "Создание файловой системы на новом диске, монтирование новой директории").