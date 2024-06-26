# Настройка сервера

Сервер оригинальной Каракатицы работает под ALT Linux, но мне для теста привычнее было поднять на Debian. Заранее предупреждаю, что вся эта конфигурация - "лишь бы работало", полноценно протестировать возможности пока нет. Хотя бы потому что нет "белого" IP-адреса, по которому можно было бы расположить сервер.

Чтобы безбоязненно проводить эксперименты, воспользуемся qemu, создадим там виртуальную машину и установим базовую систему (debian-netinst). Туда же устанавливаем все необходимые программы вроде ssh, git, screen, компилятора и т.д. и настраиваем права доступа - включаем нужных пользователей в нужные группы, пишем правила udev.

Подключение пользователей можно сделать просто по ssh. Хоть через ключи, хоть по паролю.

Для трансляции изображения с камеры, направленной на Каракатицу, можно взять ```ustreamer``` из репозитория. Просто прописываем в автозагрузку (возможно, с параметрами: ```ustreamer -i 0 -r 800x600 -f 15``` - захват с камеры 0, разрешение 800x600, фпс 15 кадров в секунду). При старте он поднимает локальный http-сервер по умолчанию на порту 8080.

Теперь нужно каким-то образом пробросить в виртуалку камеру и Каракатицу. Получить их адреса можно обычным lsusb. У меня Каракатица скрывается под ```16c0:05df```, а камера - под ```13d3:56a8```. В первую очередь важно убедиться, что у пользователя, от которого будем запускать виртуалку **на хосте** вообще есть права на них. При необходимости можно подправить udev:

```
#/etc/udev/rules.d/95.Karakatitsa.rules

SUBSYSTEM=="usb", ATTRS{idVendor}=="13d3", ATTRS{idProduct}=="56a8", GROUP="user", MODE="0660"
SUBSYSTEM=="usb", ATTRS{idVendor}=="16c0", ATTRS{idProduct}=="05df", GROUP="user", MODE="0660"
```

```
# udevadm control --reload-rules && udevadm trigger
```

Теперь виртуальную машину можно запустить например с такими ключами:

```
qemu-system-x86_64 -m 2G -hda qemu_image.img -name "Lin_ssh_user@127.0.0.1:45022" \
-display none \
-net nic -nic user,model=virtio-net-pci,mac="52:54:00:f8:86:7e",ipv4=on,hostfwd=:127.0.0.1:45022-:22 \
-usb -device usb-host,vendorid=0x16c0,productid=0x05df \
-usb -device usb-host,vendorid=0x16c0,productid=0x05df \
-device usb-ehci,id=usb \
 --device usb-host,vendorid=0x13d3,productid=0x56a8 \
```

Шаманство с портом 45022 говорит, что к машине можно будет получить доступ локально по ssh: ```ssh -p 45022 user@127.0.0.1 -L8008:localhost:8080 ```

Обратите внимание, что в моем случае к серверу подключены две Каракатицы, поэтому и в настройках строчка ```-usb -device usb-host,vendorid=0x16c0,productid=0x05df``` продублирована.

На время настройки можно оставить ```-nographic``` для консольного доступа или ```-display gtk,gl=on``` для графического, но при повседневном использовании дисплей серверу не нужен.

Для доступа пользователей к COM-портам и "клавиатуре" они должны входить в группы ```dialout``` и ```uucp```. Также нужно прописать правило для udev:

```
#/etc/udev/rules.d/70-Karakatitsa.rules 

SUBSYSTEM=="tty", ATTRS{manufacturer}=="COKPOWEHEU" ENV{CONNECTED_vusb}="yes"
#ENV{CONNECTED_vusb}=="yes", SUBSYSTEM=="tty", ATTRS{interface}=="?*", PROGRAM="/bin/bash -c \"ls /dev | grep tty_$attr{interface}_ | wc -l \"", SYMLINK+="tty_$attr{interface}_%c"
ENV{CONNECTED_vusb}=="yes", SUBSYSTEM=="tty", ATTRS{interface}=="?*", SYMLINK+="tty_$attr{interface}_0"
# USB
SUBSYSTEM=="usb", ATTRS{idVendor}=="16c0", ATTRS{idProduct}=="05df", GROUP="uucp", MODE="0660"
```

Благодаря нему у Каракатицы появятся человеко-читаемые симлинки в системе. В отличие от номеров портов, их можно хоть в makefile прописать и быть уверенным, что прошьется то, что нужно:

```
$ ls -l  /dev | grep tty_
lrwxrwxrwx 1 root root           7 мар 29 02:13 tty_CH32_DBG_0 -> ttyACM0
lrwxrwxrwx 1 root root           7 мар 29 02:13 tty_CH32_PROG_0 -> ttyACM1
lrwxrwxrwx 1 root root           7 мар 29 02:13 tty_GD32_DBG_0 -> ttyACM2
lrwxrwxrwx 1 root root           7 мар 29 02:13 tty_GD32_LOG_0 -> ttyACM4
lrwxrwxrwx 1 root root           7 мар 29 02:13 tty_GD32_PROG_0 -> ttyACM3
```

## Настройка удаленного подключения

Допустим, логин пользователя на виртуальной машине ```user```, трансляция с камеры идет на 8080 порт, а для самого стенда обеспечен "белый" IP-адрес 120.340.560.780:100500.

Добавляем ssh-ключи на стенд. Подобная операция будет проводиться и каждый раз, когда надо добавить нового пользователя. Допустим, он прислал ключ ```ключ.pub```, сгенерированный по примеру [отсюда](Remote_lin.md) или [отсюда](Remote_win.md):

```
cd ~user # Переход в домашний каталог пользователя, который будет работать со стендом
mkdir -p .ssh
cat ключ.pub >> .ssh/authorized_keys # Добавляем ПУБЛИЧНЫЙ ключ к списку доступных
```

И отключаем удаленный доступ по паролю (вот это для каждого пользователя повторять не надо). В файле **/etc/ssh/sshd_config** раскомментируем строчку ```PasswordAuthentication``` и меняем ее значение на ```no```:

```
#/etc/ssh/sshd_config
...
#   PasswordAuthentication yes
PasswordAuthentication no
```

Но пока что удаленный пользователь может запустить ```su``` и повысить себе привилегии. Чтобы это исправить, создаем группу wheel ( ```addgroup wheel``` ) и прописываем запрет всем, кто в нее не входит (а свежесозданный пользователь не входит, хотя в этом можно и отдельно удостовериться), пользоваться этой утилитой. Для этого в файле ```/etc/pam.d/su``` находим и раскомментируем строку ```auth required pam_wheel.so```

Теперь по желанию запуск всех необходимых программ прописывается в автозагрузку.

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/

[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png

[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg