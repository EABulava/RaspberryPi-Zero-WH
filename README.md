# 🕳️ Pi-hole на Raspberry Pi Zero WH (ARMv6)

> Повна інструкція встановлення Pi-hole v6.2.1 на Raspberry Pi Zero WH з ARMv6 та Debian Trixie.  
> Вирішує проблеми сумісності: несумісний `dialog`, відсутність офіційної підтримки ARMv6, Raspbian-репозиторії.

---

## Зміст

1. [Прошивка ОС](#1-прошивка-ос)
2. [Підключення по SSH](#2-підключення-по-ssh)
3. [Блокування IPv6 для apt](#3-блокування-ipv6-для-apt)
4. [Заміна репозиторіїв на Debian](#4-заміна-репозиторіїв-на-debian)
5. [Встановлення залежностей](#5-встановлення-залежностей)
6. [Заміна dialog на фейковий (ARMv6)](#6-заміна-dialog-на-фейковий-armv6)
7. [Завантаження інсталятора та репозиторіїв](#7-завантаження-інсталятора-та-репозиторіїв)
8. [Патчинг інсталятора](#8-патчинг-інсталятора)
9. [Запуск інсталятора](#9-запуск-інсталятора)
10. [Налаштування роутера](#10-налаштування-роутера)

---

## 1. Прошивка ОС

Використовуй **Raspberry Pi Imager**:

- Вибрати образ: `Raspberry Pi OS (32-bit)` — **Trixie**
- У розширених налаштуваннях вказати:
  - `hostname`
  - SSH (увімкнути)
  - Ім'я користувача та пароль
  - Wi-Fi (SSID + пароль)
- Записати на картку пам'яті

---

## 2. Підключення по SSH

```bash
ssh-keygen -R 192.168.1.88
ssh eadmin@192.168.1.88
```

---

## 3. Блокування IPv6 для apt

ARMv6 часто має проблеми з IPv6-з'єднаннями. Примусово використовуємо IPv4:

```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

---

## 4. Заміна репозиторіїв на Debian

Raspbian-репозиторії не підтримують ARMv6 належним чином — замінюємо на офіційний Debian Trixie.

### Вимкнути Raspbian-репозиторії

```bash
sudo mv /etc/apt/sources.list.d/raspbian.sources /etc/apt/sources.list.d/raspbian.sources.bak
sudo mv /etc/apt/sources.list.d/raspi.sources /etc/apt/sources.list.d/raspi.sources.bak
```

### Додати Debian Trixie

```bash
sudo tee /etc/apt/sources.list << 'EOF'
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
EOF
```

### Імпортувати GPG-ключі Debian

```bash
sudo gpg --keyserver keyserver.ubuntu.com --recv-keys \
  6ED0E7B82643E131 78DBA3BC47EF2265 F8D2585B8783D481 \
  54404762BBB6E853 BDE6D2B9216EC7A8 605C66F00D6C9793

sudo gpg --export \
  6ED0E7B82643E131 78DBA3BC47EF2265 F8D2585B8783D481 \
  54404762BBB6E853 BDE6D2B9216EC7A8 605C66F00D6C9793 \
  | sudo tee /etc/apt/trusted.gpg.d/debian.gpg > /dev/null

sudo apt update
```

---

## 5. Встановлення залежностей

```bash
sudo dpkg --remove --force-remove-reinstreq pihole-meta 2>/dev/null

sudo apt install -y \
  libdialog15 libjq1 libjemalloc2 libprotobuf-c1 \
  liberror-perl libonig5 lshw dialog git jq dnsutils
```

---

## 6. Заміна `dialog` на фейковий (ARMv6)

Бінарник `dialog` з репозиторіїв несумісний з ARMv6 і крашить інсталятор. Підміняємо його порожнім скриптом:

```bash
sudo mv /usr/bin/dialog /usr/bin/dialog.real

sudo tee /usr/bin/dialog << 'EOF'
#!/bin/sh
exit 0
EOF

sudo chmod +x /usr/bin/dialog
```

> **Примітка:** Оригінальний бінарник збережено у `/usr/bin/dialog.real`.

---

## 7. Завантаження інсталятора та репозиторіїв

Завантажуємо архіви замість `git clone` (уникаємо проблем з git на ARMv6):

```bash
# Інсталятор Pi-hole
wget -O ~/install.sh \
  https://raw.githubusercontent.com/pi-hole/pi-hole/v6.2.1/automated%20install/basic-install.sh

# Ядро Pi-hole
sudo mkdir -p /etc/.pihole
wget -O /tmp/pihole.tar.gz \
  https://github.com/pi-hole/pi-hole/archive/refs/tags/v6.2.1.tar.gz
sudo tar -xzf /tmp/pihole.tar.gz -C /etc/.pihole --strip-components=1

# Веб-інтерфейс
sudo mkdir -p /var/www/html/admin
wget -O /tmp/pihole-web.tar.gz \
  https://github.com/pi-hole/web/archive/refs/tags/v6.2.1.tar.gz
sudo tar -xzf /tmp/pihole-web.tar.gz -C /var/www/html/admin --strip-components=1
```

---

## 8. Патчинг інсталятора

Інсталятор намагається виконувати `git`-команди в директоріях, які ми завантажили як архіви. Замінюємо їх на `true` (no-op):

```bash
sed -i 's/git stash --all --quiet.*$/true # git stash/' ~/install.sh
sed -i 's/git clean --quiet --force -d.*$/true # git clean/' ~/install.sh
sed -i 's/git pull --no-rebase --quiet.*$/true # git pull/' ~/install.sh
sed -i 's/curBranch=\$(git rev-parse.*$/curBranch="master" # git rev-parse/' ~/install.sh
sed -i 's/git reset --hard.*$/true # git reset/' ~/install.sh
sed -i 's/        git status/        true # git status/' ~/install.sh
sed -i '/git clone -q/c\        true # git clone replaced' ~/install.sh
```

---

## 9. Запуск інсталятора

```bash
sudo PIHOLE_SKIP_OS_CHECK=true \
  PIHOLE_INTERFACE=wlan0 \
  IPV4_ADDRESS=192.168.1.88/24 \
  IPV6_ADDRESS="" \
  PIHOLE_DNS_1=8.8.8.8 \
  PIHOLE_DNS_2=8.8.4.4 \
  QUERY_LOGGING=true \
  INSTALL_WEB_SERVER=true \
  INSTALL_WEB_INTERFACE=true \
  LIGHTTPD_ENABLED=true \
  CACHE_SIZE=10000 \
  DNS_BOGUS_PRIV=true \
  DNSMASQ_LISTENING=single \
  WEBPASSWORD="" \
  BLOCKING_ENABLED=true \
  bash ~/install.sh --unattended
```

> Після завершення веб-інтерфейс доступний за адресою: **http://192.168.1.88/admin**

---

## 10. Налаштування роутера

На прикладі **Linksys**:

```
Локальна мережа → DHCP → Статичний DNS 1: 192.168.1.88 → Застосувати
```

Після цього всі пристрої в мережі автоматично використовуватимуть Pi-hole як DNS-резолвер.

---

## Вирішені проблеми

| Проблема | Рішення |
|---|---|
| `dialog` крашить інсталятор на ARMv6 | Заміна на фейковий `#!/bin/sh` скрипт |
| `git clone` не працює під час інсталяції | Патчинг інсталятора + завантаження архівів вручну |
| Raspbian-пакети несумісні з ARMv6 | Перехід на офіційний Debian Trixie |
| IPv6 блокує `apt update` | `Acquire::ForceIPv4 "true"` |

---

## Версії

| Компонент | Версія |
|---|---|
| Pi-hole | v6.2.1 |
| Raspberry Pi OS | Trixie (32-bit) |
| Архітектура | ARMv6 |

---

*Перевірено на Raspberry Pi Zero WH з Pi-hole v6.2.1 та Debian Trixie.*
