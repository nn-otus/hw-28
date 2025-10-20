## ДЗ-28 Резервное копирование
#### Цель домашнего задания
Научиться настраивать резервное копирование с помощью утилиты Borg
#### Описание домашнего задания
Настроить стенд  с двумя виртуальными машинами: backup_server и client.  
Настроить удаленный бэкап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:
+ директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB; (Студент самостоятельно настраивает)
репозиторий для резервных копий должен быть зашифрован ключом или паролем - на усмотрение студента;
+ имя бэкапа должно содержать информацию о времени снятия бекапа;
+ глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
+ резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
+ написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на усмотрение студента;
+ настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов

1. Устанавливаем на client и backup сервер borgbackup
root@u22srv28:~# apt install borgbackup
![1-group_command_screen](image1.png)
2. Монтируем диск sdb в /var/backups
```
root@u22srv28:~# mkfs.ext4 /dev/sdb
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done
Creating filesystem with 8388608 4k blocks and 2097152 inodes
Filesystem UUID: fd93893f-9ecb-4480-bbd2-da51a7228c51
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624

Allocating group tables: done
Writing inode tables: done
Creating journal (65536 blocks): done
Writing superblocks and filesystem accounting information: done

root@u22srv28:~#
root@u22srv28:~# mount /dev/sdb /var/backup
root@u22srv28:~#
root@u22srv28:~# lsblk | grep sdb
sdb                         8:16   0    32G  0 disk /var/backup
root@u22srv28:~# blkid | grep sdb
/dev/sdb: UUID="fd93893f-9ecb-4480-bbd2-da51a7228c51" BLOCK_SIZE="4096" TYPE="ext4"
root@u22srv28:~# echo 'UUID=fd93893f-9ecb-4480-bbd2-da51a7228c51 /var/backup ext4 defaults 0 0' >> /etc/fstab
root@u22srv28:~# cat /etc/fstab | grep var
UUID=fd93893f-9ecb-4480-bbd2-da51a7228c51 /var/backup ext4 defaults 0 0
root@u22srv28:~#
```
3. На сервере backup создаем пользователя borg и назначаем на каталог /var/backup права пользователя borg
 ```
root@u22srv28:~# adduser borg
Adding user `borg' ...
Adding new group `borg' (1001) ...
Adding new user `borg' (1001) with group `borg' ...
Creating home directory `/home/borg' ...
root@u22srv28:~# chown borg:borg /var/backup/
 ```
4. На сервере backup создаем каталог ~/.ssh/authorized_keys в каталоге /home/borg
```
borg@u22srv28:~$ mkdir .ssh
borg@u22srv28:~$ touch .ssh/authorized_keys
borg@u22srv28:~$ chmod 700 .ssh
borg@u22srv28:~$ chmod 600 .ssh/authorized_keys
borg@u22srv28:~$
```
Переключаемся на client server u24cln28. Сгенерируем ключ

