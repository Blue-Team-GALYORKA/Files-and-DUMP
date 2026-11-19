# Практическая работа №5. Настройка и конфигурация файловых систем Linux и прав доступа (Debian 11)

## Цель работы
- Научиться работать с Linux-системами.
- Изучить настройки и конфигурационные файлы.
- Научиться настраивать права доступа.

## Задача
Развернуть стенд из двух ВМ Debian 11, сделать snapshot на момент установки, выполнить
все пункты задания. Все команды — через консоль, ВМ без графического интерфейса.

> 📷 СКРИНШОТ: каждый пункт задания.

---

## Ход работы

### 1. Установить Debian 11
Установить Debian 11 (netinst) в VMware без графической оболочки (снять галки с desktop
environment при установке tasksel, оставить «standard system utilities» и «SSH server»).

### 2. Создать snapshot
В VMware: **VM → Snapshot → Take Snapshot** — зафиксировать чистое состояние после установки.
📷 СКРИНШОТ меню снапшота.

### 3. Включить пользователя в группу sudo
```bash
su -                       # войти под root
usermod -aG sudo student   # добавить пользователя в группу sudo
# перелогиниться, проверить:
groups student             # должна присутствовать группа sudo
sudo whoami                # → root
```

### 4–5. Выделить место под новые HDD и создать два диска
В VMware выключить ВМ → **Edit settings → Add → Hard Disk** → создать два новых диска
(например по 1 ГБ). После загрузки проверить:
```bash
lsblk                      # появятся /dev/sdb и /dev/sdc
```
Разметить и создать файловые системы:
```bash
sudo fdisk /dev/sdb        # n (new) → p → w (write); аналогично /dev/sdc
sudo mkfs.ext4 /dev/sdb1
sudo mkfs.ext4 /dev/sdc1
```

### 6. Создать директории Data и bigData
```bash
sudo mkdir /Data /bigData
```

### 7–8. Примонтировать диски
```bash
# первый диск — только для чтения, к /bigData
sudo mount -o ro /dev/sdb1 /bigData
# второй диск — стандартно (чтение-запись), к /Data
sudo mount /dev/sdc1 /Data
```
Проверка:
```bash
mount | grep -E "sdb1|sdc1"
df -h | grep -E "bigData|Data"
```
Диск на `/bigData` смонтирован `ro` — попытка записи даст `Read-only file system`.
📷 СКРИНШОТ монтирования.

### 9. Примонтировать одну папку в другую (--bind)
```bash
sudo mkdir /Data/mirror
sudo mount --bind /bigData /Data/mirror
```
`--bind` делает содержимое `/bigData` доступным также по пути `/Data/mirror`.

### 10. Создать группу PT
```bash
sudo groupadd PT
```

### 11–12. Создать папку 23 в Data и назначить ей группу PT
```bash
sudo mkdir /Data/23
sudo chgrp PT /Data/23
ls -ld /Data/23            # группа должна быть PT
```

### 13–15. Создать пользователя, пароль, включить в группу PT
```bash
sudo useradd -m ptuser
sudo passwd ptuser
sudo usermod -aG PT ptuser
groups ptuser              # должна быть группа PT
```

### 16. Проверить домашний каталог
```bash
ls -ld /home/ptuser        # создан благодаря -m при useradd
```

### 17. Запретить доступ ptuser к /Data (чтобы ls давал permission denied)
```bash
sudo chmod 770 /Data              # rwx для владельца и группы, 0 для остальных
sudo chown root:root /Data        # владелец root, группа root
```
Теперь ptuser (не входит в группу-владельца root) при `ls /Data` получит:
```
ls: cannot open directory '/Data': Permission denied
```
**Пояснение:** права `770` означают, что «остальные» (other) не имеют ни чтения (r), ни
входа (x) в каталог. ptuser не владелец и не в группе root → попадает в категорию other →
доступ запрещён.
📷 СКРИНШОТ permission denied.

### 18. Залогиниться под ptuser и проверить п.17
```bash
su - ptuser
ls /Data                   # → Permission denied
exit
```
📷 СКРИНШОТ проверки под новым пользователем.

### 19. Установить SSH-сервер со сменой порта и защитой ключами
```bash
sudo apt install openssh-server
sudo nano /etc/ssh/sshd_config
```
Изменить:
```
Port 2222
PermitRootLogin no
PubkeyAuthentication yes
```
```bash
sudo systemctl restart ssh
```
Настроить вход по ключам (подробно — ПР 2.2 и 2.3):
```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id -p 2222 student@<IP-сервера>
```

### 20. Подключиться из Windows через PuTTY
PuTTY → Host `<IP>`, Port `2222`, SSH → Open → войти. (Ключ при необходимости
конвертировать в .ppk через PuTTYgen.)
📷 СКРИНШОТ подключения PuTTY.

### 21. Проверить сохранение после перезапуска; прислать скрин fstab
Чтобы монтирование сохранялось после перезагрузки, прописать диски в `/etc/fstab`:
```bash
sudo blkid                 # узнать UUID дисков
sudo nano /etc/fstab
```
Добавить строки (UUID подставить свои):
```
UUID=xxxx-first   /bigData   ext4   ro,defaults   0  2
UUID=xxxx-second  /Data      ext4   defaults      0  2
/bigData          /Data/mirror  none  bind         0  0
```
Проверить без перезагрузки:
```bash
sudo umount /bigData /Data 2>/dev/null
sudo mount -a              # смонтировать всё из fstab; ошибок быть не должно
```
Перезагрузить и убедиться, что всё примонтировано:
```bash
sudo reboot
# после загрузки:
df -h
cat /etc/fstab
```
📷 СКРИНШОТ конфигурационного файла fstab и вывода df -h после перезапуска.

---

## Контрольные вопросы
- **Как смонтировать диск только для чтения?** — `mount -o ro устройство точка`.
- **Что делает `mount --bind`?** — отображает содержимое одного каталога по второму пути.
- **Что означают права 770?** — rwx владельцу и группе, остальным — ничего.
- **Почему ls даёт Permission denied?** — у пользователя нет бита x (входа) на каталог.
- **Зачем нужен /etc/fstab?** — автоматическое монтирование ФС при загрузке.
- **Как применить fstab без перезагрузки?** — `sudo mount -a`.

## Вывод
Развёрнут стенд Debian 11, добавлены два HDD и смонтированы (один — только для чтения),
выполнено bind-монтирование, созданы группа PT, пользователь и каталоги с нужными
правами. Отработаны права доступа (включая намеренный Permission denied через `chmod 770`),
настроен SSH по ключам на нестандартном порту и закреплено монтирование через `/etc/fstab`.
