# Домашнее задание: репликация postgres

1) Настроить hot_standby репликацию с использованием слотов
2) Настроить правильное резервное копирование

## Решение:
Для выполнения задания была использована методичка.
Был использован другой образ для создания машин, т.к. приведенный методичке не был доступен. vagrantfile опиывает создание трёх виртуальных машин (`node1`, `node2`, `barman`) и запускает Ансибл-плейбук `provision.yml`, который в свою очередь запускае роли `install_postgres` (установка Postgres на node1 и node2) `postgres_replication` (нанастройка репликации) и `install_barman` (установка barman на все ноды и его настройка).

<details><summary>Содержимое Vagrantfile:</summary>

```
ENV["LC_ALL"] = "en_US.UTF-8"
ENV['VAGRANT_SERVER_URL'] = 'https://vagrant.elab.pro'

# Описание параметров ВМ
MACHINES = {
  # Имя DV "pam"
  :node1 => {
        # VM box
        :box_name => "centos/8",
        # Имя VM
        :vm_name => "node1",
        # Количество ядер CPU
        :cpus => 2,
        # Указываем количество ОЗУ (В Мегабайтах)
        :memory => 1024,
        # Указываем IP-адрес для ВМ
        :ip => "192.168.57.11",
  },
  :node2 => {
        :box_name => "centos/8",
        :vm_name => "node2",
        :cpus => 2,
        :memory => 1024,
        :ip => "192.168.57.12",

  },
  :barman => {
        :box_name => "centos/8",
        :vm_name => "barman",
        :cpus => 1,
        :memory => 1024,
        :ip => "192.168.57.13",

  },

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
    
    config.vm.define boxname do |box|
   
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      box.vm.network "private_network", ip: boxconfig[:ip]
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end

      # Запуск ansible-playbook
      if boxconfig[:vm_name] == "barman"
       box.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/provision.yml"
        ansible.inventory_path = "ansible/hosts"
        ansible.host_key_checking = "false"
        ansible.limit = "all"
       end
      end
    end
  end
end
```
</details>


В шаблонах конфигураций Postgres (`pg_hba.conf.j2` и `postgresql.conf.j2`) вместо жесткого указания ip адресов использованы переменные из `/defaults/main.yml`.

В шаблонах конфигурации `barman` вместо явного указания имени пользователя, пароля и ip адресов были использованы переменные.

Кроме того, были внесены исправления в `/tasks/main.yml` для достижения рабоспособности роли.  Например, потребовалось добавить task для создания директории /etc/barman.d/:

```
# Создаём директорию /etc/barman.d/
  - name: Create a directory if it does not exist
    ansible.builtin.file:
      path: /etc/barman.d/
      state: directory
      owner: barman
      group: barman
      mode: '0755'
    when: (ansible_hostname == "barman")   
```

Содержимое лога работы команды `vagrant up` ниже:

<details><summary>ЛОГ vagrant up > bashlog.log</summary>

Bringing machine 'node1' up with 'virtualbox' provider...
Bringing machine 'node2' up with 'virtualbox' provider...
Bringing machine 'barman' up with 'virtualbox' provider...
==> node1: Importing base box 'centos/8'...

