# 03. Харднинг: firewall (ufw) и fail2ban

## Цель

- Firewall на уровне ОС: все входящие соединения запрещены, кроме SSH; все исходящие разрешены.
- fail2ban: временно блокировать IP, с которого несколько раз подряд не смогли войти.

## Что сделал

### 1. Firewall (ufw)

Порядок важен: **сначала разрешить SSH, потом включать firewall**.

```bash
sudo ufw default deny incoming     # по умолчанию запрещаем все входящие
sudo ufw default allow outgoing    # по умолчанию разрешаем все исходящие
sudo ufw allow OpenSSH             # разрешаем входящие на 22/tcp (профиль приложения OpenSSH)
sudo ufw enable                    # включаем firewall
```

```console
eisutt@devops-01:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
```

- Правило `OpenSSH ALLOW Anywhere` разрешает **входящие** соединения на порт 22 с любого адреса.
  Исходящий трафик разрешён целиком политикой `default allow outgoing`, отдельные правила для него не нужны.
- `(v6)`: ufw автоматически создаёт такое же правило для IPv6.
- Список профилей приложений: `sudo ufw app list`, содержимое профиля: `sudo ufw app info OpenSSH`.

Подробный вывод, в нём видны политики по умолчанию:

```console
eisutt@devops-01:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
```

- `ALLOW IN`: правило для **входящих** соединений.
- `disabled (routed)`: пересылка трафика через сервер (сервер как маршрутизатор) запрещена.
  Это понадобится на неделе Docker: контейнерный трафик идёт именно через маршрутизацию.
- `Logging: on (low)`: заблокированные пакеты пишутся в лог ядра, смотреть их так: `sudo journalctl -k | grep "UFW BLOCK"`.

Вход из нового окна терминала после включения ufw работает.

**Что было бы, если включить ufw без правила для SSH?**
Текущая сессия, скорее всего, не оборвалась бы. В ufw есть правило, которое пропускает пакеты
**уже установленных** соединений (`conntrack ESTABLISHED,RELATED`), и наша сессия к ним относится.
Но любое новое подключение было бы заблокировано. Стоит текущей сессии оборваться (сеть моргнула, ноутбук уснул),
и доступ к серверу пропадает. Вернуть его можно только через серийную консоль YC.
Поэтому ufw при `enable` и предупреждает: `Command may disrupt existing ssh connections`.

### 2. fail2ban

```bash
sudo apt install fail2ban
```

Настройки нельзя писать в `jail.conf`: его перезапишет обновление пакета.
Свои настройки кладутся в `jail.local` или в `jail.d/*.local`, они читаются после `jail.conf` и переопределяют его.
Та же идея, что и с `sshd_config.d/`.

Порядок чтения конфигурации, каждый следующий файл переопределяет предыдущие:

```
jail.conf  →  jail.d/*.conf  →  jail.local  →  jail.d/*.local
```

- `jail.conf` и `jail.d/defaults-debian.conf` — файлы пакета, их не правлю.
  `defaults-debian.conf` называется так, потому что Ubuntu берёт пакеты из Debian вместе с их настройками.
  В нём уже включён jail `sshd` (`enabled = true`), а также заданы `backend = systemd` и `banaction = nftables`.
- `jail.local` — мой файл, только с теми параметрами, которые меняю:

```ini
[sshd]
bantime  = 1h    # на сколько банить IP
findtime = 10m   # окно, в котором считаются неудачные попытки
maxretry = 3     # сколько неудачных попыток в окне findtime приводят к бану
```

Как это работает вместе: **если с одного IP за 10 минут было 3 неудачные попытки, IP банится на 1 час.**
Неудачная попытка — это любой неуспешный вход: неверный ключ, несуществующий пользователь, отказ по `AllowUsers`.

Проверил, что значения применились (аналог `sshd -T` для fail2ban):

```console
eisutt@devops-01:~$ sudo fail2ban-client get sshd maxretry
3
eisutt@devops-01:~$ sudo fail2ban-client get sshd findtime
600
eisutt@devops-01:~$ sudo fail2ban-client get sshd bantime
3600
```

