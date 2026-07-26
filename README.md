# С помощью Vagrantfile создаем 2 ВМ backup и backupclient. К ВМ подключаем диск 2 Гб и монтируем в /var/backup
# Vagrantfile
MACHINES = {
  :backup => {
    :box_name => "ubuntu/jammy64",
    :cpus => 2,
    :memory => 1512,
    :ip => "192.168.56.16",
  },
  :backupclient => {
    :box_name => "ubuntu/jammy64",
    :cpus => 2,
    :memory => 1512,
    :ip => "192.168.56.17",
  }
}

Vagrant.configure("2") do |config|
  
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version] if boxconfig[:box_version]
      box.vm.hostname = boxname.to_s
      box.vm.network "private_network", ip: boxconfig[:ip]
      
      # Добавляем диск через Vagrant Disk (работает в Vagrant 2.2.0+)
      if boxname == :backup
        box.vm.disk :disk, size: "2GB", name: "backup_disk"
      end
      
      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
        v.name = boxname.to_s
      end
      
      box.vm.synced_folder ".", "/vagrant", disabled: true
      
      box.vm.provision "shell", inline: <<-SHELL
        sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
        systemctl restart sshd.service
      SHELL
      
      if boxname == :backup
        box.vm.provision "shell", inline: <<-SHELL
          sleep 10
          
          # ИСПРАВЛЕНИЕ: Скрипт выбирает только те диски, чей размер БОЛЬШЕ 500 Мегабайт (исключая sda и loop)
          # Это заставит Ubuntu полностью проигнорировать фантомный диск 10M
          VALID_DISKS=$(lsblk -dn -o NAME,SIZE -b | grep -v "sda" | grep -v "loop" | awk '$2 > 524288000 {print $1}')
          
          for disk in $VALID_DISKS; do
            full_disk="/dev/$disk"
            if [ -b "$full_disk" ]; then
              if ! blkid "${full_disk}1" >/dev/null 2>&1; then
                echo "Найден валидный диск: $full_disk. Размечаем..."
                parted -s "$full_disk" mklabel gpt
                parted -s "$full_disk" mkpart primary ext4 1MiB 100%
                udevadm settle
                mkfs.ext4 -F "${full_disk}1"
              else
                echo "Диск $full_disk уже размечен."
              fi
            fi
          done
          
          # Очищаем старые неверные записи в fstab, если они там были
          sed -i '/mnt\/disk/d' /etc/fstab
          
          # Монтируем первый найденный валидный диск
          FIRST_DISK=$(lsblk -dn -o NAME,SIZE -b | grep -v "sda" | grep -v "loop" | awk '$2 > 524288000 {print $1}' | head -n1)
          
          if [ -n "$FIRST_DISK" ]; then
            DISK_PATH="/dev/${FIRST_DISK}1"
            if [ -b "$DISK_PATH" ]; then
              mkdir -p /var/backup
              mount $DISK_PATH /var/backup
              echo "$DISK_PATH /var/backup ext4 defaults 0 0" >> /etc/fstab
              chown -R root:root /var/backup
              chmod 755 /var/backup
              
              echo "Диск $DISK_PATH успешно примонтирован в /var/backup"
              df -h /var/backup
            fi
          else
            echo "Валидный диск (>500MB) не найден. Доступные диски:"
            lsblk
          fi
        SHELL
      end
    end
  end
  
  config.vm.box_url = "file://C:/vagrant_projects/my_vm/log/ubuntu-jammy64.box"
  config.vm.boot_timeout = 600
end
# С помощью ansible учтанавливаем borgbackup, на сервере backup создаем пользователя borg
# Содержание inventory.ini
vagrant@ansible01:~/projects/backup$ cat inventory.ini
[backup_server]
192.168.56.16 ansible_user=vagrant

[client]
192.168.56.17 ansible_user=vagrant

[all:vars]
ansible_python_interpreter=/usr/bin/python3
# Содержание playbook.yml
- name: Установка BorgBackup
  hosts: all
  become: yes

  tasks:
    - name: Скачивание borgbackup
      get_url:
        url: https://github.com/borgbackup/borg/releases/download/1.2.8/borg-linux64
        dest: /usr/local/bin/borg
        mode: '0755'

