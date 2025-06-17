Основано на материале с канала Дневник UNIX’оида:
https://t.me/thmUNIX
https://youtube.com/@thmUNIX

Версия туториала: 1.1
## Шаг 0 - Меняем шрифт
Чтобы лучше было видно команды:
```sh
setfont /usr/share/kbd/consolefonts/sun12x22.psfu.gz
```

## Шаг 1 - Подключение к Интернету
- *Ethernet* → ничего делать не требуется
- *Wi-Fi* → прописываем команду **iwctl**
```sh
station list
station адаптер get-networks
station адаптер connect “SSID_сети”
# (ждем 10-15 секунд)
quit
```
Если возникла какая-либо проблема, попробуйте выполнить команду `rfkill unblock all` и снова попробуйте подключиться к Wi-Fi

## Шаг 2 - Настройка пакетного менеджера
- nano /etc/pacman.d/mirrorlist
Проверяем, что Reflector сгенерировал список зеркал. Если нет, комментируем все зеркала и расскоментируем те, которыми хотим пользоваться.
- Если списка зеркал нет, можно сгенерировать его:
```sh
sudo reflector --country "Russia" --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
```
- nano /etc/pacman.conf
Раскомментируем параметр `ParallelDownloads` и задаём желаемое значение (например 15)

## Шаг 3 - Разметка диска
- Смотрим имена разделов и находим нужный диск
```sh
fdisk -l
lsblk
```
- Размечаем нужный диск (например `/dev/nvme0n1`)
```sh
cfdisk диск
```
>- Даже если разметка дисков уже есть, её нужно удалить (кроме `/home`) и создать новую
>- Для разделов выбираем пункт `Delete`, `New`, `Type` (выбираем по таблице). В конце выбираем пункт `Write`
>- Точки монтирования будут использоваться позже

| #   | Тип              | Размер              | Точка монтирования |
| --- | ---------------- | ------------------- | ------------------ |
| 1   | EFI system       | 256M                | /boot/efi          |
| 2   | Linux filesystem | Все свободное место | /                  |
или

| #   | Тип              | Размер              | Точка монтирования |
| --- | ---------------- | ------------------- | ------------------ |
| 1   | EFI system       | 256M                | /boot/efi          |
| 2   | Linux filesystem | 25G ~ 30G           | /                  |
| 3   | Linux filesystem | Все свободное место | /home              |

## Шаг 4 - Форматирование разделов
```sh
mkfs.vfat /dev/диск1 # p1 efi
mkfs.ext4 /dev/диск2 # p2 linux
# если /home выносили в отдельный раздел:
mkfs.ext4 /dev/диск3 # p3 linux
```

## Шаг 5 - Монтирование разделов
```sh
mount /dev/диск2 /mnt # p2 linux
mkdir -p /mnt/boot/efi
mount /dev/диск1 /mnt/boot/efi # p1 efi

# если /home выносили в отдельный раздел:
mkdir -p /mnt/home
mount /dev/диск3 /mnt/home # p3 linux
```

## Шаг 6 - Установка пакетов
- Пример команды (смотрите дальше, чтобы выбрать то, что вам нужно)
```
pacstrap /mnt base base-devel linux linux-firmware linux-headers nano vim bash-completion grub efibootmgr xorg ttf-ubuntu-font-family ttf-hack ttf-dejavu ttf-opensans lxdm xfce
```
`pacstrap` позволяет выбрать место установки пакетов
- Устанавливать пакеты можно частями, если они плохо устанавливаются в live cd. Некоторые пакеты можно доустановить после `arch-chroot` с помощью обычного `pacman`:
```sh
arch-chroot /mnt
pacman -S пакеты
exit
umount -R /mnt
```
- Пакеты
Необходимый минимум
```
base base-devel linux linux-firmware linux-headers bash-completion grub efibootmgr networkmanager network-manager-applet
```
Текстовые консольные редакторы (можно выбрать один или несколько)
```
nano vim neovim
```
Дисплейный сервер Xorg
```
xorg
```
 Шрифты