Проверить, что файлы пакета не изменены: `dpkg -V fail2ban`. Пустой вывод означает, что всё совпадает с оригиналом.

Включил автозапуск и запустил сервис:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Можно одной командой: `sudo systemctl enable --now fail2ban`.

Проверил jail `sshd`:

```console
eisutt@devops-01:~$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:	12
|  |- Total failed:	47
|  `- Journal matches:	_SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned:	1
   |- Total banned:	1
   `- Banned IP list:	186.120.179.222
```

- **`Journal matches`:** fail2ban читает события не из файла, а из journald. Знак `+` в фильтре означает **ИЛИ**.
  В Ubuntu сервис называется `ssh.service`, а не `sshd.service`, поэтому записи находятся по второй части: `_COMM=sshd`.
- **Неудачных попыток 47, а забанен всего 1 IP.** Ботнет каждый раз приходит с нового адреса,
  и почти ни один IP не набирает 3 ошибки за 10 минут. См. вопрос ниже.

Через час работы:

```console
eisutt@devops-01:~$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:    10
|  |- Total failed:    79
|  `- Journal matches:    _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned:    0
   |- Total banned:    1
   `- Banned IP list:
```

79 неудачных попыток, и за всё время забанен только 1 IP: его бан уже истёк через час.
Распределённая атака почти не попадает под порог «3 ошибки с одного IP за 10 минут».

Забаненные IP видны и на уровне firewall ядра: `sudo nft list ruleset | grep -A 10 f2b`.

## Проблемы и решения

**Проблема:** сначала скопировал весь `jail.conf` в `jail.local` и поменял значения в `[DEFAULT]`.

**Почему плохо:** копия целиком переопределяет все настройки пакета, и исправления из обновлений до меня не дойдут.
А `[DEFAULT]` действует на все jail'ы, а не только на `sshd`.

---

**Проблема:** потом перенёс настройки в `jail.d/defaults-debian.conf`.

**Почему плохо:** это файл пакета. При обновлении `apt` увидит, что файл изменён, и спросит, какую версию оставить:
либо потеряю свои настройки, либо не получу изменения от мейнтейнеров. В автоматическом обновлении это может повесить процесс.

**Решение:** вернул `defaults-debian.conf` к исходному виду и создал `jail.local` только с секцией `[sshd]` и тремя параметрами.

## Что понял

- Порядок включения firewall важен: сначала правило для SSH, потом `ufw enable`.
- Установленные соединения переживают включение firewall, а новые нет.
- Направление трафика: `ALLOW IN` для SSH означает входящие подключения к серверу. Исходящие разрешены политикой по умолчанию.
- Общее правило для любых конфигов: **файлы пакета не трогать, свои настройки класть в отдельный override-файл
  и только то, что меняешь.** Так устроены sshd (`sshd_config.d/`), fail2ban (`jail.local`), sudo (`sudoers.d/`).
- Ubuntu основана на Debian, поэтому в именах файлов и в документации часто встречается «debian».

## Вопросы

**1. Насколько fail2ban помогает против ботнета, который каждый раз приходит с нового IP?**

Слабо. fail2ban считает ошибки **по каждому IP отдельно**, а распределённая атака размазывает попытки по сотням адресов.

На самом деле сервер защищает:
- **вход только по ключу.** Подобрать ключ ed25519 перебором практически невозможно, так что атака бессмысленна,
  сколько бы адресов у ботнета ни было;
- **запрет входа под root и `AllowUsers`.** Атакующему нужно угадать ещё и имя пользователя.

fail2ban — дополнительный слой. Он снижает шум в логах и нагрузку, отсекает простых сканеров с одного IP
и пригодится для сервисов, где пароли всё-таки есть, например веб-админок.

Ещё сильнее можно сузить доступ:
- открыть порт 22 только для своего IP (в ufw или в группе безопасности YC);
- ходить на серверы через VPN или bastion-хост, не выставляя SSH в интернет вообще.
