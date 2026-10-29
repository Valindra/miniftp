# MiniFTP — многопоточное клиент-серверное файловое хранилище

Кастомный бинарный протокол **MFTP/1** поверх TCP, сервер на Python (Docker, ASUSTOR ARM64),
публикация в интернет через CloudPub, клиент на C# WPF в стиле WinSCP,
двухфакторный вход: пароль + аппаратный USB-ключ.

```
miniftp/
├── docker-compose.yml          сервер + агент CloudPub
├── .env.example                токен CloudPub, UID/GID, первый пользователь
├── server/
│   ├── Dockerfile              python:3.12-alpine (multi-arch, есть arm64)
│   ├── app/
│   │   ├── protocol.py         формат кадров MFTP/1
│   │   ├── accounts.py         пользователи, PBKDF2, USB-ключи, HMAC
│   │   ├── server.py           ThreadingTCPServer: поток на соединение
│   │   └── manage_users.py     CLI: add / passwd / rekey / list / remove
│   └── tests/test_e2e.py       сквозной тест протокола (20 параллельных клиентов)
├── client/
│   ├── MiniFtpClient.sln
│   └── MiniFtpClient/
│       ├── Protocol/Mftp.cs    кадры, коды, сериализация
│       ├── Security/UsbKey.cs  поиск ключа на флешке, проверка, HMAC, запись
│       ├── Services/MftpClient.cs  сетевой клиент (async, прогресс, SHA-256)
│       ├── Views/LoginWindow   вход + индикатор USB-ключа
│       └── Views/MainWindow    двухпанельный менеджер файлов
└── docs/PROTOCOL.md            описание протокола для отчёта
```

---

## Как работает USB-ключ

1. Сервер при создании пользователя генерирует `key_id` и 256-битный секрет и сохраняет их
   в `users.json`; одновременно выпускается файл `<логин>.auth_token.key`.
2. Этот файл кладётся в **корень флешки** под именем `.auth_token.key`
   (кнопка «Записать ключ на флешку…» в окне входа делает это сама и скрывает файл).
3. Клиент раз в секунду опрашивает **только съёмные диски** (`DriveType.Removable`), находит
   файл, проверяет сигнатуру и контрольную сумму.
4. При входе сервер присылает случайный `nonce`, клиент отвечает
   `HMAC-SHA256(secret, nonce‖user)`. Секрет по сети не передаётся, перехваченный ответ
   повторно не используется.
5. Нет флешки → кнопка «Войти» выдаёт «Доступ запрещён», даже при верном пароле.
   Флешку вынули во время работы → сеанс немедленно завершается.

Проверка двойная: клиент не пустит без ключа, а сервер не пустит модифицированный клиент
без знания секрета.

---

## Блок 2. Развёртывание сервера на ASUSTOR AS1104T

### 1. Подготовка NAS

1. App Central → установите **Docker Engine** (и при желании **Portainer CE**).
2. Настройки → Службы → Терминал → включите **SSH**.
3. Подключитесь: `ssh admin@<IP_NAS>` и проверьте архитектуру:
   ```sh
   uname -m        # ожидается aarch64
   docker version
   ```
   Если вывод `aarch64` — подходят `python:3.12-alpine` (arm64/v8) и `cloudpub/cloudpub:latest-arm64`.

### 2. Копирование проекта

```sh
mkdir -p /volume1/Docker/miniftp && cd /volume1/Docker/miniftp
# скопируйте сюда docker-compose.yml, .env.example и папку server/ (через File Explorer ADM, SMB или scp)
cp .env.example .env
vi .env             # CLOUDPUB_TOKEN, PUID/PGID
id admin            # узнать uid/gid для PUID/PGID
mkdir -p data && chown -R <PUID>:<PGID> data
```

### 3. Запуск

```sh
docker compose up -d --build        # на старых ADM: docker-compose up -d --build
docker compose ps                   # python-ftp-server должен стать (healthy)
docker compose logs -f python-ftp-server
```

