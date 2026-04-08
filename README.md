# 🛡️ Настройка FreeRADIUS + rsyslog — Олимпиада НТО по сетевой безопасности

> Памятка для Роли 4 «Инженер по защите и восстановлению»  
> Задача: настроить сервер SRV (RADIUS + Syslog) — **12 баллов из 90**

---

## 📋 Топология сети (экзамен)

```
ISP-1 ── R1 ── SW_Agg1 ── SW_Acc1 ── PC1
              |          └── SW_Acc2 ── PC2
              └── SW_Agg2
                   └── (филиал R2)
```

### IP-адресация по заданию

| Устройство | VLAN | Сеть | IP |
|---|---|---|---|
| SRV (сервер) | vlan 100 | 100.1.2.0/24 | 100.1.2.2 |
| R1 | vlan 50 | 50.1.2.0/24 | 50.1.2.1 |
| R2 | vlan 50 | 50.1.2.0/24 | 50.1.2.100 |
| SW_Agg1 | vlan 50 | 50.1.2.0/24 | 50.1.2.2 |
| SW_Agg2 | vlan 50 | 50.1.2.0/24 | 50.1.2.3 |
| SW_Acc1 | vlan 50 | 50.1.2.0/24 | 50.1.2.4 |
| SW_Acc2 | vlan 50 | 50.1.2.0/24 | 50.1.2.5 |
| PC1 | vlan 10 | 110.1.2.0/24 | 110.1.2.x |
| PC2 | vlan 10 | 110.1.2.0/24 | 110.1.2.x |

---

## 🎯 Что нужно настроить на сервере

| Задача | Баллы | Файл |
|---|---|---|
| RADIUS сервер (dot1x) | 4 | `/etc/freeradius/3.0/` |
| SSH-вход через RADIUS | 5 | `/etc/pam.d/sshd` |
| Syslog раздельные логи | 6 | `/etc/rsyslog.d/` |

---

## ⚡ БЫСТРЫЙ СТАРТ (всё с нуля за 10 минут)

```bash
# 1. Проверить что установлено
dpkg -l | grep -E "freeradius|rsyslog|libpam-radius"

# 2. Если не установлено:
sudo apt update
sudo apt install -y freeradius freeradius-utils libpam-radius-auth rsyslog openssh-server
```

---

## 1️⃣ FREERADIUS

### 1.1 Пользователи

```bash
sudo nano /etc/freeradius/3.0/users
```

Добавить в **самое начало** файла:

```
admin   Cleartext-Password := "Admin123"
user1   Cleartext-Password := "User1pass"
user2   Cleartext-Password := "User2pass"
```

> ⚠️ Важно: вторая строка и далее — без отступа, каждый пользователь с новой строки

### 1.2 Клиенты (сетевое оборудование)

```bash
sudo nano /etc/freeradius/3.0/clients.conf
```

Добавить в **конец** файла:

```
# Для PAM (SSH через RADIUS на самом сервере)
client localhost_pam {
    ipaddr = 127.0.1.1
    secret = testing123
    shortname = localhost_pam
}

# R1 — маршрутизатор головного офиса
client R1 {
    ipaddr = 50.1.2.1
    secret = radius_secret123
    shortname = R1
}

# R2 — маршрутизатор филиала
client R2 {
    ipaddr = 50.1.2.100
    secret = radius_secret123
    shortname = R2
}

# SW_Agg1 — агрегирующий коммутатор 1
client SW_Agg1 {
    ipaddr = 50.1.2.2
    secret = radius_secret123
    shortname = SW_Agg1
}

# SW_Agg2 — агрегирующий коммутатор 2
client SW_Agg2 {
    ipaddr = 50.1.2.3
    secret = radius_secret123
    shortname = SW_Agg2
}

# SW_Acc1 — коммутатор доступа 1
client SW_Acc1 {
    ipaddr = 50.1.2.4
    secret = radius_secret123
    shortname = SW_Acc1
}

# SW_Acc2 — коммутатор доступа 2
client SW_Acc2 {
    ipaddr = 50.1.2.5
    secret = radius_secret123
    shortname = SW_Acc2
}
```

### 1.3 Запуск и проверка

```bash
# Остановить и запустить в debug-режиме
sudo systemctl stop freeradius
sudo freeradius -X

# В другом терминале — проверить
radtest admin Admin123 127.0.0.1 0 testing123
# → Received Access-Accept ✅

radtest admin wrongpass 127.0.0.1 0 testing123
# → Received Access-Reject ✅

# Остановить debug и запустить нормально
# Ctrl+C
sudo systemctl start freeradius
sudo systemctl enable freeradius

# Проверить статус
sudo systemctl status freeradius
sudo ss -ulnp | grep 1812
```

