# Установка MikroTik CHR на VPS (UEFI и Legacy)

Инструкция для установки **Cloud Hosted Router (CHR)** на любой VPS/облачный сервер через rescue/recovery (Debian/Ubuntu и аналоги).

Подходит, если провайдер отдаёт сервер с **eth0**, статическим публичным IP и шлюзом по умолчанию.

> **Внимание:** команды `dd` перезаписывают диск (`/dev/sda`). Убедитесь, что работаете на нужном сервере и с нужным диском.

---

## Перед установкой

Обновить пакеты и **записать сетевые параметры VPS** — они понадобятся для `autorun.scr`:

```bash
apt update
ip a
ip r
```

Пример вывода (значения будут **ваши**):

```text
# ip a
inet YOUR_PUBLIC_IP/32 scope global eth0

# ip r
default via YOUR_GATEWAY dev eth0
```

| Параметр | Откуда взять | Пример плейсхолдера |
|----------|--------------|---------------------|
| `YOUR_PUBLIC_IP` | `ip a` → eth0 | `203.0.113.10` |
| `YOUR_GATEWAY` | `ip r` → default via | `172.31.1.1` |
| `eth0` | имя интерфейса в rescue | обычно `eth0` |

---

## UEFI (рекомендуется для современных VPS)

### 1. Скачать образ

```bash
wget https://github.com/tikoci/fat-chr/releases/download/Build13501134290-jaclaz/chr-uefi-fat.raw
```

