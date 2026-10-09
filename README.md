# Отчёт по лабораторной работе: Установка Arch Linux на программный RAID1 с загрузчиком UEFI

## 1. Введение и конфигурация окружения

Целью данной лабораторной работы является ручная установка операционной системы Arch Linux на программный массив RAID1 (зеркалирование) с обеспечением отказоустойчивости загрузки в режиме UEFI.

### Конфигурация виртуальной машины (VirtualBox)

* **Операционная система:** Arch Linux (x86_64)
* **Накопители:** 2 виртуальных диска SATA (`/dev/sda` объемом 20 ГБ и `/dev/sdb` объемом 20 ГБ)
* **Первоначальный режим загрузки:** Legacy BIOS (SeaBIOS)
* **Итоговый режим загрузки:** UEFI

---

## 2. Подготовка и проверка режима Legacy BIOS (SeaBIOS)

Согласно условиям задания, первоначальный запуск установочного диска выполняется в режиме Legacy BIOS (SeaBIOS).

### Проверка режима загрузки в установочном окружении

Для подтверждения загрузки в режиме BIOS в консоли установочного медиа выполняется проверка наличия каталога EFI:

```bash
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"
```

**Результат:**
```text
BIOS
```
Вывод слова `BIOS` подтверждает, что система запущена в режиме SeaBIOS.

---

## 3. Разметка дисков и создание массива RAID1

Прошивка материнской платы не способна считывать загрузочные разделы ESP с метаданными `mdraid`, поэтому EFI-разделы создаются локально на обоих дисках вне массива RAID, а корневая файловая система размещается на программном массиве RAID1 (`/dev/md0`).

### 3.1. Создание таблицы разделов GPT

На обоих дисках (`/dev/sda` и `/dev/sdb`) создается идентичная разметка GPT с помощью утилиты `sgdisk`:

```bash
# Очистка и создание разделов на первом диске
sgdisk -Z /dev/sda
sgdisk -n 1:2048:+1M    -t 1:ef02 -c 1:"BIOSBOOT" /dev/sda
sgdisk -n 2:0:+512M     -t 2:ef00 -c 2:"EFISYS"   /dev/sda
sgdisk -n 3:0:0         -t 3:fd00 -c 3:"RAID1"    /dev/sda

# Клонирование таблицы разделов на второй диск
sgdisk -R /dev/sdb /dev/sda
sgdisk -G /dev/sdb
```

**Структура разделов:**
* `/dev/sda1` и `/dev/sdb1`: Разделы BIOS Boot (`ef02`, 1 МБ)
* `/dev/sda2` и `/dev/sdb2`: Разделы ESP (`ef00`, 512 МБ)
* `/dev/sda3` и `/dev/sdb3`: Разделы под программный RAID1 (`fd00`, остаток диска)

### 3.2. Создание и форматирование RAID1 (`mdadm`)

Создание программного зеркального массива из разделов `/dev/sda3` и `/dev/sdb3`:

```bash
# Создание массива md0 с версией метаданных 1.2
mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sda3 /dev/sdb3

# Форматирование ESP-разделов в FAT32 (на обоих дисках отдельно)
mkfs.vfat -F32 /dev/sda2
mkfs.vfat -F32 /dev/sdb2

# Форматирование корневого массива RAID1 в ext4
mkfs.ext4 /dev/md0
```

Проверка состояния созданного массива:

```bash
mdadm --detail /dev/md0
```

**Результат:**
```text
/dev/md0:
           Version : 1.2
     Creation Time : Mon Oct  5 16:22:45 2026
        Raid Level : raid1
        Array Size : 20427712 (19.48 GiB 20.92 GB)
     Used Dev Size : 20427712 (19.48 GiB 20.92 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Tue Oct  6 15:50:31 2026
             State : clean 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : bitmap

              Name : archiso:0
              UUID : 86e38e82:beedaec2:00a70d94:a5bcae3a
            Events : 22

    Number   Major   Minor   RaidDevice State
       0       8        3        0      active sync   /dev/sda3
       1       8       19        1      active sync   /dev/sdb3
```

---

## 4. Ручная установка системы (pacstrap, genfstab, arch-chroot)

Установка системы производится строго вручную без использования автоустановщика `archinstall`.

### 4.1. Монтирование файловых систем

```bash
# Монтирование корневого массива
mount /dev/md0 /mnt

# Монтирование первого ESP-раздела
mkdir -p /mnt/boot
mount /dev/sda2 /mnt/boot
```

### 4.2. Установка базовых пакетов и генерация fstab

```bash
# Установка базового окружения, ядра и утилит управления RAID
pacstrap -K /mnt base linux linux-firmware mdadm e2fsprogs efibootmgr grub nano

# Генерация файла таблицы монтирования по UUID
genfstab -U /mnt >> /mnt/etc/fstab
```

### 4.3. Настройка конфигурации `mdadm.conf` и initramfs

```bash
# Переход в окружение новой системы
arch-chroot /mnt

# Сохранение конфигурации RAID-массива
mdadm --detail --scan >> /etc/mdadm.conf
```

В файле `/etc/mkinitcpio.conf` в секцию `HOOKS` перед `filesystems` добавляется хук `mdadm_udev`:

```bash
HOOKS=(base udev autodetect microcode modprobes block mdadm_udev filesystems keyboard fsck)
```

Пересборка образа initramfs:

```bash
mkinitcpio -P
```

---

## 5. Настройка загрузчика GRUB и UEFI на обоих дисках

