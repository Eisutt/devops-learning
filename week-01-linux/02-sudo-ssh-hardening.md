# 02. Харднинг: sudo и SSH

## Цель

- Разобраться, откуда у пользователя права sudo, и включить запрос пароля.
- Закрыть SSH: запретить вход под root и по паролю, разрешить вход только своему пользователю,
  ограничить число попыток аутентификации.

## Что сделал

### 1. Откуда у пользователя sudo

```console
eisutt@devops-01:~$ id
uid=1000(eisutt) gid=1000(eisutt) groups=1000(eisutt),1001(google-sudoers)
```

Пользователь состоит в группе `google-sudoers`. Её создаёт гостевой агент Yandex Cloud
(он основан на агенте Google Cloud, отсюда название).

Права sudo выдаются в двух местах, и **оба правила разрешают sudo без пароля**:

| Где | Правило | Кому |
|-----|---------|------|
| `/etc/sudoers.d/90-cloud-init-users` (создал cloud-init при первой загрузке) | `eisutt ALL=(ALL) NOPASSWD:ALL` | пользователю `eisutt` |
| правило для группы `google-sudoers` (создал гостевой агент YC) | `%google-sudoers ... NOPASSWD:ALL` | всем участникам группы |

Как читать правило `eisutt ALL=(ALL) NOPASSWD:ALL`:

```
eisutt    ALL     =(ALL)          NOPASSWD:     ALL
  │        │         │               │           │
кто    на каком   от имени       без пароля   какие
       хосте      какого                      команды
                  пользователя
```

Файлы sudoers смотрел и редактировал через `visudo`: перед сохранением он проверяет синтаксис.
Ошибка в sudoers может полностью сломать sudo, и без root-доступа починить его будет уже нечем.

### 2. Включил запрос пароля для sudo

Решил, что sudo должен спрашивать пароль: если кто-то получит доступ к сессии пользователя,
он не получит root автоматически.

1. Задал пользователю пароль:
   ```bash
   sudo passwd eisutt
   ```
2. Через `sudo visudo /etc/sudoers.d/90-cloud-init-users` убрал `NOPASSWD:`:
   ```console
   eisutt@devops-01:~$ sudo cat /etc/sudoers.d/90-cloud-init-users
   # Created by cloud-init v. 26.1-0ubuntu1~24.04.1 on Thu, 29 Oct 2026 13:10:02 +0000

   # User rules for eisutt
   eisutt ALL=(ALL) ALL
   ```
3. sudo всё равно не спрашивал пароль: оставалось правило `NOPASSWD` для группы.
   Удалил пользователя из группы:
   ```bash
   sudo gpasswd -d eisutt google-sudoers
   ```
4. После этого sudo начал запрашивать пароль.

### 3. Ошибка с паролем root

Сначала задавал пароль командой `sudo passwd $USERS`, с опечаткой: правильно `$USER`.
Переменная `$USERS` пустая, поэтому выполнилась `sudo passwd` без аргументов,
а она меняет пароль пользователя, от имени которого запущена, то есть **root**.

Заблокировал пароль root и проверил статус:

```console
eisutt@devops-01:~$ sudo passwd -l root
eisutt@devops-01:~$ sudo passwd -S root
root L 2026-11-04 0 99999 7 -1
eisutt@devops-01:~$ sudo passwd -S eisutt
eisutt P 2026-11-04 0 99999 7 -1
```

Второе поле — статус пароля: `P` задан, `L` заблокирован, `NP` отсутствует.
С заблокированным паролем root нельзя войти через `su` или консоль, но `sudo` продолжает работать:
он спрашивает пароль пользователя `eisutt`, а не root.

Вывод: перед командой с переменной стоит проверить её значение (`echo $USERS`).
Пустая переменная может молча изменить смысл команды.

### 4. Настройка SSH

Сделал бэкап конфига, чтобы можно было откатиться:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Свои настройки вынес в отдельный drop-in файл `/etc/ssh/sshd_config.d/00-hardening.conf`,
а основной `sshd_config` оставил стандартным:

```
maxauthtries 3                    # максимум попыток аутентификации за одно подключение
permitrootlogin no                # запрет входа под root
passwordauthentication no         # запрет входа по паролю, только ключи
kbdinteractiveauthentication no   # через keyboard-interactive тоже можно войти по паролю
allowusers eisutt                 # входить может только этот пользователь
x11forwarding no                  # графики на сервере нет, лишняя поверхность атаки
```

Почему отдельный файл, а не правка `sshd_config`:
- `Include /etc/ssh/sshd_config.d/*.conf` стоит в начале `sshd_config`, а для обычных параметров
  **побеждает первое значение**. Значит, настройки из `sshd_config.d/` перебивают основной файл.
- Файлы в каталоге читаются в алфавитном порядке, поэтому `00-hardening.conf` идёт первым и побеждает остальные.
- При обновлении пакета openssh мои настройки не конфликтуют с новой версией `sshd_config`.
- Все изменения видны в одном коротком файле.

В каталоге уже были файлы от облачного образа:

```console
eisutt@devops-01:~$ ls -l /etc/ssh/sshd_config.d/
-rw-r--r-- 1 root root   ... 00-hardening.conf
-rw------- 1 root root   26  50-cloud-init.conf
-rw-r--r-- 1 root root   26  60-cloudimg-settings.conf
-r--r--r-- 1 root root   87  sshd_casignature_alhgorithms.conf
eisutt@devops-01:~$ sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
PasswordAuthentication no
eisutt@devops-01:~$ sudo cat /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
PasswordAuthentication no
```

Здесь оба файла запрещают пароль, но в другом образе или с другим cloud-config там вполне может быть
`PasswordAuthentication yes`. Тогда правка одного `sshd_config` **молча не сработала бы**.

**Ошибки по ходу:**
- Сначала правил сам `sshd_config`, а при создании drop-in файла скопировал в него весь `sshd_config` целиком,
  со строкой `Include`. Drop-in должен содержать только изменяемые параметры.
  Основной конфиг вернул из бэкапа: `sudo cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config`.
- Из-за этого `AllowUsers` в `sshd -T` выводился дважды. **Списочные параметры** (`AllowUsers`, `AcceptEnv`,
  `ListenAddress`) не подчиняются правилу «первое побеждает», а **накапливаются** из всех файлов.

Применил настройки:

```console
eisutt@devops-01:~$ sudo systemctl reload ssh.service
eisutt@devops-01:~$ sudo systemctl status ssh.service
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; disabled; preset: enabled)
     Active: active (running) since Wed 2026-11-04 13:45:49 UTC; 36min ago
TriggeredBy: ● ssh.socket
    Process: 3758 ExecReload=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
    Process: 3760 ExecReload=/bin/kill -HUP $MAINPID (code=exited, status=0/SUCCESS)
   Main PID: 3222 (sshd)
```

Что видно в выводе:
- **`ExecReload=/usr/sbin/sshd -t`:** перед перезагрузкой systemd сам проверяет конфиг.
  Если в нём ошибка, reload не выполнится и сервер продолжит работать со старыми настройками.
- **`ExecReload=/bin/kill -HUP`:** reload — это сигнал SIGHUP. sshd перечитывает конфиг,
  а уже открытые сессии не обрываются.
- **`disabled` и `TriggeredBy: ssh.socket`:** в Ubuntu 24.04 SSH запускается через socket activation.
  Порт 22 слушает systemd (`ssh.socket`) и запускает `ssh.service` при первом подключении.
  Поэтому сам сервис может быть `disabled` и при этом работать.

Проверил итоговые значения:

```console
eisutt@devops-01:~$ sudo sshd -T | grep -iE "permitroot|passwordauth|kbdinteractive|allowusers|maxauthtries|x11forwarding"
maxauthtries 3
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
x11forwarding no
allowusers eisutt
```

### 5. Проверка входа

С мака, в новом окне терминала:

```console
❯ ssh -o PubkeyAuthentication=no eisutt@<SERVER_IP>
eisutt@<SERVER_IP>: Permission denied (publickey).

❯ ssh root@<SERVER_IP>
root@<SERVER_IP>: Permission denied (publickey).
```

