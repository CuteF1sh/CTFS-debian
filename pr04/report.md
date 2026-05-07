
# ПР №4. Аудит событий: journalctl и auditd

## 1. journalctl — системный журнал

### Что зафиксировал журнал о команде sudo cat /etc/shadow

Лог из journalctl:
Maq 07 16:00:54 debian sudo[3555]: ctfs : TTY=pts/1 ; PWD=/home/ctfs ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow

Разбор полей:
- Maq 07 16:00:54 — дата и время
- debian — хост
- sudo[3555] — процесс и PID
- ctfs — пользователь, выполнивший sudo
- COMMAND=/usr/bin/cat /etc/shadow — команда

### Ошибки в системе с последней загрузки

Вывод journalctl -p err -b:
Ma 07 15:56:44 debian systemd[1468]: Failed to start app-gnome-gnome\x2dkeyring\x2dssh-1624.scope
Ma 07 15:56:46 debian systemd[1468]: Failed to start app-gnome-xdg\x2duser\x2ddirs\x2dkde-1787.scope

Объяснение: графическая оболочка GNOME не смогла запустить пользовательские сервисы (ssh-агент, XDG user dirs). Ошибки некритичны.

### Статистика входов

- Успешных входов SSH: 0
- Неудачных попыток SSH: 0

## 2. auditd — настройка правил

### Применённые правила (sudo auditctl -l)

-w /etc/passwd -p rwa -k auth-files
-w /etc/shadow -p rwa -k auth-files
-w /etc/group -p rwa -k auth-files
-w /etc/sudoers -p rwa -k priv-files
-w /etc/sudoers.d -p rwa -k priv-files
-w /etc/ssh/sshd_config -p rwa -k ssh-config
-a always,exit -F arch=b64 -S open,openat -F success=0 -F key=access-denied
-a always,exit -F arch=b64 -S unlink,unlinkat -F key=file-delete
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid!=0 -F uid!=1 -F key=priv-exec

### Разбор записи SYSCALL

Пример записи из audit.log:
type=SYSCALL auid=ctfs uid=ctfs euid=root comm=sudo exe=/usr/bin/sudo key=watch-passwd

| Поле | Значение | Что означает |
|------|----------|--------------|
| auid | ctfs | Оригинальный пользователь (не меняется при sudo) |
| uid | ctfs | ID пользователя до повышения прав |
| euid | root | Эффективный ID после повышения |
| comm | sudo | Имя команды |
| exe | /usr/bin/sudo | Путь к исполняемому файлу |

## 3. Расследование инцидента

### Хронология действий нарушителя (bob, uid=500)

| Время | Действие | Результат |
|-------|----------|-----------|
| 07.05 16:44:46 | systemctl | Отказ (EINVAL) |
| 07.05 16:44:46 | gpg-agent | Отказ |
| 07.05 16:52:39 | cat /etc/passwd | Успех |
| 07.05 17:29:26 | чтение /etc/sudoers | Успех |

### Как auditd помог расследованию

- Зафиксировал все неудачные попытки доступа (access-denied)
- Отследил обращение к /etc/passwd и /etc/sudoers
- Сохранил auid для идентификации реального пользователя

### Чем auditd лучше journalctl для аудита безопасности

| Характеристика | auditd | journalctl |
|----------------|--------|------------|
| Уровень фиксации | Системные вызовы (ядро) | Прикладные логи |
| Отслеживание отказа в доступе | Да | Нет |
| Фиксация auid | Да | Нет |

## 4. Связь с нормативкой (группа мер АУД)

| Мера АУД | Как реализована |
|----------|-----------------|
| АУД.4 Аудит безопасности | Мониторинг /etc/passwd, /etc/shadow, /etc/sudoers |
| АУД.7 Мониторинг НСД | Правило access-denied для неудачных попыток |

## 5. Выводы

- journalctl показывает логи sudo и ошибки сервисов, но не фиксирует отказы в доступе на уровне ядра.
- auditd обеспечивает глубокий аудит системных вызовов, отслеживает неудачные попытки и сохраняет auid.
- В работе созданы правила для контроля критических файлов и фиксации НСД.
- auditd позволяет восстановить хронологию действий нарушителя.