---

## 2️⃣ RSYSLOG (раздельные логи)

### 2.1 Включить приём по сети

```bash
sudo nano /etc/rsyslog.conf
```

Найти и раскомментировать (убрать `#`):

```
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

> ⚠️ Убедись что рядом нет лишнего текста на той же строке!

### 2.2 Раздельные файлы по устройствам

```bash
sudo nano /etc/rsyslog.d/network-devices.conf
```

```
# Шаблон: один файл на устройство по IP
template(name="PerDevice" type="string"
         string="/var/log/network/%FROMHOST-IP%.log")

# R1
if $fromhost-ip == '50.1.2.1' then {
    action(type="omfile" DynaFile="PerDevice")
    stop
}

# R2
if $fromhost-ip == '50.1.2.100' then {
    action(type="omfile" DynaFile="PerDevice")
    stop
}

# SW_Agg1
if $fromhost-ip == '50.1.2.2' then {
    action(type="omfile" DynaFile="PerDevice")
    stop
}

# SW_Agg2
if $fromhost-ip == '50.1.2.3' then {
    action(type="omfile" DynaFile="PerDevice")
    stop
}

# SW_Acc1
if $fromhost-ip == '50.1.2.4' then {
    action(type="omfile" DynaFile="PerDevice")
    stop
}

# SW_Acc2
if $fromhost-ip == '50.1.2.5' then {
    action(type="omfile" DynaFile="PerDevice")
    stop
}
```

### 2.3 Создать папку и перезапустить

```bash
sudo mkdir -p /var/log/network
sudo chown root:root /var/log/network
sudo chmod 755 /var/log/network
sudo systemctl restart rsyslog
```

### 2.4 Проверка

```bash
# Убедиться что слушает порт 514
sudo ss -ulnp | grep 514

# Проверить файлы после того как оборудование начнёт слать логи
ls /var/log/network/
# Должно быть:
# 50.1.2.1.log   ← R1
# 50.1.2.2.log   ← SW_Agg1
# 50.1.2.3.log   ← SW_Agg2
# и т.д.

cat /var/log/network/50.1.2.1.log
```

---

## 3️⃣ SSH ЧЕРЕЗ RADIUS (PAM)

### 3.1 Настроить RADIUS сервер для PAM

```bash
sudo nano /etc/pam_radius_auth.conf
```

Оставить **только одну строку** (остальные закомментировать или удалить):

```
127.0.0.1    testing123    3
```

### 3.2 Подключить RADIUS к SSH

```bash
sudo nano /etc/pam.d/sshd
```

Добавить **первой строкой** в самом начале:

```
auth    sufficient    /usr/lib/aarch64-linux-gnu/security/pam_radius_auth.so
```

> ⚠️ Путь к модулю зависит от архитектуры:
> - ARM (M1/M2 Mac, Eltex): `/usr/lib/aarch64-linux-gnu/security/pam_radius_auth.so`
> - x86_64 (обычный ПК): `/usr/lib/x86_64-linux-gnu/security/pam_radius_auth.so`
>
> Проверить путь: `find /usr/lib -name "pam_radius_auth.so"`

### 3.3 Разрешить парольный вход SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Найти и установить:

```
PasswordAuthentication yes
ChallengeResponseAuthentication yes
UsePAM yes
```

### 3.4 Создать системного пользователя

```bash
# Создать пользователя без пароля (пароль через RADIUS)
sudo useradd -m admin
```

### 3.5 Перезапустить SSH и проверить

```bash
sudo systemctl restart freeradius
sudo systemctl restart ssh

# Проверить с другой машины:
ssh admin@100.1.2.2
# Пароль: Admin123 (из файла users)
# → Должен войти ✅
```

---

## 4️⃣ НАСТРОЙКА НА ОБОРУДОВАНИИ ELTEX

### На роутере ESR (R1, R2) — RADIUS для SSH

```
# Войти в режим конфигурации
configure

# Указать RADIUS сервер
radius-server host 100.1.2.2 auth-port 1812 key radius_secret123

# Включить RADIUS аутентификацию для SSH
aaa authentication login default radius local

# Сохранить
commit
```

### На коммутаторе MES (SW_Agg1, SW_Agg2, SW_Acc1, SW_Acc2) — RADIUS для SSH

```
# Указать RADIUS сервер
radius-server host 100.1.2.2 auth-port 1812 key radius_secret123

# Включить RADIUS аутентификацию
aaa authentication login default radius local

# Сохранить
write memory
```

### На всём оборудовании — отправка логов на syslog

```
# На ESR (роутер)
configure
syslog host 100.1.2.2
syslog facility local7
syslog severity info
commit