[KProgress: 20%
[KProgress: 40%
[KProgress: 50%
[KProgress: 70%
[KProgress: 90%
[K==> node1: Matching MAC address for NAT networking...
==> node1: Checking if box 'centos/8' version '1.0.0' is up to date...
==> node1: Setting the name of the VM: OTUS-Task28_node1_1712776187206_47806
==> node1: Clearing any previously set network interfaces...
==> node1: Preparing network interfaces based on configuration...
    node1: Adapter 1: nat
    node1: Adapter 2: hostonly
==> node1: Forwarding ports...
    node1: 22 (guest) => 2222 (host) (adapter 1)
==> node1: Running 'pre-boot' VM customizations...
==> node1: Booting VM...
==> node1: Waiting for machine to boot. This may take a few minutes...
    node1: SSH address: 127.0.0.1:2222
    node1: SSH username: vagrant
    node1: SSH auth method: private key
    node1: 
    node1: Vagrant insecure key detected. Vagrant will automatically replace
    node1: this with a newly generated keypair for better security.
    node1: 
    node1: Inserting generated public key within guest...
    node1: Removing insecure key from the guest if it's present...
    node1: Key inserted! Disconnecting and reconnecting using new SSH key...
==> node1: Machine booted and ready!
==> node1: Checking for guest additions in VM...
    node1: No guest additions were detected on the base box for this VM! Guest
    node1: additions are required for forwarded ports, shared folders, host only
    node1: networking, and more. If SSH fails on this machine, please install
    node1: the guest additions and repackage the box to continue.
    node1: 
    node1: This is not an error message; everything may continue to work properly,
    node1: in which case you may ignore this message.
==> node1: Setting hostname...
==> node1: Configuring and enabling network interfaces...
==> node1: Rsyncing folder: /home/andrej/Nextcloud/Документы/OTUS-HOMETASKS/OTUS-Task28/ => /vagrant
==> node2: Importing base box 'centos/8'...

[KProgress: 20%
[KProgress: 40%
[KProgress: 50%
[KProgress: 70%
[KProgress: 90%
[K==> node2: Matching MAC address for NAT networking...
==> node2: Checking if box 'centos/8' version '1.0.0' is up to date...
==> node2: Setting the name of the VM: OTUS-Task28_node2_1712776248723_54683
==> node2: Fixed port collision for 22 => 2222. Now on port 2200.
==> node2: Clearing any previously set network interfaces...
==> node2: Preparing network interfaces based on configuration...
    node2: Adapter 1: nat
    node2: Adapter 2: hostonly
==> node2: Forwarding ports...
    node2: 22 (guest) => 2200 (host) (adapter 1)
==> node2: Running 'pre-boot' VM customizations...
==> node2: Booting VM...
==> node2: Waiting for machine to boot. This may take a few minutes...
    node2: SSH address: 127.0.0.1:2200
    node2: SSH username: vagrant
    node2: SSH auth method: private key
    node2: 
    node2: Vagrant insecure key detected. Vagrant will automatically replace
    node2: this with a newly generated keypair for better security.
    node2: 
    node2: Inserting generated public key within guest...
    node2: Removing insecure key from the guest if it's present...
    node2: Key inserted! Disconnecting and reconnecting using new SSH key...
==> node2: Machine booted and ready!
==> node2: Checking for guest additions in VM...
    node2: No guest additions were detected on the base box for this VM! Guest
    node2: additions are required for forwarded ports, shared folders, host only
    node2: networking, and more. If SSH fails on this machine, please install
    node2: the guest additions and repackage the box to continue.
    node2: 
    node2: This is not an error message; everything may continue to work properly,
    node2: in which case you may ignore this message.
==> node2: Setting hostname...
==> node2: Configuring and enabling network interfaces...
==> node2: Rsyncing folder: /home/andrej/Nextcloud/Документы/OTUS-HOMETASKS/OTUS-Task28/ => /vagrant
==> barman: Importing base box 'centos/8'...

[KProgress: 20%
[KProgress: 40%
[KProgress: 50%
[KProgress: 70%
[KProgress: 90%
[K==> barman: Matching MAC address for NAT networking...
==> barman: Checking if box 'centos/8' version '1.0.0' is up to date...
==> barman: Setting the name of the VM: OTUS-Task28_barman_1712776307701_45220
==> barman: Fixed port collision for 22 => 2222. Now on port 2201.
==> barman: Clearing any previously set network interfaces...
==> barman: Preparing network interfaces based on configuration...
    barman: Adapter 1: nat
    barman: Adapter 2: hostonly
==> barman: Forwarding ports...
    barman: 22 (guest) => 2201 (host) (adapter 1)
==> barman: Running 'pre-boot' VM customizations...
==> barman: Booting VM...
==> barman: Waiting for machine to boot. This may take a few minutes...
    barman: SSH address: 127.0.0.1:2201
    barman: SSH username: vagrant
    barman: SSH auth method: private key
    barman: 
    barman: Vagrant insecure key detected. Vagrant will automatically replace
    barman: this with a newly generated keypair for better security.
    barman: 
    barman: Inserting generated public key within guest...
    barman: Removing insecure key from the guest if it's present...
    barman: Key inserted! Disconnecting and reconnecting using new SSH key...
==> barman: Machine booted and ready!
==> barman: Checking for guest additions in VM...
    barman: No guest additions were detected on the base box for this VM! Guest
    barman: additions are required for forwarded ports, shared folders, host only
    barman: networking, and more. If SSH fails on this machine, please install
    barman: the guest additions and repackage the box to continue.
    barman: 
    barman: This is not an error message; everything may continue to work properly,
    barman: in which case you may ignore this message.
==> barman: Setting hostname...
==> barman: Configuring and enabling network interfaces...
==> barman: Rsyncing folder: /home/andrej/Nextcloud/Документы/OTUS-HOMETASKS/OTUS-Task28/ => /vagrant
==> barman: Running provisioner: ansible...
    barman: Running ansible-playbook...

PLAY [Postgres] ****************************************************************

TASK [Gathering Facts] *********************************************************
ok: [barman]
ok: [node1]
ok: [node2]

TASK [fix repos] ***************************************************************
changed: [node1]
changed: [barman]
changed: [node2]

TASK [install base tools] ******************************************************
changed: [barman]
changed: [node2]
changed: [node1]

PLAY [install postgres 14 and set up replication] ******************************

TASK [Gathering Facts] *********************************************************
ok: [node2]
ok: [node1]

TASK [install_postgres : disable firewalld service] ****************************
ok: [node2]
ok: [node1]

TASK [install_postgres : Disable SELinux] **************************************
[WARNING]: SELinux state temporarily changed from 'enforcing' to 'permissive'.
State change will take effect next reboot.
changed: [node1]
changed: [node2]

TASK [install_postgres : Ensure SELinux is set to disable mode] ****************
ok: [node2]
ok: [node1]

TASK [install_postgres : install repo] *****************************************
changed: [node2]
changed: [node1]

TASK [install_postgres : disable old postgresql module] ************************
changed: [node2]
changed: [node1]

TASK [install_postgres : install postgresql-server 14] *************************
changed: [node1]
changed: [node2]

TASK [install_postgres : check init] *******************************************
ok: [node1]
ok: [node2]

TASK [install_postgres : initialization setup] *********************************
changed: [node1]
changed: [node2]

TASK [install_postgres : enable and start service] *****************************
changed: [node2]
changed: [node1]

TASK [postgres_replication : install base tools] *******************************
changed: [node1]
changed: [node2]

TASK [postgres_replication : Create replicator user] ***************************
skipping: [node2]
[WARNING]: Module remote_tmp /var/lib/pgsql/.ansible/tmp did not exist and was
created with a mode of 0700, this may cause issues when running as another
user. To avoid this, create the remote_tmp dir with the correct permissions
manually
changed: [node1]

TASK [postgres_replication : stop postgresql-server on node2] ******************
skipping: [node1]
changed: [node2]

TASK [postgres_replication : copy postgresql.conf] *****************************
skipping: [node2]
changed: [node1]

TASK [postgres_replication : copy pg_hba.conf] *********************************
skipping: [node2]
changed: [node1]

TASK [postgres_replication : restart postgresql-server on node1] ***************
skipping: [node2]
changed: [node1]

TASK [postgres_replication : Remove files from data catalog] *******************
skipping: [node1]
changed: [node2]

TASK [postgres_replication : copy files from master to slave] ******************
skipping: [node1]
changed: [node2]

TASK [postgres_replication : copy postgresql.conf] *****************************
skipping: [node1]
ok: [node2]

TASK [postgres_replication : copy pg_hba.conf] *********************************
skipping: [node1]
ok: [node2]

TASK [postgres_replication : start postgresql-server on node2] *****************
skipping: [node1]
changed: [node2]

PLAY [set up backup] ***********************************************************

TASK [Gathering Facts] *********************************************************
ok: [node2]
ok: [node1]
ok: [barman]

TASK [install_barman : install base tools] *************************************
changed: [barman]
changed: [node1]
changed: [node2]

TASK [install_barman : disable firewalld service] ******************************
skipping: [node1]
skipping: [node2]
ok: [barman]

TASK [install_barman : Disable SELinux] ****************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : Ensure SELinux is set to disable mode] ******************
skipping: [node1]
skipping: [node2]
ok: [barman]

TASK [install_barman : install repo] *******************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : disable old postgresql module] **************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : install epel-release] ***********************************
changed: [barman]
changed: [node2]
changed: [node1]

TASK [install_barman : install barman and postgresql packages on barman] *******
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : install barman-cli and postgresql packages on nodes] ****
skipping: [barman]
changed: [node2]
changed: [node1]

TASK [install_barman : generate SSH key for postgres] **************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : generate SSH key for barman] ****************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : fetch all public ssh keys node1] ************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : transfer public key to barman] **************************
skipping: [node2]
skipping: [barman]
changed: [node1 -> barman(192.168.57.13)]

TASK [install_barman : fetch all public ssh keys barman] ***********************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : transfer public key to barman] **************************
skipping: [node1]
skipping: [node2]
changed: [barman -> node1(192.168.57.11)]

TASK [install_barman : Create barman user] *************************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : Add permission for barman] ******************************
skipping: [barman]
changed: [node1]
changed: [node2]

TASK [install_barman : Add permission for barman] ******************************
skipping: [barman]
changed: [node1]
changed: [node2]

TASK [install_barman : restart postgresql-server on node1] *********************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : Create DB for backup] ***********************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : Add tables to otus_backup] ******************************
skipping: [node2]
skipping: [barman]
changed: [node1]

TASK [install_barman : copy .pgpass] *******************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : copy barman.conf] ***************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : Create a directory if it does not exist] ****************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : copy node1.conf] ****************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

TASK [install_barman : barman switch-wal node1] ********************************
skipping: [node1]
skipping: [node2]
changed: [barman]
[WARNING]: Module remote_tmp /var/lib/barman/.ansible/tmp did not exist and was
created with a mode of 0700, this may cause issues when running as another
user. To avoid this, create the remote_tmp dir with the correct permissions
manually

TASK [install_barman : barman cron] ********************************************
skipping: [node1]
skipping: [node2]
changed: [barman]

PLAY RECAP *********************************************************************
barman                     : ok=21   changed=17   unreachable=0    failed=0    skipped=10   rescued=0    ignored=0   
node1                      : ok=31   changed=25   unreachable=0    failed=0    skipped=21   rescued=0    ignored=0   
node2                      : ok=26   changed=18   unreachable=0    failed=0    skipped=26   rescued=0    ignored=0   


</details>

---
### Проверка работы barman

```bash
[vagrant@barman ~]$ sudo -u barman  barman check node1
Server node1:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
        backup minimum size: OK (0 B)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK (no system Id stored on disk)
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK
```
Создание резервной копии:
```
[vagrant@barman ~]$ sudo -u barman barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20240410T194750
Backup start at LSN: 0/7000100 (000000010000000000000007, 00000100)
Starting backup copy via pg_basebackup for 20240410T194750
Copy done (time: less than one second)
Finalising the backup.
Backup size: 33.6 MiB
Backup end at LSN: 0/9000000 (000000010000000000000008, 00000000)
Backup completed (start time: 2024-04-10 19:47:50.155723, elapsed time: less than one second)
Processing xlog segments from streaming for node1
        000000010000000000000007
```
Проверка восстановления из бэкапов:
- удаляем базу `otus`
```bash
[vagrant@node1 ~]$ sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.11)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# DROP DATABASE otus;
DROP DATABASE
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)

postgres=# 
```

- восстанавливаем данные:
```bash
[vagrant@barman ~]$ sudo -u barman barman list-backup node1
node1 20240410T194750 - Wed Apr 10 16:47:50 2024 - Size: 33.6 MiB - WAL Size: 0 B
node1 20240410T194518 - Wed Apr 10 16:45:18 2024 - Size: 33.6 MiB - WAL Size: 32.2 KiB
[vagrant@barman ~]$ sudo -u barman barman recover node1 20240410T194750 /var/lib/pgsql/14/data/ --remote-ssh-comman "ssh postgres@192.168.57.11"
The authenticity of host '192.168.57.11 (192.168.57.11)' can't be established.
ECDSA key fingerprint is SHA256:c3F5W3jVuqKdeGGg6M/UUUsUMz6i2cQbzsXJnXmlQkA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20240410T194750
Destination directory: /var/lib/pgsql/14/data/
Remote command: ssh postgres@192.168.57.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2024-04-10 19:55:01.504875+00:00, elapsed time: 5 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```
Проверяем, что удалённая база снова на своём месте:
```bash
[vagrant@node1 ~]$ sudo -i
[root@node1 ~]# systemctl restart postgresql-14.service 
[root@node1 ~]# exit
logout
[vagrant@node1 ~]$ sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.11)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# 
```

Файл /var/lib/pgsql/14/data/postgresql.conf c `node1` выглядит так:
```bash
#Указываем ip-адреса, на которых postgres будет слушать трафик на порту 5432 (параметр port)
listen_addresses = 'localhost, 192.168.57.11'
#Указываем порт порт postgres
port = 5432
#Устанавливаем максимально 100 одновременных подключений
max_connections = 100
log_directory = 'log'
log_filename = 'postgresql-%a.log'
log_rotation_age = 1d
log_rotation_size = 0
log_truncate_on_rotation = on
max_wal_size = 1GB
min_wal_size = 80MB
log_line_prefix = '%m [%p] '
#Указываем часовой пояс для Москвы
log_timezone = 'UTC+3'
timezone = 'UTC+3'
datestyle = 'iso, mdy'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
#можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления; 
hot_standby = on
#Включаем репликацию
wal_level = replica
#Количество планируемых слейвов
max_wal_senders = 3
#Максимальное количество слотов репликации
max_replication_slots = 3
#будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.
hot_standby_feedback = on
#Включаем использование зашифрованных паролей
password_encryption = scram-sha-256
~                                     
```

Файл /var/lib/pgsql/14/data/pg_hba.conf выглядит так:
```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all                  all                                                peer
# IPv4 local connections:
host    all                  all             127.0.0.1/32              scram-sha-256
# IPv6 local connections:
host    all                  all             ::1/128                   scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication      all                                                peer
host    replication      all             127.0.0.1/32               scram-sha-256
host    replication      all             ::1/128                    scram-sha-256
host    replication replication      192.168.57.11/32               scram-sha-256
host    replication replication      192.168.57.12/32               scram-sha-256
host    all           barman         192.168.57.13/32               scram-sha-256
host    replication   barman         192.168.57.13/32               scram-sha-256
```
Файл /etc/barman.d/node1.conf на ноде `barman` выглядит так:
```bash
[node1]
#Описание задания
description = "backup node1"
#Команда подключения к хосту node1
ssh_command = ssh postgres@192.168.57.11
#Команда для подключения к postgres-серверу
conninfo = host=192.168.57.11  user=barman port=5432 dbname=postgres
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 7 days
wal_retention_policy = main
streaming_archiver=on
#Указание префикса, который будет использоваться как $PATH на хосте node1
path_prefix = /usr/pgsql-14/bin/
#настройки слота
create_slot = auto
slot_name = node1
#Команда для потоковой передачи от postgres-сервера
streaming_conninfo = host=192.168.57.11  user=barman
#Тип выполняемого бекапа
backup_method = postgres
archiver = off
```