- name: Настройка сервера
  hosts: 192.168.56.16
  become: yes

  tasks:
    - name: Создание пользователя borg
      user:
        name: borg
        state: present

    - name: Создание /var/backup
      file:
        path: /var/backup
        state: directory
        owner: borg
        group: borg
        mode: '0755'

    - name: Создание .ssh
      file:
        path: /home/borg/.ssh
        state: directory
        owner: borg
        group: borg
        mode: '0700'

    - name: Создание authorized_keys
      file:
        path: /home/borg/.ssh/authorized_keys
        state: touch
        owner: borg
        group: borg
        mode: '0600'

- name: Настройка клиента
  hosts: 192.168.56.17
  become: no

  tasks:
    - name: Генерация SSH ключа
      command: ssh-keygen -t rsa -b 4096 -f /home/vagrant/.ssh/id_rsa_borg -N ""
      args:
        creates: /home/vagrant/.ssh/id_rsa_borg

    - name: Копирование ключа через ssh-copy-id (с ручным вводом пароля)
      command: ssh-copy-id -i /home/vagrant/.ssh/id_rsa_borg.pub borg@192.168.56.16
      ignore_errors: yes
      register: copy_result
    - name: Если ssh-copy-id не сработал - копируем через vagrant пользователя
      shell: |
        cat /home/vagrant/.ssh/id_rsa_borg.pub | ssh vagrant@192.168.56.16 "sudo tee -a /home/borg/.ssh/authorized_keys && sudo chown borg:borg /home/borg/.ssh/authorized_keys && sudo chmod 600 /home/borg/.ssh/authorized_keys"
      when: copy_result is failed
# Генерация ключа на клиенте 

vagrant@backupclient:~$ ssh-keygen -t rsa -b 4096 -C "borg-backup-key"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/vagrant/.ssh/id_rsa
Your public key has been saved in /home/vagrant/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:MZe0jz5WfA6F/w5fD8NcmmK9aYJRp2Qy2v/Lrsy+Uao borg-backup-key
The key's randomart image is:
+---[RSA 4096]----+
|          .      |
|         . o .   |
|        o + . .  |
|         +o++o.  |
|        So.*=o+ .|
|        ..o..X = |
|          ++= @ o|
|         ..*o+.Oo|
|          E.*BB.+|
+----[SHA256]-----+
#  Копируем ключ на сервер
vagrant@backupclient:~$ ssh vagrant@192.168.56.16 "sudo mkdir -p /home/borg/.ssh && sudo tee -a /home/borg/.ssh/authorized_keys" < ~/.ssh/id_rsa.pub
# Проверяем подключение
vagrant@backupclient:~$ ssh borg@192.168.56.16
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-181-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Jul 26 09:17:06 UTC 2026

  System load:  0.0               Processes:               104
  Usage of /:   4.2% of 38.70GB   Users logged in:         1
  Memory usage: 13%               IPv4 address for enp0s3: 192.168.1.54
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
New release '24.04.4 LTS' available.
Run 'do-release-upgrade' to upgrade to it.
# С клиента инициализируем репозиторий borg
vagrant@backupclient:~$ borg init --encryption=repokey borg@192.168.56.16:/var/backup/
Enter new passphrase:
Enter same passphrase again:
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "cat"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.56.16/var/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).

# Запускаем для проверки создания бэкапа
vagrant@backupclient:~$ borg create --stats --list borg@192.168.56.16:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc

Repository: ssh://borg@192.168.56.16/var/backup
Archive name: etc-2026-07-26_09:46:04
Archive fingerprint: 30145100a2614a19fe41e3c73cc901f0892b993ded13c1a4618e09f662736076
Time (start): Sun, 2026-07-26 09:46:17
Time (end):   Sun, 2026-07-26 09:46:20
Duration: 3.51 seconds
Number of files: 678
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:                2.08 MB            914.16 kB            883.62 kB
All archives:                2.08 MB            913.54 kB            950.49 kB

                       Unique chunks         Total chunks
Chunk index:                     643                  671
-----------------------------------------------------------------------
# Смотрим, что у нас получилось
vagrant@backupclient:~$ borg list borg@192.168.56.16:/var/backup/
Enter passphrase for key ssh://borg@192.168.56.16/var/backup:
etc-2026-07-26_09:46:04              Sun, 2026-07-26 09:46:17 [30145100a2614a19fe41e3c73cc901f0892b993ded13c1a4618e09f662736076]
# Смотрим список файлов
vagrant@backupclient:~$ borg list borg@192.168.56.16:/var/backup/::etc-2026-07-26_09:46:04
# Достаем файл из бекапа
borg extract borg@192.168.56.16:/var/backup/::etc-2026-07-26_09:46:04 etc/hostname
# Автоматизируем создание бэкапов с помощью systemd
# Создаем сервис и таймер в каталоге /etc/systemd/system/
vagrant@backupclient:~$ sudo nano /etc/systemd/system/borg-backup.service