# На MES (коммутатор)
logging host 100.1.2.2
logging facility local7
write memory
```

### На коммутаторе MES — 802.1x (dot1x)

```
# Включить dot1x глобально
dot1x system-auth-control

# Указать RADIUS сервер
radius-server host 100.1.2.2 auth-port 1812 key radius_secret123

# Настроить на каждом порту доступа
interface gi 1/0/1
  dot1x port-control auto
  exit

interface gi 1/0/2
  dot1x port-control auto
  exit

write memory
```

---

## ✅ ФИНАЛЬНЫЙ ЧЕКЛИСТ

```bash
# === FREERADIUS ===
sudo systemctl status freeradius          # active (running) ✅
sudo ss -ulnp | grep 1812                 # слушает порт 1812 ✅
radtest admin Admin123 127.0.0.1 0 testing123  # Access-Accept ✅
radtest admin wrong 127.0.0.1 0 testing123     # Access-Reject ✅

# === RSYSLOG ===
sudo systemctl status rsyslog             # active (running) ✅
sudo ss -ulnp | grep 514                  # слушает порт 514 ✅
ls /var/log/network/                      # файлы по IP ✅

# === SSH ЧЕРЕЗ RADIUS ===
ssh admin@100.1.2.2                       # входит с паролем из RADIUS ✅

# === ПРОВЕРКА С ОБОРУДОВАНИЯ ===
# После настройки Eltex:
# - SSH на коммутатор → пароль проверяется через FreeRADIUS
# - В /var/log/network/ появляются файлы от каждого устройства
```

---

## 🚨 ТИПИЧНЫЕ ОШИБКИ И РЕШЕНИЯ

| Проблема | Причина | Решение |
|---|---|---|
| `Access-Reject` при правильном пароле | Пробелы вместо TAB в файле users | Перепечатать пользователя без пробелов перед паролем |
| PAM не работает | Неверный путь к модулю | `find /usr/lib -name "pam_radius_auth.so"` |
| SSH не пускает | PAM не получает Accept от RADIUS | Добавить `127.0.1.1` в clients.conf |
| Логи не пишутся | Нет папки `/var/log/network` | `sudo mkdir -p /var/log/network` |
| rsyslog не слушает 514 | Не раскомментированы imudp/imtcp | Проверить `/etc/rsyslog.conf` |
| Оборудование не шлёт логи | Неверный IP сервера или порт | Проверить `syslog host` на оборудовании |

---

## 📁 Структура файлов конфигурации

```
/etc/freeradius/3.0/
├── users                    ← пользователи (admin, user1, user2)
├── clients.conf             ← клиенты (R1, R2, SW_Agg1-2, SW_Acc1-2)
└── mods-enabled/
    └── eap                  ← dot1x (уже включён по умолчанию)

/etc/rsyslog.conf            ← включить imudp/imtcp
/etc/rsyslog.d/
└── network-devices.conf     ← раздельные файлы по IP

/etc/pam.d/sshd              ← SSH через RADIUS
/etc/pam_radius_auth.conf    ← адрес RADIUS сервера для PAM
/etc/ssh/sshd_config         ← PasswordAuthentication yes

/var/log/network/
├── 50.1.2.1.log             ← логи R1
├── 50.1.2.2.log             ← логи SW_Agg1
├── 50.1.2.3.log             ← логи SW_Agg2
├── 50.1.2.4.log             ← логи SW_Acc1
└── 50.1.2.5.log             ← логи SW_Acc2
```

---

## 🔑 Секреты и пароли (для экзамена уточнить у организаторов)

| Параметр | Значение |
|---|---|
| RADIUS secret (PAM) | `testing123` |
| RADIUS secret (оборудование) | `radius_secret123` |
| Пароль пользователя admin | `Admin123` |
| IP сервера SRV | `100.1.2.2` |
| RADIUS порт | `1812` |
| Syslog порт | `514` |

> ⚠️ На экзамене secret должен совпадать на сервере (clients.conf) и на оборудовании Eltex!

---

## 📝 Порядок работы на экзамене (Роль 4)

1. **0:30** — Получить доступ к серверу SRV
2. **0:35** — Настроить FreeRADIUS (users + clients.conf) — 5 минут
3. **0:40** — Настроить rsyslog (network-devices.conf) — 5 минут
4. **0:45** — Настроить PAM для SSH — 5 минут
5. **0:50** — Перейти к оборудованию Eltex — прописать RADIUS и syslog
6. **1:15** — Проверить что логи идут в `/var/log/network/`
7. **1:20** — Проверить SSH через RADIUS на оборудовании
8. **1:30** — Сохранить конфиги (`write memory` / `commit`)