```
ttf-ubuntu-font-family ttf-hack ttf-dejavu ttf-opensans
```
Дисплейный менеджер (выберите ОДИН)
```
sddm / lightdm / lxdm / gdm
```
Окружение рабочего стола (выберите ОДНО)
```
gnome / plasma / cinnamon /
budgie / xfce4 / lxqt / lxde / ...
```
Проприетарный драйвер NVIDIA
```
nvidia
```

## Шаг 7 - Генерация fstab
```sh
genfstab /mnt >> /mnt/etc/fstab
```

## Шаг 8 - Смена корня системы
```sh
arch-chroot /mnt
```

## Шаг 9 - Включение сервисов
- Включаем NetworkManager
```sh
systemctl enable NetworkManager
```
- Включаем дисплейный менеджер
```sh
systemctl enable название_DM
# sddm / lxdm / gdm / ...
```

## Шаг 10 - Пользователи & пароли
- Создаем пользователя
```
useradd -m имя_пользователя
```
- Устанавливаем пароль пользователю
```
passwd имя_пользователя
```
- Устанавливаем пароль пользователю root
```
passwd
```
- Даем себе право на sudo
Рекомендуется (если вы умеете пользоваться vim):
```
visudo
```
Отредактировать напрямую:
```
nano /etc/sudoers
```
Добавляем:
```
root ALL=(ALL:ALL) ALL
имя_пользователя ALL=(ALL:ALL) ALL
```

## Шаг 11 - Локали & язык системы
- nano /etc/locale.gen
Раскомментируем нужные вам локали. Например:
```
en_US.UTF-8 UTF-8
ru_RU.UTF-8 UTF-8
```
- nano /etc/locale.conf
Вписываем строку:
```
LANG= ”локаль”
```
Укажите желаемый язык системы. Например, **”en_US.UTF-8”**
- Генерация раскомментированных локалей
```sh
locale-gen
```

## Шаг 12 - Установка загрузчика
- Установить загрузчик на диск
```sh
grub-install /dev/диск # /dev/nvme0n1
```
- **nano /etc/default/grub**
Из параметра `GRUB_CMDLINE_LINUX_DEFAULT` убираем параметр `quiet` - это не обязательно 
- Сохраняем конфиг загрузчика
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

## Шаг 13 - Перезагрузка в новую систему
- Переходим обратно в shell Live CD
```sh
exit
```
- Размонтируем все разделы диска
```sh
umount -R /mnt
```
- Перезагружаемся
```sh
reboot
```

## Шаг 14 - Подключение к Интернету
- Ethernet → ничего делать не требуется
- Wi-Fi →
Если в DE предусмотрен GUI для подключения к Wi-Fi, то подключаемся как обычно
Если нет, то используем **nmtui**
    - Выбираем пункт `Radio`, проверяем, что Wi-Fi включен. Если нет, включаем.
    - Возвращаемся назад, выбираем пункт `Activate a connection`
    - Выбираем `AP`, вводим пароль, подключаемся
    - Выходим

## Шаг 15 - Настройка даты & времени
- Устанавливаем часовой пояс
```sh
sudo timedatectl set-timezone часовой_пояс
# Например, Europe/Moscow
```
Часовые пояса лежат в `/usr/share/zoneinfo`
- Включаем NTP (время по сети)
```sh
sudo timedatectl set-ntp true
```

## Шаг 16 - Создание swap-файла
Например, 6G. Рекомендуется создавать в двое меньше, чем размер оперативной памяти
```sh
sudo mkswap -U clear --size размер --file /swapfile
sudo swapon /swapfile
```
- **nano /etc/fstab**
Дописываем в конец (колонки отделяем Tab’ом):
```
/swapfile none swap defaults 0 0
```
- Всё готово, перезагружаемся
```sh
reboot
```