При первом запуске создаётся пользователь из `.env` (`Test` / `Test123331=`), а ключ
записывается в `./data/config/keys/Test.auth_token.key`.

### 4. Адрес туннеля CloudPub

```sh
docker compose logs cloudpub
```

В логе агент выводит публичный TCP-адрес вида `host:port` — его и вводите в клиенте.
В локальной сети можно подключаться напрямую: `<IP_NAS>:2121`.

> Формат `TCP=python-ftp-server:2121` взят из документации CloudPub, приведённой в задании.
> Если агент сообщает об ошибке, сверьтесь с актуальной документацией CloudPub — имя
> переменной и формат вывода адреса зависят от версии агента.

### 5. Управление пользователями

```sh
docker compose exec python-ftp-server python manage_users.py list
docker compose exec python-ftp-server python manage_users.py add Ivan          # спросит пароль
docker compose exec python-ftp-server python manage_users.py rekey Test        # перевыпуск ключа, старый перестаёт работать
docker compose exec python-ftp-server python manage_users.py passwd Test --keep-key
docker compose exec python-ftp-server python manage_users.py remove Ivan
```

Рестарт не нужен: сервер перечитывает `users.json` при входе.
**После переноса ключа на флешку удалите файл из `data/config/keys/`.**

### Переменные окружения сервера

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `MFTP_PORT` | 2121 | TCP-порт |
| `MFTP_MAX_CLIENTS` | 32 | максимум одновременных сеансов (потоков) |
| `MFTP_IDLE_TIMEOUT` | 300 | закрытие неактивного соединения, с |
| `MFTP_STORAGE` | /data/storage | корень хранилища (у каждого пользователя своя папка) |
| `MFTP_USERS` | /data/config/users.json | учётные записи |
| `MFTP_BOOTSTRAP_USER/PASSWORD` | — | создать пользователя при первом запуске |

### Тест сервера без Docker

```sh
cd server && python tests/test_e2e.py     # ALL TESTS PASSED
```

---

## Блок 1. Сборка и использование клиента

Требования: Windows 10/11, .NET 8 SDK, Visual Studio 2022 (рабочая нагрузка «Разработка классических приложений .NET»).

```bat
cd client
dotnet build -c Release
dotnet run --project MiniFtpClient
:: или единый exe без установки .NET:
dotnet publish MiniFtpClient -c Release -r win-x64 --self-contained -p:PublishSingleFile=true
```

Сторонних NuGet-пакетов нет — только стандартная библиотека .NET.

**Первый вход**

1. Скопируйте с NAS `data/config/keys/Test.auth_token.key` на ПК.
2. Вставьте флешку → «Записать ключ на флешку…» → выберите файл → подтвердите диск.
3. Индикатор станет зелёным «USB-ключ обнаружен».
4. Адрес/порт (CloudPub или `<IP_NAS>:2121`), логин `Test`, пароль `Test123331=` → «Войти».
5. Удалите исходный `.key` с ПК и с NAS.

**Главное окно** — слева локальный компьютер, справа сервер; перетаскивание между панелями
и из Проводника; горячие клавиши как в WinSCP: F5 — копировать из активной панели,
F7 — новая папка, Del — удалить на сервере, Ctrl+R — обновить, Backspace — вверх.
Целостность каждого файла проверяется по SHA-256, при обрыве клиент переподключается сам
(с новым HMAC-доказательством от ключа).

---

## Ограничения и что улучшить

- **Трафик не шифруется.** Пароль в кадре AUTH идёт открытым текстом (секрет ключа — нет).
  Через публичный туннель это существенно. Решение — обернуть сокет в TLS
  (`ssl.SSLContext.wrap_socket` на сервере, `SslStream` в клиенте); формат кадров не меняется.
- Ключ — это файл: его копию на другую флешку клиент примет. Привязка к серийному номеру
  накопителя (WMI `Win32_DiskDrive.SerialNumber`) или настоящий токен (YubiKey, Рутокен) это закрывают.
- Нет докачки прерванных передач (можно добавить поле `offset` в UPLOAD/DOWNLOAD).
