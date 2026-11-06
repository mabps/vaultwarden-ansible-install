# Vaultwarden Deployment via Ansible

## Русский (Russian Version)

### Запуск инсталляции Vaultwarden на целевой хост

Для развертывания проекта выполните следующую команду в терминале:

```bash
ansible-playbook -i inventory.yml install_vaultwarden.yml -k -K
```

> **Примечание:** В процессе выполнения команды потребуется ввести пароль дважды: первый раз — для SSH-подключения к серверу, второй раз — для повышения привилегий (SUDO). Подключаемый пользователь обязательно должен иметь SUDO-права на целевом хосте.

---

### Краткое описание проекта

Этот Ansible-плейбук разворачивает Vaultwarden (серверное приложение, альтернативу Bitwarden и KeePass) в максимально безопасном и изолированном корпоративном исполнении.

Вместо использования стандартных Docker-контейнеров, данный проект реализует bare-metal установку:
1. Компилирует бинарный файл Vaultwarden из официальных исходных кодов непосредственно на целевом сервере.
2. Настраивает локальную СУБД PostgreSQL с ограничением доступа.
3. Изолирует процесс через службу Systemd (применяя жесткие ограничения прав и изоляцию файловой системы).
4. Публикует сервис во внешнюю сеть через веб-сервер Nginx, выступающий в роли Reverse Proxy с настроенным TLS-шифрованием.

---

### Важно перед запуском

* **Настройка хостов:** Обязательно измените адрес целевого хоста и имя пользователя для подключения в файле `inventory.yml`.
* **Параметры безопасности:** Если вы производите продуктивную (боевую) инсталляцию — обязательно проверьте и измените дефолтные пароли, DNS-имя, разрешения на самостоятельную регистрацию пользователей и другие критически важные параметры в файле переменных `./group_vars/all.yml`.
* **Совместимость ОС:** Инсталляция была успешно протестирована на операционных системах Debian 12 и Debian 13. На других дистрибутивах Linux работоспособность не гарантируется.

---

### Как начать пользоваться приложением после деплоя

1. **Доступ к панели администратора:** Откройте веб-интерфейс администрирования по адресу `https://vaultwarden.local/admin`. Токен для авторизации находится в файле конфигурации `all.yml`.
2. **Настройка почты:** В первую очередь перейдите в раздел SMTP Email Settings и настройте интеграцию с вашим почтовым сервером. Это необходимо для корректной отправки приглашений и верификации регистрации новых пользователей.
3. **Создание организации:** После регистрации первого пользователя и входа в систему создайте Организацию. Организация выполняет роль верхнеуровневого контейнера (пространства имен), в котором будут централизованно храниться все корпоративные записи.
4. **Управление доступом:** Будучи владельцем (Owner) созданной организации, вы можете отправлять приглашения другим зарегистрированным в системе пользователям для безопасного совместного использования общих паролей и секретов.

---

### Краткое описание структуры файлов плейбука

#### Системные файлы

* **`ansible.cfg`** — Основная конфигурация Ansible. Настраивает путь к инвентарю по умолчанию, отключает валидацию SSH-ключей хостов (`host_key_checking = False`) и автоматически активирует повышение прав (`become = True`) через `sudo` для выполнения системных задач.
* **`inventory.yml`** — Файл инвентаря инфраструктуры. Содержит IP-адрес целевого хоста (`192.168.1.10`), имя пользователя для подключения и принудительно указывает использовать интерпретатор Python 3 на удаленной машине.
* **`install_vaultwarden.yml`** — Главный управляющий плейбук (Master Playbook). Не содержит прямых задач, а последовательно импортирует (`import_playbook`) 12 шагов сценария для сборки, установки и настройки системы.
* **`./group_vars/all.yml`** — Централизованный файл конфигурационных переменных. Хранит фиксированные версии ПО (Vaultwarden 1.37.4, Web Vault v2026.4.1), пути к рабочим директориям (`/opt/vaultwarden`), реквизиты доступа к PostgreSQL, параметры токена администратора, настройки сетевых портов, доменное имя и флаги безопасности.

#### Файлы конфигурации и шаблоны (Templates)

* **`vaultwarden.env`** *(Шаблон `.j2`)* — Конфигурация параметров окружения. Передает бэкенд-приложению настройки из Ansible: строку подключения к СУБД, лимиты пула соединений, запрет на свободную регистрацию (`SIGNUPS_ALLOWED=false`), флаг работы инвайтов, путь к фронтенду и токен админки.
* **`vaultwarden.service`** *(Шаблон Systemd `.j2`)* — Конфигурационный юнит системной службы. Обеспечивает автозапуск Vaultwarden после инициализации сети и PostgreSQL. Включает строгие механизмы безопасности ядра Linux (`NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`), разрешая запись данных исключительно в выделенный каталог хранения (`ReadWritePaths`).
* **`nginx.conf`** *(Шаблон конфигурации хоста `.j2`)* — Настройки веб-сервера. Конфигурирует Nginx на безусловный редирект с HTTP (80) на безопасный HTTPS (443), задает пути к TLS-сертификатам и ограничивает протоколы до современных TLSv1.2 и TLSv1.3. Организует проксирование запросов на бэкенд Vaultwarden (порт 8000) и выделенный WebSocket-канал (порт 3012) для мгновенной синхронизации клиентов.

