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
