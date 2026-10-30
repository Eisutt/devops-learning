server-setup
Создал публичный репазиторий на gihub, связал его с моим ПК. 

echo "# devops-learning1" >> README.md # Создал README файл 
git init # проинициализировал директорию
git add README.md # Добавил файл 
git commit -m "first commit" # сделал первый коммит
git branch -M main # перешел в ветку main
git remote add origin https://github.com/Eisutt/devops-learning.git # связал с репазиторием на github
git push -u origin main # сделал пуш в корневую ветку на удаленный репазиторий

Через vscode и расширение source control сделал коммит и пуш репазитория на сервере 

Создал ВМ на Yandex cloud со следующими характеристиками: 

Платформа 
Intel Ice Lake
Гарантированная доля vCPU 
100%
​vCPU
2
RAM
2 ГиБ
Объём дискового пространства
20 ГиБ

SSH ключ для яндекса у меня уже был создан, при создании ВМ просто добавил публичный ключ в поле 

eisutt@devops-01:~$ whoami # показывает под каким пользователем сейчас работаем
eisutt

eisutt@devops-01:~$ hostnamectl # показывает информацию о машине
 Static hostname: devops-01
       Icon name: computer-vm
         Chassis: vm 🖴
      Machine ID: 23000007c6d8f0f3625fd7802fdc166f
         Boot ID: 426e6594ab1a4edd8564d8b5754441a5
  Virtualization: kvm
Operating System: Ubuntu 24.04.5 LTS
          Kernel: Linux 6.8.0-147-generic
    Architecture: x86-64
 Hardware Vendor: Yandex
  Hardware Model: xeon-gold-6338
Firmware Version: 1.16.3-1
   Firmware Date: Tue 2014-04-01
    Firmware Age: 12y 6month 4w 1d

eisutt@devops-01:~$ ip -br a # просмотрел сетевой адаптер и какой айпи получаем (внутренний)
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             10.128.0.12/24 metric 100 fe80::d20d:18ff:fef0:f362/64

eisutt@devops-01:~$ df -h # смотрим объем памяти на диске
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.1M  196M   1% /run
/dev/vda1        19G  2.2G   17G  12% /
tmpfs           984M     0  984M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/vda15      599M  6.2M  593M   2% /boot/efi
tmpfs           197M  8.0K  197M   1% /run/user/1000

eisutt@devops-01:~$ free -h # смотрим размер ОЗУ
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       302Mi       1.1Gi       1.0Mi       656Mi       1.6Gi
Swap:             0B          0B          0B

В принципе проблем на первом шаге не было, все более менее знакомо. Вспомнил как создавать и связывать репазиторий. Вспомнил команды для просмотра информации