Так как установка проводилась из-под режима SeaBIOS (Legacy), прямое создание переменных NVRAM через `efibootmgr` недоступно внутри chroot. По этой причине GRUB устанавливается с флагом `--no-nvram`, а создание записей выполняется после переключения VirtualBox в режим UEFI.

### 5.1. Установка GRUB на первый и второй ESP-разделы

```bash
# Установка GRUB на первый диск (/dev/sda2, смонтирован в /boot)
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id="GRUB" --no-nvram
grub-mkconfig -o /boot/grub/grub.cfg

# Монтирование второго ESP-раздела и копирование файлов загрузчика
mkdir -p /mnt/boot2
mount /dev/sdb2 /mnt/boot2
cp -a /boot/* /mnt/boot2/
umount /mnt/boot2
```

Завершение работы в chroot-окружении и выключение виртуальной машины:

```bash
exit
umount -R /mnt
poweroff
```

---

## 6. Переключение в режим UEFI и регистрация записей загрузки

1. В настройках виртуальной машины VirtualBox (**Настройки** -> **Система**) включена опция **«Включить EFI»**.
2. Осуществлён запуск виртуальной машины.

### Регистрация дублированных записей в NVRAM с помощью `efibootmgr`

После успешной загрузки в режиме UEFI в консоли выполняются команды добавления независимых записей загрузки для дисков `sda` и `sdb`:

```bash
efibootmgr --create --disk /dev/sda --part 2 --label "Arch Linux (sda)" --loader '\EFI\GRUB\grubx64.efi'
efibootmgr --create --disk /dev/sdb --part 2 --label "Arch Linux (sdb)" --loader '\EFI\GRUB\grubx64.efi'
```

Проверка конфигурации загрузчика:

```bash
efibootmgr -v
```

*Демонстрация этапа:* **`part1.gif`**

**Результат (вывод `efibootmgr -v`):**
```text
BootCurrent: 0002
Timeout: 0 seconds
BootOrder: 0001,0004,0000,0002,0003
Boot0000* UiApp FvVol(7cb8bdc9-f8eb-4f34-aaea-3ee4af6516a1)/FvFile(462caa21-7614-4503-836e-8ab6f4662331)
Boot0001* Arch Linux (sdb)	HD(2,GPT,4245d2fe-2f35-4a9a-b63d-b7207efbc339,0x1000,0x100000)/\EFI\GRUB\grubx64.efi
Boot0002* UEFI VBOX HARDDISK VBb19cb6e8-e2826861 ...
Boot0003* UEFI VBOX HARDDISK VB2f9caedb-f71d76eb ...
Boot0004* Arch Linux (sda)	HD(2,GPT,4b1c8209-31d9-4b8b-9388-2717838cf5f3,0x1000,0x100000)/\EFI\GRUB\grubx64.efi
```
Вывод подтверждает наличие двух самостоятельных EFI-записей для `sda` (`Boot0004`) и `sdb` (`Boot0001`), ссылающихся на соответствующие ESP-разделы (`HD(2,GPT,...)`).

---

## 7. Демонстрация отказоустойчивости RAID1

Для выполнения ключевого требования сдачи проведена проверка работы системы при полном физическом отключении первого диска и последующей ресинхронизации массива.

### 7.1. Отключение первого диска в VirtualBox и запуск в режиме `degraded`

1. Виртуальная машина выключена (`poweroff`).
2. В настройках VirtualBox (**Настройки** -> **Носители** -> **Контроллер: SATA**) накопитель `sda.vdi` был извлечён/отключен.
3. Осуществлён запуск виртуальной машины с единственного оставшегося накопителя `sdb.vdi` в режиме UEFI (через загрузочную запись `Arch Linux (sdb)`).

После успешного входа в систему проверен статус RAID1-массива:

```bash
cat /proc/mdstat
```

*Демонстрация этапа:* **`part2.gif`**

**Результат:**
```text
Personalities : [raid1] 
md0 : active raid1 sdb3[1]
      20427712 blocks super 1.2 [2/1] [_U]
      bitmap: 1/1 pages [4KB], 65536KB chunk

unused devices: <none>
```
Массив перешёл в состояние `degraded` с индикатором статуса `[_U]`. Операционная система и загрузчик UEFI успешно загрузились с единственного оставшегося диска `sdb`, подтвердив полномерную отказоустойчивость.

### 7.2. Возврат диска, восстановление массива и ресинхронизация (`recovery`)

1. Виртуальная машина выключена, накопитель `sda.vdi` подключен обратно в SATA-контроллер VirtualBox.
2. После запуска системы выполнен возврат раздела `/dev/sda3` в состав RAID1-массива:

```bash
mdadm /dev/md0 --add /dev/sda3
cat /proc/mdstat
```

*Демонстрация этапа:* **`part3.gif`**

**Результат:**
Команда `cat /proc/mdstat` фиксирует процесс ресинхронизации (`recovery`), после чего массив возвращается в полностью исправное состояние `[UU]`.

---

## 8. Заключение

В ходе работы полностью выполнены все требования задания:
1. Выполнена ручная установка Arch Linux на программный RAID1 (`mdadm`) без применения автоустановщиков.
2. Подтверждён первоначальный запуск в режиме Legacy BIOS (SeaBIOS) и последующий перевод системы в режим UEFI.
3. Продемонстрирована независимая загрузка UEFI с резервной записи на втором диске при физическом извлечении первого накопителя в VirtualBox.
4. Документированы выводы команд `efibootmgr -v` и процедура восстановления деградировавшего массива (`cat /proc/mdstat`).
