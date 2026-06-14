# Установка MikroTik CHR на VPS (UEFI и Legacy)

Инструкция для установки **Cloud Hosted Router (CHR)** на VPS/облачный сервер с Linux (Debian/Ubuntu и аналоги).

Подходит, если провайдер отдаёт VPS со **статическим публичным IP** и шлюзом по умолчанию (один сетевой интерфейс к интернету).

> **Внимание:** команды `dd` перезаписывают **весь системный диск** (см. ниже, как его определить). Убедитесь, что работаете на нужном сервере и с нужным устройством.

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
inet YOUR_PUBLIC_IP/32 scope global YOUR_LINUX_IF

# ip r
default via YOUR_GATEWAY dev YOUR_LINUX_IF
```

| Параметр | Откуда взять | Пример |
|----------|--------------|--------|
| `YOUR_PUBLIC_IP` | `ip a` — адрес на интерфейсе с default route | `203.0.113.10` |
| `YOUR_GATEWAY` | `ip r` → `default via` | `172.31.1.1` |
| `YOUR_LINUX_IF` | `ip a` / `ip r` — имя в **Linux** (rescue) | см. ниже |

### Имя интерфейса (не везде `eth0`!)

В **Linux** (rescue/SSH) имя зависит от провайдера и драйвера:

| Имя | Где встречается |
|-----|-----------------|
| `eth0` | классика, часть KVM |
| `ens3`, `ens18` | systemd predictable names (Proxmox, OVH, …) |
| `enp0s3`, `eno1` | PCI-имена |
| `enX0` | некоторые облака (Azure-подобные) |

Смотрите вывод `ip a` — интерфейс с **вашим публичным IP** и через который идёт default route.

> **Важно для `autorun.scr`:** в Linux интерфейс может называться `ens3`, а в **RouterOS CHR** первый NIC — обычно **`ether1`**, не `eth0` и не `ens3`.  
> В `autorun.scr` указывайте **`YOUR_ROS_IF`** — чаще всего `ether1`. Если после установки сети нет — зайдите в консоль провайдера и проверьте `/interface print`.

| Контекст | Плейсхолдер | Типичное значение на CHR |
|----------|-------------|--------------------------|
| Linux (`ip a`) | `YOUR_LINUX_IF` | `eth0`, `ens3`, … |
| RouterOS (`autorun.scr`) | `YOUR_ROS_IF` | **`ether1`** |

### Определить системный диск (не всегда `/dev/sda`!)

Имя диска **зависит от провайдера и типа виртуализации**:

| Устройство | Где встречается |
|------------|-----------------|
| `/dev/sda` | Классические VPS, часть KVM |
| `/dev/vda` | VirtIO (KVM, Proxmox, многие облака) |
| `/dev/nvme0n1` | NVMe-диски |
| `/dev/xvda` | Старый Xen |

Перед `dd` **обязательно** проверьте:

```bash
lsblk
df -h /
```

Корень `/` смонтирован с раздела вроде `/dev/vda1` → писать нужно на **`/dev/vda`** (whole disk), не на раздел.

Пример `lsblk`:

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    252:0    0   20G  0 disk
└─vda1 252:1    0   20G  0 part /
```

Здесь: `dd ... of=/dev/vda`

Дополнительно (чтобы не перепутать диск):

```bash
ls -l /dev/disk/by-id/
```

Во всех командах ниже **`YOUR_DISK`** — ваше устройство целиком (`/dev/vda`, `/dev/sda`, `/dev/nvme0n1`, …).

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

Содержимое (подставьте **свои** IP, шлюз и интерфейс **RouterOS**):

```routeros
/ip address
add address=YOUR_PUBLIC_IP/32 interface=YOUR_ROS_IF network=YOUR_PUBLIC_IP
/ip route
add gateway=YOUR_GATEWAY
```

Пример (на CHR первый интерфейс чаще **`ether1`**):

