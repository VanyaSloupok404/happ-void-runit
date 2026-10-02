# Happ VPN on Void Linux (runit)

Руководство по установке и настройке VPN-клиента Happ (Happ-proxy) в Void Linux с системой инициализации runit.

---

## Архитектура и принцип работы

Официальный клиент Happ ориентирован на дистрибутивы с systemd. В бинарнике GUI зашиты вызовы `systemctl start/enable` для управления фоновым демоном `happd`.

На Void Linux управление демоном передаётся runit:
1. Демон `happd` запускается как системный сервис runit от пользователя `root`.
2. Демон самостоятельно создаёт IPC-сокет `/tmp/happd.sock`, поднимает TUN-интерфейс `happ-xray` и выставляет маршруты ядра.
3. Графический интерфейс `happ` запускается от обычного пользователя, подключается к активному сокету `/tmp/happd.sock` и работает напрямую по IPC.
4. Вызовы `systemctl` и повышение привилегий (`sudo`) со стороны GUI полностью исключаются.

---

## 1. Зависимости

Qt6-библиотеки поставляются вендором в `/opt/happ/lib/`. В Void Linux требуются стандартные системные утилиты:

    sudo xbps-install -S binutils tar xz

---

## 2. Установка бинарников Happ

1. Скачайте актуальный `.deb`-пакет со страницы релизов [Happ-proxy/happ-desktop](https://github.com/Happ-proxy/happ-desktop/releases).
2. Распакуйте архив в корень системы:

    ar x happ_*_amd64.deb
    sudo tar -xf data.tar.* -C /

    # Выставление прав
    sudo chown -R root:root /opt/happ
    sudo chmod +x /opt/happ/bin/Happ /opt/happ/bin/happd /opt/happ/bin/core/xray

    # Права для записи маршрутов пользователем
    sudo chown root:users /opt/happ/bin/core/routing
    sudo chmod 775 /opt/happ/bin/core/routing

    # Симлинки в PATH
    sudo ln -sf /opt/happ/bin/Happ /usr/bin/happ
    sudo ln -sf /opt/happ/bin/happd /usr/bin/happd

> **Безопасность:** Если при установке появился файл `/etc/sudoers.d/happ` с правилом `NOPASSWD: ALL` — немедленно удалите его. Демон работает через runit, и клиенту root-права не нужны:
>
>     sudo rm -f /etc/sudoers.d/happ

---

## 3. Настройка сервиса runit

1. Создайте каталог службы и поместите скрипт запуска:

    sudo mkdir -p /etc/sv/happd
    sudo cp happd.run /etc/sv/happd/run
    sudo chmod +x /etc/sv/happd/run

2. Активируйте службу runit:

    sudo ln -s /etc/sv/happd /var/service/

3. Проверьте статус:

    sudo sv status happd

---

## 4. Интеграция с рабочим столом (Desktop Entry)

Для интеграции с меню приложений (Rofi, Wofi, Fuzzel) и поддержки ссылок подписки `happ://`:

    sudo cp happ.desktop /usr/share/applications/
    sudo update-desktop-database

---

## 5. Управление сервисом

- Запуск: `sudo sv up happd`
- Остановка: `sudo sv down happd`
- Перезапуск: `sudo sv restart happd`
- Просмотр логов: `tail -f /var/log/happd.log`

---

## Технические примечания по безопасности

1. **Сокет `/tmp/happd.sock` (0666):** Права задаются кодом демона `happd` на этапе компиляции апстрима для связи непривилегированного GUI с root-процессом без polkit/sudo.
2. **Логирование:** Демон `happd` имеет встроенный C++ обработчик, который пишет напрямую в `/var/log/happd.log`. Дополнительный лог-сервис runit не требуется.
3. **Каталог `core/routing`:** Апстрим поставляет директорию с правами `777`. В данной инструкции доступ ограничен группой `users` с правами `775`.
