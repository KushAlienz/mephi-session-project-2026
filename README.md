# MEPHI Session Project 2026

## Сессионный проект

**Тема:** Настройка базовых средств защиты ОС GNU/Linux  
**Студент:** Николаев Дмитрий Дмитриевич  
**Номер зачётки:** М265273  
**Дата:** 30.09.2026  
**Операционная система:** Red OS 8.0.3, VirtualBox

---

## Раздел 1. Установка дистрибутива

| Действие | Команда |
|---|---|
| Проверка соединений | nmcli con show |
| Проверка IP | ip -4 a show enp0s3 |
| Установка hostname | sudo hostnamectl set-hostname mephi-2026.domain.local |
| Проверка hostname | hostname -f |
| Проверка связности | ping -c 4 8.8.8.8 > ~/ping.out |

**Результат:** хост mephi-2026.domain.local, IP 10.0.2.15/24 получен по DHCP.

---

## Раздел 2. Управление программным обеспечением

| Действие | Команда |
|---|---|
| Обновление | sudo dnf update -y |
| Установка пакетов | sudo dnf install -y nginx libcap-ng-utils |
| Скачивание RPM | cd /tmp && sudo dnf download tcpdump |
| Установка RPM | sudo rpm -Uvh --replacepkgs /tmp/tcpdump-*.x86_64.rpm |
| История | dnf history > ~/dnf.out |

**Результат:** установлены nginx-1.30.4, libcap-ng-utils-0.8.3, tcpdump-4.99.5.

---

## Раздел 3. Управление файловыми системами

| Действие | Команда |
|---|---|
| Создание раздела | sudo parted /dev/sdb --script mklabel msdos mkpart primary ext4 1MiB 100% |
| Форматирование | sudo mkfs.ext4 -L MEPHI_WEB /dev/sdb1 |
| Точка монтирования | sudo mkdir -p /mephi-web |
| Добавление в fstab | echo 'LABEL=MEPHI_WEB /mephi-web ext4 defaults 0 2' >> /etc/fstab |
| Монтирование | sudo mount -a && findmnt /mephi-web |

**Результат:** /dev/sdb1 (20 ГБ, ext4, метка MEPHI_WEB) автоматически монтируется в /mephi-web.

---

## Раздел 4. Управление сервисами

| Действие | Команда |
|---|---|
| Автозапуск nginx | sudo systemctl enable --now nginx |
| Проверка | systemctl is-enabled nginx && systemctl is-active nginx |
| Сохранение журнала | journalctl -u nginx -b > ~/journalctl.out |

**Результат:** nginx работает, автозапуск включён.

---

## Раздел 5.1. Дискреционное управление доступом (DAC)

| Действие | Команда |
|---|---|
| Группа кураторов | sudo groupadd -g 4444 curators |
| Разработчики | sudo useradd -u 5501 -m user1 (также user2, user3) |
| Кураторы | sudo useradd -m curator1 && sudo usermod -aG curators curator1 |
| Группа разработчиков | sudo groupadd developers && sudo usermod -aG developers user1 |
| Директория проекта | sudo mkdir -p /data/mephi-2026 |
| Владелец и права | sudo chown root:developers /data/mephi-2026 && sudo chmod 2770 /data/mephi-2026 |
| ACL по умолчанию | sudo setfacl -d -m g:developers:rwx,g:curators:r-x,o::--- /data/mephi-2026 |

**Результат:** разработчики работают совместно, кураторы только читают, остальные без доступа.

---

## Раздел 5.2. Привилегии (capabilities)

| Действие | Команда |
|---|---|
| Проверка прав | ls -la /usr/sbin/tcpdump |
| Выдача capabilities | sudo setcap cap_net_raw,cap_net_admin+eip /usr/sbin/tcpdump |
| Проверка | getcap /usr/sbin/tcpdump > ~/getcap.out |
| Тест от user1 | sudo -u user1 tcpdump -c 1 -i any |

**Результат:** getcap показывает cap_net_admin,cap_net_raw=eip, tcpdump работает от user1.

---

## Раздел 5.3. Мандатное управление доступом (SELinux)

| Действие | Команда |
|---|---|
| Режим SELinux | getenforce > ~/getenforce.out |
| Правило fcontext | sudo semanage fcontext -a -t httpd_sys_content_t "/mephi-web(/.*)?" |
| Применение | sudo restorecon -Rv /mephi-web |
| Проверка | ls -Z /mephi-web |

**Результат:** SELinux в режиме Enforcing, контекст httpd_sys_content_t.

---

## Раздел 6.1. Ограничение входа для кураторов

| Действие | Команда |
|---|---|
| Правила запрета | echo "-:curator1:ALL" >> /etc/security/access.conf (и для curator2) |
| PAM | echo "account required pam_access.so" >> /etc/pam.d/login |

**Результат:** curator1 и curator2 не могут войти через консоль, user1/2/3 входят нормально.

---

## Раздел 6.2. Управление паролями

| Действие | Команда |
|---|---|
| Срок пароля | sudo sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs |
| Применение | for u in user1 user2 user3 curator1 curator2; do sudo chage -M 90 "$u"; done |
| Минимальная длина | echo "minlen = 12" >> /etc/security/pwquality.conf |
| Enforcing | echo "enforcing = 1" >> /etc/security/pwquality.conf |
| Enforce для root | sudo sed -i 's/pam_pwquality.so.*/pam_pwquality.so enforce_for_root/' /etc/pam.d/system-auth |

**Результат:** пароль меняется каждые 90 дней, минимальная длина 12 символов.

---

## Раздел 7. Тестирование

| Действие | Команда |
|---|---|
| Создание страницы | echo "Hello from Student: М265273" > /mephi-web/index.html |
| Владелец и права | sudo chown nginx:nginx /mephi-web/index.html && sudo chmod 644 /mephi-web/index.html |
| Контекст | sudo restorecon -v /mephi-web/index.html |
| Кодировка | добавлено charset utf-8; в /etc/nginx/nginx.conf |
| Проверка | curl http://localhost/ |

**Результат:** curl возвращает Hello from Student: М265273.

---

## Раздел 8. Публикация на GitHub

### Артефакты

| Файл | Назначение |
|---|---|
| mephi-screenshot.png | Скриншот web-страницы |
| history.out | История команд |
| ping.out | Проверка сети |
| dnf.out | Управление пакетами |
| stat.out | Права доступа и SELinux |
| journalctl.out | Запуск nginx |
| getcap.out | Capabilities tcpdump |
| getenforce.out | Режим SELinux |
| curl.out | Результат тестирования |

### Системные файлы

| Файл | Содержимое |
|---|---|
| fstab | Монтирование /mephi-web |
| passwd | Пользователи user1/2/3, curator1/2 |
| shadow | Пароли |
| group | Группы curators, developers |
| pwquality.conf | minlen=12, enforcing=1 |

### Пакет tcpdump

- tcpdump-4.99.5-1.red80.x86_64.rpm

---

## Итог

Все 8 разделов проекта выполнены. Настройки сохраняются после перезагрузки:
