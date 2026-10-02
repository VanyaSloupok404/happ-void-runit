# Happ VPN on Void Linux (runit)

Руководство по установке и настройке VPN-клиента Happ (Happ-proxy) в Void Linux с системой инициализации runit.

---

## Архитектура и принцип работы

Официальный клиент Happ ориентирован на дистрибутивы с systemd. В бинарнике GUI зашиты прямые вызовы systemctl start/enable для управления фоновым демоном happd.

На Void Linux управление демоном берёт на себя runit:
1. Демон happd запускается как системный сервис runit от пользователя root.
2. Демон создаёт сокет IPC /tmp/happd.sock с правами 0666 (rw-rw-rw-), поднимает TUN-интерфейс happ-xray и управляет маршрутами ядра.
3. Графический интерфейс happ запускается от непривилегированного пользователя, автоматически находит активный сокет /tmp/happd.sock и взаимодействует с демоном напрямую.
4. Вызовы systemctl и повышение привилегий (sudo) со стороны GUI полностью исключаются.

---

## 1. Зависимости

Все необходимые Qt6-библиотеки упакованы внутри самого клиента Happ в директории /opt/happ/lib/. В Void Linux требуются лишь базовые системные пакеты для работы туннелей и распаковки deb-пакета:

    sudo xbps-install -S binutils tar xz

---

## 2. Установка бинарников Happ

1. Скачайте актуальный .deb-пакет со страницы релизов Happ-proxy/happ-desktop на GitHub.
2. Распакуйте содержимое пакета в корень системы:

    # Распаковка deb-архива
    ar x happ_*_amd64.deb
    sudo tar -xf data.tar.* -C /

    # Выставление прав на директории и бинарники
    sudo chown -R root:root /opt/happ
    sudo chmod +x /opt/happ/bin/Happ /opt/happ/bin/happd /opt/happ/bin/core/xray
    sudo chmod 777 /opt/happ/bin/core/routing

    # Создание симлинков в PATH
    sudo ln -sf /opt/happ/bin/Happ /usr/bin/happ
    sudo ln -sf /opt/happ/bin/happd /usr/bin/happd

Внимание: Если инсталлятор создал файл /etc/sudoers.d/happ с директивой NOPASSWD: ALL, немедленно удалите его. Клиенту не требуются права root:

    sudo rm -f /etc/sudoers.d/happ

---

## 3. Настройка сервиса runit

1. Создайте каталог службы и поместите скрипт запуска:

    sudo mkdir -p /etc/sv/happd
    sudo cp happd.run /etc/sv/happd/run
    sudo chmod +x /etc/sv/happd/run

2. Активируйте службу runit:

    sudo ln -s /etc/sv/happd /var/service/

3. Проверьте статус службы:

    sudo sv status happd
    # Ожидаемый вывод: run: happd: (pid ...) Xs

---

## 4. Интеграция с рабочим столом (Desktop Entry)

Для отображения программы в лаунчерах приложений (Rofi, Wofi, Fuzzel) и поддержки импорта конфигураций по ссылкам happ://:

    sudo cp happ.desktop /usr/share/applications/
    sudo update-desktop-database

---

## 5. Управление сервисом

- Запуск: sudo sv up happd
- Остановка: sudo sv down happd
- Перезапуск: sudo sv restart happd
- Просмотр логов демона: tail -f /var/log/happd.log