```
root@u24cln28:~# ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:CRrSmdw+m0KPHJ333LMFrig0uZddrslrleYCrr49UjE root@u24cln28
The key's randomart image is:
+--[ED25519 256]--+
|                 |
|   o +           |
|  . * o          |
|   . = oE.       |
|    + =.So  ..   |
|   o ++=oo o=.   |
|    +.+= +o*+ .  |
|     .+.=o+oo+   |
|     .+*oo*+.    |
+----[SHA256]-----+
root@u24cln28:~#
```
Инициализируем репозиторий borg на backup сервере с client сервера:
```
root@u24cln28:~# borg init --encryption=repokey borg@192.168.240.122:/var/backup/
borg@192.168.240.122's password:
Enter new passphrase:
Enter same passphrase again:
Passphrases do not match
Enter new passphrase:
Enter same passphrase again:
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "password123"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.240.122/var/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).

root@u24cln28:~#
borg@u22srv28:~$ ls -la /var/backup
total 76
drwxr-xr-x  3 borg borg  4096 окт 10 16:24 .
drwxr-xr-x 14 root root  4096 окт  9 15:42 ..
-rw-------  1 borg borg   700 окт 10 16:24 config
drwx------  3 borg borg  4096 окт 10 16:24 data
-rw-------  1 borg borg    70 окт 10 16:24 hints.1
-rw-------  1 borg borg 41258 окт 10 16:24 index.1
-rw-------  1 borg borg   190 окт 10 16:24 integrity.1
-rw-------  1 borg borg    16 окт 10 16:24 nonce
-rw-------  1 borg borg    73 окт 10 16:24 README
borg@u22srv28:~$

```
Запускаем для проверки создания бэкапа
```
root@u24cln28:~# borg create --stats --list borg@192.168.240.122:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
Enter passphrase for key ssh://borg@192.168.240.122/var/backup:
s /etc/localtime
...
...
------------------------------------------------------------------------------
Repository: ssh://borg@192.168.240.122/var/backup
Archive name: etc-2025-10-10_16:40:11
Archive fingerprint: 9b354e87cea0c8c46f7d478a6060dfafb0472b20b4dc53d3e8201a515b7beb46
Time (start): Fri, 2025-10-10 16:40:38
Time (end):   Fri, 2025-10-10 16:40:39
Duration: 1.44 seconds
Number of files: 715
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:                2.04 MB            899.49 kB            876.26 kB
All archives:                2.04 MB            898.87 kB            944.42 kB

                       Unique chunks         Total chunks
Chunk index:                     691                  713
------------------------------------------------------------------------------

```
Смотрим, что у нас получилось
```
root@u24cln28:~# borg list borg@192.168.240.122:/var/backup/
borg@192.168.240.122's password:
Enter passphrase for key ssh://borg@192.168.240.122/var/backup:
etc-2025-10-10_16:40:11              Fri, 2025-10-10 16:40:38 [9b354e87cea0c8c46f7d478a6060dfafb0472b20b4dc53d3e8201a515b7beb46]
root@u24cln28:~#

```
Смотрим список файлов
```
borg@192.168.240.122's password:
Enter passphrase for key ssh://borg@192.168.240.122/var/backup:
etc-2025-10-10_16:40:11              Fri, 2025-10-10 16:40:38 [9b354e87cea0c8c46f7d478a6060dfafb0472b20b4dc53d3e8201a515b7beb46]
root@u24cln28:~#
```
Извлекаем файл из архива
```
root@u24cln28:~# borg extract borg@192.168.240.122:/var/backup/::etc-2025-10-10_16:40:11 etc/hostname
borg@192.168.240.122's password:
Enter passphrase for key ssh://borg@192.168.240.122/var/backup:
root@u24cln28:~#
```
#### Автоматизируем создание бэкапов с помощью systemd
Создаем сервис и таймер в каталоге /etc/systemd/system/
```
root@u22srv28:~$ nano /etc/systemd/system/borg-backup.service
---
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=password123"
# Репозиторий
Environment=REPO=borg@192.168.240.122:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}
```
#### Включаем и запускаем службу таймера

```
root@u24cln28:~# nano /etc/systemd/system/borg-backup.timer
root@u24cln28:~# cat /etc/systemd/system/borg-backup.timer
[Unit]
Description=Borg Backup

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
root@u24cln28:~#

root@u24cln28:~#
root@u24cln28:~# systemctl enable borg-backup.timer
Created symlink /etc/systemd/system/timers.target.wants/borg-backup.timer → /etc/systemd/system/borg-backup.timer.
root@u24cln28:~#
root@u24cln28:~# systemctl daemon-reload
root@u24cln28:~# systemctl start borg-backup.timer

root@u24cln28:~# systemctl status borg-backup.timer
● borg-backup.timer - Borg Backup
     Loaded: loaded (/etc/systemd/system/borg-backup.timer; enabled; vendor preset: enabled)
     Active: active (elapsed) since Mon 2025-10-20 12:48:41 UTC; 39s ago
    Trigger: n/a
   Triggers: ● borg-backup.service

окт 20 12:48:41 u24cln28 systemd[1]: Started Borg Backup.
root@u24cln28:~#
```
#### Проверяем работу бэкап сервиса
```
root@u24cln28:~# systemctl list-timers borg-backup.timer
NEXT                        LEFT          LAST                        PASSED  UNIT              ACTIVATES
Mon 2025-10-20 13:24:30 UTC 4min 24s left Mon 2025-10-20 13:19:30 UTC 35s ago borg-backup.timer borg-backup.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.
root@u24cln28:~#
```
## Домашнее задание 28 выполнено
