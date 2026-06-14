# Установка MikroTik CHR на VPS (UEFI и Legacy)

Инструкция для установки **Cloud Hosted Router (CHR)** на VPS/облачный сервер с Linux (Debian/Ubuntu и аналоги).

Подходит, если провайдер отдаёт сервер с **eth0**, статическим публичным IP и шлюзом по умолчанию.

> **Внимание:** команды `dd` перезаписывают диск (`/dev/sda`). Убедитесь, что работаете на нужном сервере и с нужным диском.

---

## Нужен ли recovery mode?

**Не обязательно** — но нужна **Linux-среда с доступом к диску**, откуда вы запускаете `wget`, `mount` и `dd`.

| Способ | Суть |
|--------|------|
| **Recovery/rescue у провайдера** | Рекомендуется: диск не занят рабочей системой, меньше риска |
| **Обычный SSH в Linux на VPS** | **Тоже работает** — те же команды; вы сознательно затираете текущую ОС и после `dd` делаете reboot |
| **Marketplace / готовый образ** | У части хостеров CHR ставится из панели — без `dd` вручную |
| **Загрузка с ISO** | Если провайдер даёт boot from ISO — можно загрузить образ CHR напрямую |

### SSH без recovery — когда это ок

Если VPS уже под Linux и вы **всё равно его перезаписываете**, можно просто зайти по SSH и выполнить шаги ниже **без** переключения в rescue mode провайдера. Многие так и делают.

Ограничения:

- Корневая файловая система в этот момент **смонтирована с того же диска**, который затираете — теоретически рискованнее, чем rescue; на практике для «одноразовой переустановки» обычно проходит нормально.
- После `dd` **обязательно reboot** — старая ОС больше не загрузится.
- Rescue удобнее, если на диске **важные данные** или несколько разделов — там проще не ошибиться.

**Итого:** recovery mode — не единственный путь; это **самый аккуратный**. Для чистого VPS «под ключ под CHR» достаточно обычного SSH в Linux.

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

## Первый вход, обновление прошивки и лицензия

1. Подключитесь по **Winbox** или **SSH** на `YOUR_PUBLIC_IP`.
2. Логин по умолчанию: **`admin`**, пароль — **пустой** (сразу смените).

### Обновить RouterOS до последней версии (до активации лицензии!)

Сразу после первого входа, **пока лицензия ещё не активирована**, обновите прошивку:

**Winbox → Quick Set** → кнопка **Check for updates** → установить последнюю доступную версию → дождаться перезагрузки.

Альтернатива через CLI:

```routeros
/system package update
set channel=stable
check-for-updates once
# если status: New version is available:
/system package update
install
```

> **Важно:** обновление делайте **до** trial P1/P10. После активации лицензии на CHR **upgrade прошивки больше недоступен** — останетесь на той версии, которая стоит в момент активации.

### Активация trial P1 / P10

3. Откройте **System → License**.

В разделе лицензии введите **свои** учётные данные аккаунта MikroTik (создайте на [mikrotik.com](https://mikrotik.com) если нет):

| Поле | Что указать |
|------|-------------|
| **Login (email)** | `YOUR_MIKROTIK_ACCOUNT@example.com` |
| **Password** | `YOUR_MIKROTIK_PASSWORD` |

Нажмите **Start trial** и выберите уровень **P1** или **P10** (зависит от лимитов CHR у MikroTik на момент активации).

### Что важно знать про лицензию

- После окончания trial **лицензия сохраняется на этом CHR** и продолжает работать **без ограничения по времени** для уже активированного уровня (P1/P10).
- Это практичный способ получить полноценный CHR на своём VPS без покупки perpetual license — условно «MikroTik для тех, кто поднимает свой роутер на VPS».
- **Ограничение:** после активации trial на таком CHR **нельзя обновлять RouterOS** — прошивка фиксируется на версии, которая была **на момент активации лицензии**. Поэтому сначала **Check for updates**, потом **Start trial**.

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

Guide for installing **Cloud Hosted Router (CHR)** on a VPS with Linux (Debian/Ubuntu, etc.).

Works when the provider gives you **eth0**, a static public IP, and a default gateway.

> **Warning:** `dd` overwrites the disk (`/dev/sda`). Confirm server and disk before running.

---

## Do you need recovery mode?

**Not strictly** — you need a **Linux environment with disk access** to run `wget`, `mount`, and `dd`.

| Method | Notes |
|--------|--------|
| **Provider rescue/recovery** | Recommended: disk not used by a live root FS, lower risk |
| **Regular SSH into Linux VPS** | **Works too** — same commands; you intentionally wipe the current OS and reboot after `dd` |
| **Marketplace / ready-made image** | Some hosts offer one-click CHR — no manual `dd` |
| **Boot from ISO** | If the provider supports ISO boot — load CHR image directly |

### SSH without recovery — when it’s fine

If the VPS already runs Linux and you **plan to replace it entirely**, SSH in and run the steps below **without** switching to provider rescue. Many people do this.

Caveats:

- Root filesystem is **mounted from the same disk** you overwrite — theoretically riskier than rescue; in practice fine for a one-shot reinstall.
- **Reboot is mandatory** after `dd` — the old OS will not boot again.
- Rescue is safer when the disk has **data you care about** or multiple partitions.

**Bottom line:** recovery mode is not the only way — it’s the **cleanest**. For a fresh VPS dedicated to CHR, plain SSH into Linux is enough.

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

## First login, firmware update, and license

1. Connect via **Winbox** or **SSH** to `YOUR_PUBLIC_IP`.
2. Default login: **`admin`**, password **empty** (change immediately).

### Update RouterOS to the latest version (before license activation!)

Right after first login, **before activating the license**, update firmware:

**Winbox → Quick Set** → **Check for updates** → install the latest available version → wait for reboot.

CLI alternative:

```routeros
/system package update
set channel=stable
check-for-updates once
# if status: New version is available:
/system package update
install
```

> **Important:** update **before** starting P1/P10 trial. After license activation, **firmware upgrade is no longer available** on this CHR — you stay on whatever version was installed at activation time.

### P1 / P10 trial activation

3. Open **System → License**.

Enter **your** MikroTik account credentials ([mikrotik.com](https://mikrotik.com)):

| Field | Value |
|-------|--------|
| **Login (email)** | `YOUR_MIKROTIK_ACCOUNT@example.com` |
| **Password** | `YOUR_MIKROTIK_PASSWORD` |

Click **Start trial** and choose **P1** or **P10**.

### License notes

- After the trial ends, the **license remains on this CHR** and keeps working **without a time limit** for the activated tier (P1/P10).
- Practical way to run a full CHR on your own VPS without buying a perpetual license.
- **Limitation:** after trial activation you **cannot upgrade RouterOS** — firmware is locked to the version that was running **at license activation**. Run **Check for updates** first, then **Start trial**.

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