#### Пошаговые сценарии (Playbooks)

<details>
<summary>Нажмите, чтобы раскрыть полный список Playbooks</summary>

* **`00-check-system.yml`** — Валидация системных требований (Pre-flight checks). Проверяет соответствие ОС требованиям дистрибутива Debian (версии 12+), архитектуру процессора (x86_64 или aarch64), наличие как минимум 2 ГБ оперативной памяти и не менее 10 ГБ свободного пространства на корневом разделе `/`.
* **`01-prepare-os.yml`** — Базовая подготовка операционной системы. Обновляет кэш пакетов `apt`, выполняет безопасное обновление установленных компонентов и инсталлирует пакеты утилит и библиотек (включая `build-essential`, dev-пакеты SSL, компрессии, а также библиотеки для PostgreSQL, SQLite и MariaDB), необходимые для сборки приложения.
* **`02-create-user-and-directories.yml`** — Настройка прав доступа и файловой структуры. Создает выделенного системного пользователя и группу `vaultwarden` без возможности интерактивного входа в систему (`/usr/sbin/nologin`). Формирует структуру каталогов в `/opt/vaultwarden` для хранения бинарных файлов, данных, исходников и окружения компилятора Rust, выставляя строгие права доступа `0750`.
* **`03-install-dependencies.yml`** — Развертывание компилятора. Проверяет наличие установленного Rust в системе, при необходимости скачивает официальный скрипт `rustup.sh` и разворачивает стабильную (`stable`) версию инструментария разработки от имени системного пользователя `vaultwarden`.
* **`04-install-postgresql.yml`** — Развертывание и изоляция СУБД. Устанавливает PostgreSQL, создает выделенную база данных и пользователя с типом шифрования паролей `scram-sha-256`. Модифицирует файлы `postgresql.conf` и `pg_hba.conf`, изолируя СУБД на локальном интерфейсе `127.0.0.1` для предотвращения доступа из сети.
* **`05-download-sources.yml`** — Загрузка исходного кода. Очищает временные папки и выполняет клонирование официального репозитория Vaultwarden с помощью `git clone`, фиксируя исходный код на релизной версии `1.37.4`.
* **`06-build-vaultwarden.yml`** — Компиляция приложения. Запускает процесс сборки оптимизированного релиза через утилиту `cargo build --release` с флагом интеграции PostgreSQL. Проверяет успешность генерации бинарного файла и копирует готовый исполняемый файл в рабочую директорию `/opt/vaultwarden/bin/` с правами `0755`.
</details>

---
---

## English (English Version)

### Running Vaultwarden Installation on the Target Host

To deploy the project, execute the following command in your terminal:

```bash
ansible-playbook -i inventory.yml install_vaultwarden.yml -k -K
```

> **Note:** During execution, you will be prompted to enter passwords twice: first for the SSH connection to the server, and second for privilege escalation (SUDO). The connecting user must have active SUDO privileges on the target host.

---

### Brief Project Description

This Ansible playbook deploys Vaultwarden (a lightweight server application compatible with Bitwarden and KeePass APIs) in a highly secure and isolated corporate environment.

Instead of utilizing standard Docker containers, this project implements a bare-metal installation:
1. Compiles the Vaultwarden binary directly from official source codes on the target server.
2. Configures a local, network-isolated PostgreSQL database instance.
3. Isolates the process using a dedicated Systemd service unit (applying strict sandboxing and filesystem restrictions).
4. Exposes the service to the network via an Nginx web server acting as a Reverse Proxy with secure TLS encryption.

---

### Important Before Deployment

* **Host Configuration:** Ensure you update the target host IP address and connection username inside the `inventory.yml` file.
* **Security Settings:** If performing a production deployment, carefully review and modify default passwords, DNS names, public signup permissions, and other crucial variables in the `./group_vars/all.yml` file.
* **OS Compatibility:** This deployment framework has been successfully tested on Debian 12 and Debian 13. Operational stability on other Linux distributions is not guaranteed.

---

### Getting Started After Deployment

1. **Accessing Admin Panel:** Open your browser and navigate to the admin interface at `https://vaultwarden.local`. The authorization token can be found inside your configuration variables (`all.yml`).
2. **Email Configuration:** Your first step should be navigating to the SMTP Email Settings section to configure your mail server integration. This is required for sending out invitations and verifying new user registrations.
3. **Creating an Organization:** Once the initial user is registered and logged in, create an Organization. The Organization acts as a top-level secure container for all enterprise credentials and records.
