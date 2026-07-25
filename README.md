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