Актуальные сборки: [tikoci/fat-chr releases](https://github.com/tikoci/fat-chr/releases).

### 2. Найти offset раздела

```bash
fdisk -lu chr-uefi-fat.raw
```

Запомните **offset** раздела с данными (в примере ниже — `33571840` байт; у другого образа может отличаться).

### 3. Смонтировать образ

```bash
mount -o loop,offset=33571840 chr-uefi-fat.raw /mnt
```

### 4. Прописать сеть в autorun.scr

```bash
nano /mnt/rw/autorun.scr
```

Содержимое (подставьте **свои** IP и шлюз):

```routeros
/ip address
add address=YOUR_PUBLIC_IP/32 interface=eth0 network=YOUR_PUBLIC_IP
/ip route
add gateway=YOUR_GATEWAY
```

Пример:

```routeros
/ip address
add address=203.0.113.10/32 interface=eth0 network=203.0.113.10
/ip route
add gateway=172.31.1.1
```

### 5. Размонтировать и записать на диск

```bash
umount /mnt
echo u > /proc/sysrq-trigger
dd if=chr-uefi-fat.raw of=/dev/sda bs=4M oflag=sync
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

Последняя команда — **перезагрузка**. После reboot VPS поднимется уже с RouterOS.

---

## Legacy (старые VPS без UEFI)

### 1. Скачать и распаковать образ

```bash
apt install -y unzip
wget https://download.mikrotik.com/routeros/7.16.1/chr-7.16.1.img.zip
unzip chr-7.16.1.img.zip
```

Версию RouterOS можно взять новее с [mikrotik.com/download](https://mikrotik.com/download) — для Legacy нужен файл **`chr-X.X.X.img.zip`**.

### 2. Смонтировать и настроить autorun.scr

```bash
fdisk -lu chr-7.16.1.img
mount -o loop,offset=33571840 chr-7.16.1.img /mnt
nano /mnt/rw/autorun.scr
```

Тот же `autorun.scr`, что в UEFI-секции (`YOUR_PUBLIC_IP`, `YOUR_GATEWAY`).

### 3. Запись на диск и reboot

```bash
umount /mnt
echo u > /proc/sysrq-trigger
dd if=chr-7.16.1.img of=/dev/sda bs=4M oflag=sync
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

---

## Первый вход и лицензия

1. Подключитесь по **Winbox** или **SSH** на `YOUR_PUBLIC_IP`.
2. Логин по умолчанию: **`admin`**, пароль — **пустой** (сразу смените).
3. Откройте **System → License**.

### Активация trial P1 / P10

В разделе лицензии введите **свои** учётные данные аккаунта MikroTik (создайте на [mikrotik.com](https://mikrotik.com) если нет):

| Поле | Что указать |
|------|-------------|
| **Login (email)** | `YOUR_MIKROTIK_ACCOUNT@example.com` |
| **Password** | `YOUR_MIKROTIK_PASSWORD` |

Нажмите **Start trial** и выберите уровень **P1** или **P10** (зависит от лимитов CHR у MikroTik на момент активации).

### Что важно знать про лицензию

- После окончания trial **лицензия сохраняется на этом CHR** и продолжает работать **без ограничения по времени** для уже активированного уровня (P1/P10).
- Это практичный способ получить полноценный CHR на своём VPS без покупки perpetual license — условно «MikroTik для тех, кто поднимает свой роутер на VPS».
- **Ограничение:** на таком CHR **нельзя обновлять RouterOS** через стандартный upgrade — прошивка остаётся на версии образа, с которым установили. Для home/lab/edge-router с фиксированной версией это обычно приемлемо.

> Не публикуйте и не коммитьте в git свои логин/пароль MikroTik — только в Winbox/CLI на роутере.

---

## Рекомендуемый hardening после установки

Сразу после первого входа — отключить лишние сервисы и layer-2 доступ:

```routeros
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set api disabled=yes
set api-ssl disabled=yes

/ip neighbor discovery-settings
set discover-interface-list=none

/tool mac-server
set allowed-interface-list=none
/tool mac-server mac-winbox
set allowed-interface-list=none
/tool mac-server ping
set enabled=no
```

| Команда | Зачем |
|---------|--------|
| Отключение telnet/ftp/www/api | Убрать лишние точки входа на публичном IP |
| `discover-interface-list=none` | Не «светиться» в Neighbors (MNDP/CDP) |
| MAC-server / mac-winbox / ping off | Закрыть доступ к роутеру по MAC в обход IP-firewall |

Ограничить SSH и Winbox по адресам (подставьте свои):

```routeros
/ip service
set ssh address=YOUR_LAN_SUBNET,YOUR_OFFICE_IP/32
set winbox address=YOUR_LAN_SUBNET,YOUR_OFFICE_IP/32
```

Защита от брутфорса по логам — см. [CHR_MIKROTIK_FAIL2BAN.md](./CHR_MIKROTIK_FAIL2BAN.md).

---

## Troubleshooting

| Проблема | Что проверить |
|----------|----------------|
| После reboot нет сети | `autorun.scr`: верные `YOUR_PUBLIC_IP/32` и `YOUR_GATEWAY`; имя интерфейса `eth0` |
| Не монтируется образ | Пересчитать `offset` через `fdisk -lu` |
| VPS не грузится (UEFI/Legacy) | Попробовать другой режим образа (UEFI ↔ Legacy) |
| Winbox не подключается | Firewall провайдера, порт 8291; временно `/ip service set winbox disabled=no` |

---

---

# Installing MikroTik CHR on a VPS (UEFI and Legacy)

Guide for installing **Cloud Hosted Router (CHR)** on any VPS/cloud server via rescue/recovery (Debian/Ubuntu, etc.).

Works when the provider gives you **eth0**, a static public IP, and a default gateway.

> **Warning:** `dd` overwrites the disk (`/dev/sda`). Confirm server and disk before running.

---

## Before installation

Update packages and **note network settings** for `autorun.scr`:

```bash
apt update
ip a
ip r
```

Example output (use **your** values):

```text
# ip a
inet YOUR_PUBLIC_IP/32 scope global eth0

# ip r
default via YOUR_GATEWAY dev eth0
```

| Parameter | Source | Placeholder example |
|-----------|--------|---------------------|
| `YOUR_PUBLIC_IP` | `ip a` → eth0 | `203.0.113.10` |
| `YOUR_GATEWAY` | `ip r` → default via | `172.31.1.1` |
| `eth0` | interface name in rescue | usually `eth0` |

---

## UEFI (recommended for modern VPS)

### 1. Download image

```bash
wget https://github.com/tikoci/fat-chr/releases/download/Build13501134290-jaclaz/chr-uefi-fat.raw
```

Latest builds: [tikoci/fat-chr releases](https://github.com/tikoci/fat-chr/releases).

### 2. Find partition offset

```bash
fdisk -lu chr-uefi-fat.raw
```

Note the data partition **offset** (example below uses `33571840` bytes — yours may differ).

### 3. Mount the image

```bash
mount -o loop,offset=33571840 chr-uefi-fat.raw /mnt
```

### 4. Edit autorun.scr

```bash
nano /mnt/rw/autorun.scr
```

Content (replace with **your** IP and gateway):

```routeros
/ip address
add address=YOUR_PUBLIC_IP/32 interface=eth0 network=YOUR_PUBLIC_IP
/ip route
add gateway=YOUR_GATEWAY
```

### 5. Unmount and write to disk

```bash
umount /mnt
echo u > /proc/sysrq-trigger
dd if=chr-uefi-fat.raw of=/dev/sda bs=4M oflag=sync
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

The last command **reboots** the VPS into RouterOS.

---

## Legacy (older VPS without UEFI)

### 1. Download and extract

```bash
apt install -y unzip
wget https://download.mikrotik.com/routeros/7.16.1/chr-7.16.1.img.zip
unzip chr-7.16.1.img.zip
```

For Legacy use **`chr-X.X.X.img.zip`** from [mikrotik.com/download](https://mikrotik.com/download).

### 2. Mount and configure autorun.scr

```bash
fdisk -lu chr-7.16.1.img
mount -o loop,offset=33571840 chr-7.16.1.img /mnt
nano /mnt/rw/autorun.scr
```

Same `autorun.scr` as in the UEFI section.

### 3. Write to disk and reboot

```bash
umount /mnt
echo u > /proc/sysrq-trigger
dd if=chr-7.16.1.img of=/dev/sda bs=4M oflag=sync
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

---

## First login and license

1. Connect via **Winbox** or **SSH** to `YOUR_PUBLIC_IP`.
2. Default login: **`admin`**, password **empty** (change immediately).
3. Open **System → License**.

### P1 / P10 trial activation

Enter **your** MikroTik account credentials ([mikrotik.com](https://mikrotik.com)):

| Field | Value |
|-------|--------|
| **Login (email)** | `YOUR_MIKROTIK_ACCOUNT@example.com` |
| **Password** | `YOUR_MIKROTIK_PASSWORD` |

Click **Start trial** and choose **P1** or **P10**.

### License notes

- After the trial ends, the **license remains on this CHR** and keeps working **without a time limit** for the activated tier (P1/P10).
- Practical way to run a full CHR on your own VPS without buying a perpetual license.
- **Limitation:** you **cannot upgrade RouterOS** on this CHR via normal upgrade — firmware stays at the install image version. Usually fine for a fixed-version edge router.

> Never commit or publish your MikroTik login/password — enter only in Winbox on the router.

---

## Recommended hardening

```routeros
/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set api disabled=yes
set api-ssl disabled=yes

/ip neighbor discovery-settings
set discover-interface-list=none

/tool mac-server
set allowed-interface-list=none
/tool mac-server mac-winbox
set allowed-interface-list=none
/tool mac-server ping
set enabled=no
```

Restrict SSH/Winbox by address:

```routeros
/ip service
set ssh address=YOUR_LAN_SUBNET,YOUR_OFFICE_IP/32
set winbox address=YOUR_LAN_SUBNET,YOUR_OFFICE_IP/32
```

Log-based brute-force ban — see [CHR_MIKROTIK_FAIL2BAN.md](./CHR_MIKROTIK_FAIL2BAN.md).

---

## Troubleshooting

| Issue | Check |
|-------|--------|
| No network after reboot | `autorun.scr`: correct IP/gateway; interface `eth0` |
| Mount fails | Recalculate `offset` with `fdisk -lu` |
| VPS won't boot | Try the other image type (UEFI ↔ Legacy) |
| Winbox won't connect | Provider firewall, port 8291 |

---

## License (this document)

MIT — see `LICENSE` in the repository root (when published).
