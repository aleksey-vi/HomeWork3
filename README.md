# HomeWork3
## otus_3

## lvm homework

На имеющемся образе centos/7 - v. 1804.21 (приложенный Vagrantfile)

   Уменьшить том под / до 8G
   Выделить том под /home
   Выделить том под /var - сделать в mirror
   /home - сделать том для снапшотов
   Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами ( на выбор)

Работа со снапшотами:- сгенерить файлы в /home/

   снять снапшот
   удалить часть файлов
   восстановится со снапшота Залоггировать работу можно с помощью утилиты script


Разворачиваем Vagrantfile, подключаемся к машине lvm

        vagrant up
        vagrant ssh

На vm устанавливаем утилиты, кот. помогут в дальнейшей работе

        sudo yum install xfsdump vim

Смотрим список дисков и их разделов сразу после старта vm

        lsblk

Подготовим временный том для / раздела

        pvcreate /dev/sdb
        vgcreate vg_root /dev/sdb
        lvcreate -n lv_root -l +100%FREE /dev/vg_root

Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

        mkfs.xfs /dev/vg_root/lv_root
        mount /dev/vg_root/lv_root /mnt

Image alt Этой командой скопируем все данные с / раздела в /mnt - итог вывода на скрине:

        xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

Проверим, что скопировалось

        ls /mnt

Переконфигурируем grub для того, чтобы при старте перейти в новый / Сымитируем текущий root -> сделаем в него chroot и обновим grub:

        for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
        chroot /mnt/
        grub2-mkconfig -o /boot/grub2/grub.cfg

Обновим образ initrd.

        cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

Чтобы при загрузке был смонтирован нужный root: в файле /boot/grub2/grub.cfg заменим rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root. Для редактирования я использовала vim, установленный в самом начале.

        nano /etc/default/grub

После изменений - обновила grub

        grub2-mkconfig -o /boot/grub2/grub.cfg

Выход, перезагрузка. После подключения можно проверить, что успешно загрузились с новым рут томом

        lsblk

Теперь изменяем размер старой VG и возвращаем на него рут. Для этого удаляем старый LV размеров в 40G и создаем новый на 8G:

        lvremove /dev/VolGroup00/LogVol00
        lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

Создаём файловую систему

        mkfs.xfs /dev/VolGroup00/LogVol00

Монтируем файловую систему, копируем данные

        mount /dev/VolGroup00/LogVol00 /mnt
        xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

Переконфигурируем grub, за исключением правки /etc/grub2/grub.cfg

        for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
        chroot /mnt/
        grub2-mkconfig -o /boot/grub2/grub.cfg
        cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

Пока не перезагружаемся и не выходим из под chroot - мы можем заодно перенести /var На свободных дисках создаем зеркало:

        pvcreate /dev/sdc /dev/sdd
        vgcreate vg_var /dev/sdc /dev/sdd
        lvcreate -L 950M -m1 -n lv_var vg_var

Создаем на нем ФС и перемещаем туда /var:

        mkfs.ext4 /dev/vg_var/lv_var
        mount /dev/vg_var/lv_var /mnt
        cp -aR /var/* /mnt/      # rsync -avHPSAX /var/ /mnt/

На всякий случай сохраняем содержимое старого var:

        mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

Монтируем новый var в каталог /var:

        umount /mnt
        mount /dev/vg_var/lv_var /var

Правим fstab для автоматического монтирования /var:

        echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

Внесём правки в grub, заменив rd.lvm.lv=vg_root/lv_root на rd.lvm.lv=VolGroup00/LogVol00

        grub2-mkconfig -o /boot/grub2/grub.cfg
        nano /etc/default/grub

После изменений - обновим grub

        grub2-mkconfig -o /boot/grub2/grub.cfg

Перезагружаемся в новый (уменьшенный root), удаляем временную Volume Group:

        lvremove /dev/vg_root/lv_root
        vgremove /dev/vg_root
        pvremove /dev/sdb

Выделяем том под /home по тому же принципу что делали для /var:

        lvcreate -n LogVol_Home -L 2G /dev/VolGroup00 
        mkfs.xfs /dev/VolGroup00/LogVol_Home
        mount /dev/VolGroup00/LogVol_Home /mnt/
        cp -aR /home/* /mnt/
        rm -rf /home/*
        umount /mnt
        mount /dev/VolGroup00/LogVol_Home /home/

Правим fstab для автоматического монтирования /home

        echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

Сгенерируем файлы в /home/:

        touch /home/file{1..20}

Снять снапшот:

        lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

Удалить часть файлов:

        rm -f /home/file{11..20}

Процесс восстановления со снапшота:

        lvconvert --merge /dev/VolGroup00/home_snap
        mount /home

Выход, перезагрузка, проверяем, что всё в норме lsblk
        