[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=cat"
Environment="BORG_RSH=ssh -i /home/vagrant/.ssh/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/home/vagrant/.ssh/known_hosts"
# Репозиторий
Environment=REPO=borg@192.168.56.16:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/usr/local/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/usr/local/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=/usr/local/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}


# /etc/systemd/system/borg-backup.timer
[Unit]
Description=Borg Backup

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target


# Создаем файл таймера
vagrant@backupclient:~$ sudo nano /etc/systemd/system/borg-backup.timer
vagrant@backupclient:~$ sudo cat /etc/systemd/system/borg-backup.timer [Unit]
Description=Borg Backup Timer
Requires=borg-backup.service

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
# Включаем и запускаем службу borg-backup.service
vagrant@backupclient:~$ sudo systemctl enable borg-backup.service
Created symlink /etc/systemd/system/timers.target.wants/borg-backup.service → /etc/systemd/system/borg-backup.service.
systemctl start borg-backup.timer
vagrant@backupclient:~$ sudo systemctl status borg-backup.service      ○ borg-backup.service - Borg Backup
     Loaded: loaded (/etc/systemd/system/borg-backup.service; enabled;>
     Active: inactive (dead) since Sun 2026-07-26 10:19:32 UTC; 2min 5>
    Process: 1419 ExecStart=/usr/local/bin/borg create --stats ${REPO}>
    Process: 1423 ExecStart=/usr/local/bin/borg check ${REPO} (code=ex>
    Process: 1428 ExecStart=/usr/local/bin/borg prune --keep-daily 90 >
   Main PID: 1428 (code=exited, status=0/SUCCESS)
        CPU: 6.452s

Jul 26 10:19:23 backupclient borg[1420]: ----------------------------->
Jul 26 10:19:23 backupclient borg[1420]:                        Origin>
Jul 26 10:19:23 backupclient borg[1420]: This archive:                >
Jul 26 10:19:23 backupclient borg[1420]: All archives:                >
Jul 26 10:19:23 backupclient borg[1420]:                        Unique>
Jul 26 10:19:23 backupclient borg[1420]: Chunk index:                 >
Jul 26 10:19:23 backupclient borg[1420]: ----------------------------->
Jul 26 10:19:32 backupclient systemd[1]: borg-backup.service: Deactiva>
Jul 26 10:19:32 backupclient systemd[1]: Finished Borg Backup.
Jul 26 10:19:32 backupclient systemd[1]: borg-backup.service: Consumed>

# Включаем и запускаем службу таймера
vagrant@backupclient:~$ sudo systemctl daemon-reload
vagrant@backupclient:~$ sudo systemctl enable borg-backup.timer
vagrant@backupclient:~$ systemctl list-timers --all
NEXT                        LEFT          LAST                        PASSED            UNIT                           ACTIVATES
Sun 2026-07-26 10:40:57 UTC 4min 18s left n/a                         n/a               borg-backup.timer              borg-backup.service
# Проверяем, что бэкапы создаются
vagrant@backupclient:~$ borg list borg@192.168.56.16:/var/backup/
Enter passphrase for key ssh://borg@192.168.56.16/var/backup:
etc-2026-07-26_09:46:04              Sun, 2026-07-26 09:46:17 [30145100a2614a19fe41e3c73cc901f0892b993ded13c1a4618e09f662736076]
etc-2026-07-26_10:51:39              Sun, 2026-07-26 10:51:42 [dcd274a354e3f722bbe2de049ac3c43e0094941995b39f09c16b1a243c708e54]
# Проверка логов бэкапа 
vagrant@backupclient:~$ sudo journalctl -u borg-backup.service -f
Jul 26 10:51:43 backupclient borg[1765]: ------------------------------------------------------------------------------
Jul 26 10:51:43 backupclient borg[1765]:                        Original size      Compressed size    Deduplicated size
Jul 26 10:51:43 backupclient borg[1765]: This archive:                2.11 MB            929.66 kB                619 B
Jul 26 10:51:43 backupclient borg[1765]: All archives:                6.29 MB              2.77 MB              1.04 MB
Jul 26 10:51:43 backupclient borg[1765]:                        Unique chunks         Total chunks
Jul 26 10:51:43 backupclient borg[1765]: Chunk index:                     671                 2061
Jul 26 10:51:43 backupclient borg[1765]: ------------------------------------------------------------------------------
Jul 26 10:51:52 backupclient systemd[1]: borg-backup.service: Deactivated successfully.
Jul 26 10:51:52 backupclient systemd[1]: Finished Borg Backup.
Jul 26 10:51:52 backupclient systemd[1]: borg-backup.service: Consumed 6.072s CPU time.
# Логирование через syslog/logger
# Изменим файл службы
vagrant@backupclient:~$ sudo nano /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=cat"
Environment="BORG_RSH=ssh -i /home/vagrant/.ssh/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/home/vagrant/.ssh/known_hosts"
# Репозиторий
Environment=REPO=borg@192.168.56.16:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/bin/bash -c "/usr/local/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET} \
    2>&1 | logger -t borg-backup -p info"

# Проверка бэкапа
ExecStart=/bin/bash -c "/usr/local/bin/borg check ${REPO} \
    2>&1 | logger -t borg-backup -p info"

# Очистка старых бэкапов
ExecStart=/bin/bash -c "/usr/local/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO} \
    2>&1 | logger -t borg-backup -p info"

[Install]
WantedBy=timers.target

# Просмотр логов 
vagrant@backupclient:~$ sudo grep borg-backup /var/log/syslog
Jul 26 11:16:41 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:16:41 backupclient borg-backup: Repository: ssh://borg@192.168.56.16/var/backup
Jul 26 11:16:41 backupclient borg-backup: Archive name: etc-2026-07-26_11:16:36
Jul 26 11:16:41 backupclient borg-backup: Archive fingerprint: 32a7bd5ddce553a9a1755f2de1c773a1fda7e52495a1a431845a40ad3f71a709
Jul 26 11:16:41 backupclient borg-backup: Time (start): Sun, 2026-07-26 11:16:40
Jul 26 11:16:41 backupclient borg-backup: Time (end):   Sun, 2026-07-26 11:16:40
Jul 26 11:16:41 backupclient borg-backup: Duration: 0.47 seconds
Jul 26 11:16:41 backupclient borg-backup: Number of files: 704
Jul 26 11:16:41 backupclient borg-backup: Utilization of max. archive size: 0%
Jul 26 11:16:41 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:16:41 backupclient borg-backup:                        Original size      Compressed size    Deduplicated size
Jul 26 11:16:41 backupclient borg-backup: This archive:                2.11 MB            929.65 kB              1.33 kB
Jul 26 11:16:41 backupclient borg-backup: All archives:                6.29 MB              2.77 MB              1.11 MB
Jul 26 11:16:41 backupclient borg-backup:
Jul 26 11:16:41 backupclient borg-backup:                        Unique chunks         Total chunks
Jul 26 11:16:41 backupclient borg-backup: Chunk index:                     674                 2061
Jul 26 11:16:41 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:16:50 backupclient systemd[1]: borg-backup.service: Deactivated successfully.
Jul 26 11:16:50 backupclient systemd[1]: borg-backup.service: Consumed 6.330s CPU time.
Jul 26 11:21:43 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:21:43 backupclient borg-backup: Repository: ssh://borg@192.168.56.16/var/backup
Jul 26 11:21:43 backupclient borg-backup: Archive name: etc-2026-07-26_11:21:39
Jul 26 11:21:43 backupclient borg-backup: Archive fingerprint: f3fb9b65d8207a66d59f9cef85d1f1f660f256d97591eb489d4eb67d9ccc02bd
Jul 26 11:21:43 backupclient borg-backup: Time (start): Sun, 2026-07-26 11:21:42
Jul 26 11:21:43 backupclient borg-backup: Time (end):   Sun, 2026-07-26 11:21:43
Jul 26 11:21:43 backupclient borg-backup: Duration: 0.25 seconds
Jul 26 11:21:43 backupclient borg-backup: Number of files: 704
Jul 26 11:21:43 backupclient borg-backup: Utilization of max. archive size: 0%
Jul 26 11:21:43 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:21:43 backupclient borg-backup:                        Original size      Compressed size    Deduplicated size
Jul 26 11:21:43 backupclient borg-backup: This archive:                2.11 MB            929.65 kB                619 B
Jul 26 11:21:43 backupclient borg-backup: All archives:                6.29 MB              2.77 MB              1.04 MB
Jul 26 11:21:43 backupclient borg-backup:
Jul 26 11:21:43 backupclient borg-backup:                        Unique chunks         Total chunks
Jul 26 11:21:43 backupclient borg-backup: Chunk index:                     671                 2061
Jul 26 11:21:43 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:21:52 backupclient systemd[1]: borg-backup.service: Deactivated successfully.
Jul 26 11:21:52 backupclient systemd[1]: borg-backup.service: Consumed 6.023s CPU time.
Jul 26 11:27:07 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:27:07 backupclient borg-backup: Repository: ssh://borg@192.168.56.16/var/backup
Jul 26 11:27:07 backupclient borg-backup: Archive name: etc-2026-07-26_11:27:03
Jul 26 11:27:07 backupclient borg-backup: Archive fingerprint: d23f34110da4345f6a7aabfcee3600d5abcda5c1908c76d1de9106d600cc8638
Jul 26 11:27:07 backupclient borg-backup: Time (start): Sun, 2026-07-26 11:27:06
Jul 26 11:27:07 backupclient borg-backup: Time (end):   Sun, 2026-07-26 11:27:07
Jul 26 11:27:07 backupclient borg-backup: Duration: 0.40 seconds
Jul 26 11:27:07 backupclient borg-backup: Number of files: 704
Jul 26 11:27:07 backupclient borg-backup: Utilization of max. archive size: 0%
Jul 26 11:27:07 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:27:07 backupclient borg-backup:                        Original size      Compressed size    Deduplicated size
Jul 26 11:27:07 backupclient borg-backup: This archive:                2.11 MB            929.65 kB                618 B
Jul 26 11:27:07 backupclient borg-backup: All archives:                6.29 MB              2.77 MB              1.04 MB
Jul 26 11:27:07 backupclient borg-backup:
Jul 26 11:27:07 backupclient borg-backup:                        Unique chunks         Total chunks
Jul 26 11:27:07 backupclient borg-backup: Chunk index:                     671                 2061
Jul 26 11:27:07 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:27:16 backupclient systemd[1]: borg-backup.service: Deactivated successfully.
Jul 26 11:27:16 backupclient systemd[1]: borg-backup.service: Consumed 6.589s CPU time.
Jul 26 11:32:46 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:32:46 backupclient borg-backup: Repository: ssh://borg@192.168.56.16/var/backup
Jul 26 11:32:46 backupclient borg-backup: Archive name: etc-2026-07-26_11:32:43
Jul 26 11:32:46 backupclient borg-backup: Archive fingerprint: 26d401c8bd08947080b1974ac6144c0900d28391a60fcbd599b8762d15266d87
Jul 26 11:32:46 backupclient borg-backup: Time (start): Sun, 2026-07-26 11:32:46
Jul 26 11:32:46 backupclient borg-backup: Time (end):   Sun, 2026-07-26 11:32:46
Jul 26 11:32:46 backupclient borg-backup: Duration: 0.36 seconds
Jul 26 11:32:46 backupclient borg-backup: Number of files: 704
Jul 26 11:32:46 backupclient borg-backup: Utilization of max. archive size: 0%
Jul 26 11:32:46 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:32:46 backupclient borg-backup:                        Original size      Compressed size    Deduplicated size
Jul 26 11:32:46 backupclient borg-backup: This archive:                2.11 MB            929.65 kB                616 B
Jul 26 11:32:46 backupclient borg-backup: All archives:                6.29 MB              2.77 MB              1.04 MB
Jul 26 11:32:46 backupclient borg-backup:
Jul 26 11:32:46 backupclient borg-backup:                        Unique chunks         Total chunks
Jul 26 11:32:46 backupclient borg-backup: Chunk index:                     671                 2061
Jul 26 11:32:46 backupclient borg-backup: ------------------------------------------------------------------------------
Jul 26 11:32:55 backupclient systemd[1]: borg-backup.service: Deactivated successfully.
Jul 26 11:32:55 backupclient systemd[1]: borg-backup.service: Consumed 5.973s CPU time.






    

