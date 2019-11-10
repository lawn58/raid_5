# raid_5
 
   <<Vagrant-file>>
   
В репозитории присутствует Vagrant-файл, с помощью которого можно собрать любой рейд,
для каждого дополнительного диска необходимо добавить в Vagrant-файл следующий блок: 
:sata5 => {
 :dfile => './sata5.vdi', # Путь, по которому будет создан файл диска
 :size => 250, # Размер диска в мегабайтах
 :port => 5 # Номер порта на который будет зацеплен диск
},

Обязательно увеличив номер порта и изменив имя файла диска, чтобы
исключить дублирование.

   <<Создание raid-a>>
   
В vagrant-файл подразумевается, что мы собираем raid из 5 дисков,
просмотреть кол-во какие блочные устройства присутствуют у нас в системе
можно с помощью команд:
- fdisk -l
- lsblk
- lshw
- lsscsi

Заполним суперблоки нулями с помощью команды:
mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
И можно создавать рейд следующей командой:
mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}

- Мы выбрали RAID 5. Опция "-l" - какого уровния RAID создавать.
- Опция "-n" указывает на кол-во устройств в RAID.

Проверим что RAID собрался нормально:
cat /proc/mdstat
mdadm -D /dev/md0

   <<Создание конфигурационного файла mdadm.conf>>


Для того, чтобы быть уверенным что ОС запомнила какой RAID-массив
требуется создать и какие компоненты в него входят, создадим файл
mdadm.conf
Сначала убедимся, что информация верна:
mdadm --detail --scan --verbose

А затем создадим файл mdadm.conf

sudo mkdir -p /etc/mdadm (если такой папки нет, у вас в системе)
sudo touch /etc/mdadm/mdadm.conf
sudo mdadm --detail --scan --verbose|awk /ARRAY/{print} >> /etc/mdadm/mdadm.conf
возможен такой вариант: sudo mdadm --detail --scan --verbose|awk '/ARRAY/{print}' >> /etc/mdadm/mdadm.conf

   <<Сломать/починить RAID>>

Сделать это можно, например, искусственно "зафейлив" одно из блочных устройств командой:
mdadm /dev/md0 --fail /dev/sde

Посмотрим как это отразилось на RAID:

cat /proc/mdstat
mdadm -D /dev/md0

Удалим "сломанный" диск из массива:

 mdadm /dev/md0 --remove /dev/sde
 
 Представим, что мы вставили новый диск в сервер и теперь нам нужно добавить его в RAID.
 Делается это так:
 
 mdadm /dev/md0 --add /dev/sde
 mdadm -D /dev/md0
 
   <<Создать GPT раздел, пять партиция и смонтировать их на диск>>
   
   Создаем раздел GPT на RAID:
   
   parted -s /dev/md0 mklabel gpt
   
   Создаем партиции:
   
   parted /dev/md0 mkpart primary ext4 0% 20%
   parted /dev/md0 mkpart primary ext4 20% 40%
   parted /dev/md0 mkpart primary ext4 40% 60%
   parted /dev/md0 mkpart primary ext4 60% 80%
   parted /dev/md0 mkpart primary ext4 80% 100%
   
   Далее, можно создать на этих партициях ФС:
   
   for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
   
   И смонтировать их по каталогам:
   
   mkdir -p /raid/part{1,2,3,4,5}
   for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
   