```routeros
/ip address
add address=203.0.113.10/32 interface=ether1 network=203.0.113.10
/ip route
add gateway=172.31.1.1
```

### 5. Размонтировать и записать на диск

```bash
umount /mnt
echo u > /proc/sysrq-trigger
dd if=chr-uefi-fat.raw of=YOUR_DISK bs=4M oflag=sync
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
dd if=chr-7.16.1.img of=YOUR_DISK bs=4M oflag=sync
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

### Что важно знать про лицензию («MikroTik для бедных»)

По правилам MikroTik для CHR:

- **60 дней** — пробная версия с **полным доступом** к функционалу выбранного уровня (P1 / P10).
- **После окончания trial**, если **платную лицензию не покупали**, CHR **продолжает работать** на том же уровне — маршрутизация, firewall, VPN и остальное **не отключаются**.
- **Единственное жёсткое ограничение** без покупки: **нельзя обновлять RouterOS** (ни во время trial после привязки, ни после его окончания без perpetual license).

Практический итог для VPS:

| Этап | Что получаете |
|------|----------------|
| Trial 60 дней | Полный CHR бесплатно |
| После trial без покупки | CHR **работает дальше**, но прошивка **заморожена** |
| Покупка perpetual | Можно снова обновляться |

Это удобный способ поднять **полноценный MikroTik на своём VPS** без perpetual license — условно **«MikroTik для бедных»**: один раз активировали trial, обновили прошивку **до** `Start trial`, дальше пользуетесь годами на фиксированной версии.

> **Порядок:** сначала **Quick Set → Check for updates** (последняя stable), **потом** **Start trial** — иначе останетесь на старой прошивке из образа навсегда.

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

Защита от брутфорса по логам — см. [CHR_MIKROTIK_FAIL2BAN](./mikrotik-fail2ban).

---

## Troubleshooting

| Проблема | Что проверить |
|----------|----------------|
| После reboot нет сети | `autorun.scr`: IP, gateway; **`YOUR_ROS_IF`** (часто `ether1`, не имя из Linux) |
| VPS не грузится после dd | Неверный `YOUR_DISK` или UEFI/Legacy — `lsblk` перед dd; другой образ |
| Не монтируется образ | Пересчитать `offset` через `fdisk -lu` |
| Winbox не подключается | Firewall провайдера, порт 8291; временно `/ip service set winbox disabled=no` |

---

---

# Installing MikroTik CHR on a VPS (UEFI and Legacy)

Guide for installing **Cloud Hosted Router (CHR)** on a VPS with Linux (Debian/Ubuntu, etc.).

Works when the provider gives you a VPS with a **static public IP** and default gateway (single NIC to the internet).

> **Warning:** `dd` overwrites the **entire system disk** (see below how to identify it). Confirm server and device before running.

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
inet YOUR_PUBLIC_IP/32 scope global YOUR_LINUX_IF

# ip r
default via YOUR_GATEWAY dev YOUR_LINUX_IF
```

| Parameter | Source | Example |
|-----------|--------|---------|
| `YOUR_PUBLIC_IP` | `ip a` — on interface with default route | `203.0.113.10` |
| `YOUR_GATEWAY` | `ip r` → `default via` | `172.31.1.1` |
| `YOUR_LINUX_IF` | `ip a` / `ip r` — name in **Linux** (rescue) | see below |

### Interface name (not always `eth0`!)

In **Linux** (rescue/SSH), naming depends on provider and driver:

| Name | Common on |
|------|-----------|
| `eth0` | classic VPS, some KVM |
| `ens3`, `ens18` | predictable names (Proxmox, OVH, …) |
| `enp0s3`, `eno1` | PCI-based names |
| `enX0` | some clouds |

Use `ip a` — the interface with **your public IP** and the default route.