- В скобках сервер перечисляет методы аутентификации, которые он принимает. Остался только `publickey`.
- Для root сервер не объясняет клиенту, почему отказал. Это сделано специально, чтобы не подсказывать атакующему.
  Настоящая причина видна только в логах сервера: `sudo journalctl -u ssh`.

Обычный вход по ключу из нового окна терминала работает.

### 6. Кто стучится на сервер

В логах за час нашлось около 50 попыток войти под root:

```console
eisutt@devops-01:~$ sudo journalctl -u ssh --since "1 hour ago" | grep root
Nov 04 14:31:58 devops-01 sshd[4027]: User root from 45.135.141.242 not allowed because not listed in AllowUsers
Nov 04 14:31:58 devops-01 sshd[4027]: Connection closed by invalid user root 45.135.141.242 port 50582 [preauth]
Nov 04 14:33:51 devops-01 sshd[4031]: User root from 45.135.141.135 not allowed because not listed in AllowUsers
...
```

- Попытки идут каждые 1–3 минуты, почти все с адресов `45.135.141.x` и `45.135.142.x`.
  Это ботнет, который перебирает пароли и меняет IP внутри одной сети.
- `[preauth]` означает, что соединение закрыто ещё до аутентификации.
  Сработал `AllowUsers`, и до проверки пароля дело не дошло.
- Публичный сервер с открытым портом 22 начинают сканировать в первые минуты после появления в интернете.
  Поэтому вход по паролю и вход под root отключаются сразу.

## Проблемы и решения

**Проблема:** задал пароль пользователю, но sudo всё равно не спрашивал его.

**Причина:** права sudo выдавались в двух местах, в правиле для пользователя и в правиле для группы `google-sudoers`.
Оба правила были с `NOPASSWD`.

**Решение:** убрал `NOPASSWD` из `90-cloud-init-users` и удалил пользователя из группы `google-sudoers`.

---

**Проблема:** случайно задал пароль root.

**Причина:** опечатка в имени переменной (`$USERS` вместо `$USER`), команда выполнилась без аргумента.

**Решение:** заблокировал пароль root через `passwd -l root`, проверил статус через `passwd -S`.

---

**Проблема:** скопировал весь `sshd_config` в drop-in файл, и настройки задублировались.

**Решение:** оставил в `00-hardening.conf` только свои параметры, основной конфиг вернул из бэкапа.

## Что понял

- Права sudo могут выдаваться не только пользователю напрямую, но и через группу.
  Проверять нужно и `id`, и все файлы в `/etc/sudoers.d/`. `sudo -l` показывает все правила, которые применяются к пользователю.
- В sudoers, если подходит несколько правил, **применяется последнее**.
- В sshd наоборот: **применяется первое найденное значение** параметра.
  Исключение составляют списочные параметры вроде `AllowUsers`: они накапливаются.
- Свои настройки sshd лучше класть в `sshd_config.d/00-*.conf` и проверять итог через `sshd -T`.
- Перед reload systemd сам проверяет конфиг SSH через `sshd -t`.
- Пустая переменная в команде может изменить её смысл: `sudo passwd $USERS` сменил пароль root.
- Сервер в интернете начинают атаковать сразу. В логах видно, как ботнет перебирает адреса.

## Вопросы

**1. Если параметр sshd указан в нескольких местах, какое значение применяется?**

Первое найденное. Директива `Include /etc/ssh/sshd_config.d/*.conf` стоит в **самом начале** `sshd_config`,
поэтому файлы из `sshd_config.d/` читаются раньше основного файла и побеждают его.
Файлы внутри каталога читаются в алфавитном порядке, так что `00-hardening.conf` победит `50-cloud-init.conf`.

Отсюда ловушка: если cloud-init или пакет положит в `sshd_config.d/` файл с `PasswordAuthentication yes`,
он молча перебьёт настройку из основного файла. Итоговое значение всегда проверять через `sshd -T`.

**2. Зачем менять sudoers только через `visudo`?**

`visudo` блокирует файл от одновременного редактирования и **проверяет синтаксис перед сохранением**.
Если сохранить sudoers с ошибкой обычным редактором, sudo перестанет работать целиком,
и исправить файл без root-доступа будет невозможно. В облаке спасёт только серийная консоль.