> **Important for `autorun.scr`:** Linux may show `ens3`, but **RouterOS CHR** first NIC is usually **`ether1`**, not `eth0` or `ens3`.  
> Use **`YOUR_ROS_IF`** in `autorun.scr` — typically `ether1`. If no network after install, use provider console and run `/interface print`.

| Context | Placeholder | Typical on CHR |
|---------|-------------|----------------|
| Linux (`ip a`) | `YOUR_LINUX_IF` | `eth0`, `ens3`, … |
| RouterOS (`autorun.scr`) | `YOUR_ROS_IF` | **`ether1`** |

### Find the system disk (not always `/dev/sda`!)

Device name **depends on provider and hypervisor**:

| Device | Common on |
|--------|-----------|
| `/dev/sda` | Classic VPS, some KVM |
| `/dev/vda` | VirtIO (KVM, Proxmox, many clouds) |
| `/dev/nvme0n1` | NVMe disks |
| `/dev/xvda` | Legacy Xen |

Before `dd`, **always** check:

```bash
lsblk
df -h /
```

If `/` is on `/dev/vda1`, write to **`/dev/vda`** (whole disk), not the partition.

Example `lsblk`:

```text
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    252:0    0   20G  0 disk
└─vda1 252:1    0   20G  0 part /
```

Here: `dd ... of=/dev/vda`

Optional:

```bash
ls -l /dev/disk/by-id/
```

In all commands below, **`YOUR_DISK`** is the whole device (`/dev/vda`, `/dev/sda`, `/dev/nvme0n1`, …).

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
add address=YOUR_PUBLIC_IP/32 interface=YOUR_ROS_IF network=YOUR_PUBLIC_IP
/ip route
add gateway=YOUR_GATEWAY
```

Example (on CHR first interface is usually **`ether1`**):

```routeros
/ip address
add address=203.0.113.10/32 interface=ether1 network=203.0.113.10
/ip route
add gateway=172.31.1.1
```

### 5. Unmount and write to disk

```bash
umount /mnt
echo u > /proc/sysrq-trigger
dd if=chr-uefi-fat.raw of=YOUR_DISK bs=4M oflag=sync
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
dd if=chr-7.16.1.img of=YOUR_DISK bs=4M oflag=sync
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

### License notes (“budget MikroTik”)

MikroTik CHR licensing in practice:

- **60 days** — trial with **full access** to the selected tier (P1 / P10).
- **After trial ends**, if you **did not buy** a perpetual license, CHR **keeps running** at that tier — routing, firewall, VPN, etc. **stay enabled**.
- **Main limitation** without purchase: **no RouterOS upgrades** (during trial after activation, and after trial without a paid license).

Practical outcome on a VPS:

| Stage | What you get |
|-------|----------------|
| 60-day trial | Full CHR for free |
| After trial, no purchase | CHR **still works**, firmware **frozen** |
| Perpetual license purchase | Upgrades allowed again |

A practical way to run a **full MikroTik on your own VPS** without buying perpetual — **“budget MikroTik”**: activate trial once, upgrade firmware **before** `Start trial`, then use it for years on a fixed version.

> **Order matters:** **Quick Set → Check for updates** first (latest stable), **then** **Start trial** — otherwise you stay on the old image version forever.

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

Log-based brute-force ban — see [CHR_MIKROTIK_FAIL2BAN](https://github.com/helloworld401/mikrotik-fail2ban).


---

## Troubleshooting

| Issue | Check |
|-------|--------|
| No network after reboot | `autorun.scr`: IP, gateway; **`YOUR_ROS_IF`** (often `ether1`, not the Linux name) |
| VPS won't boot after dd | Wrong `YOUR_DISK` or UEFI/Legacy — run `lsblk` before dd; try other image |
| Mount fails | Recalculate `offset` with `fdisk -lu` |
| Winbox won't connect | Provider firewall, port 8291 |

---

## License (this document)

MIT — see `LICENSE` in the repository root (when published).
