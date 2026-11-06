
Русский (Russian Version)


Запуск инсталляции Vaultwarden на целевой хост

Для развертывания проекта выполните следующую команду в терминале:
bash
ansible-playbook -i inventory.yml install_vaultwarden.yml -k -K
Используйте код с осторожностью.
Примечание: В процессе выполнения команды потребуется ввести пароль дважды: первый раз — для SSH-подключения к серверу, второй раз — для повышения привилегий (SUDO). Подключаемый пользователь обязательно должен иметь SUDO-права на целевом хосте.

Краткое описание проекта

Этот Ansible-плейбук разворачивает Vaultwarden (серверное приложение, альтернативу Bitwarden и KeePass) в максимально безопасном и изолированном корпоративном исполнении.
Вместо использования стандартных Docker-контейнеров, данный проект реализует bare-metal установку:
1. Компилирует бинарный файл Vaultwarden из официальных исходных кодов непосредственно на целевом сервере.
2. Настраивает локальную СУБД PostgreSQL с ограничением доступа.
3. Изолирует процесс через службу Systemd (применяя жесткие ограничения прав и изоляцию файловой системы).
4. Публикует сервис во внешнюю сеть через веб-сервер Nginx, выступающий в роли Reverse Proxy с настроенным TLS-шифрованием.

Важно перед запуском

• Настройка хостов: Обязательно измените адрес целевого хоста и имя пользователя для подключения в файле inventory.yml.
• Параметры безопасности: Если вы производите продуктивную (боевую) инсталляцию — обязательно проверьте и измените дефолтные пароли, DNS-имя, разрешения на самостоятельную регистрацию пользователей и другие критически важные параметры в файле переменных ./group_vars/all.yml.
• Совместимость ОС: Инсталляция была успешно протестирована на операционных системах Debian 12 и Debian 13. На других дистрибутивах Linux работоспособность не гарантируется.

Как начать пользоваться приложением после деплоя

1. Доступ к панели администратора: Откройте веб-интерфейс администрирования по адресу https://vaultwarden.local. Токен для авторизации находится в файле конфигурации all.yml.
2. Настройка почты: В первую очередь перейдите в раздел SMTP Email Settings и настройте интеграцию с вашим почтовым сервером. Это необходимо для корректной отправки приглашений и верификации регистрации новых пользователей.
3. Создание организации: После регистрации первого пользователя и входа в систему создайте Организацию. Организация выполняет роль верхнеуровневого контейнера (пространства имен), в котором будут централизованно храниться все корпоративные записи.
4. Управление доступом: Будучи владельцем (Owner) созданной организации, вы можете отправлять приглашения другим зарегистрированным в системе пользователям для безопасного совместного использования общих паролей и секретов.

Краткое описание структуры файлов плейбука


Системные файлы

• ansible.cfg — Основная конфигурация Ansible. Настраивает путь к инвентарю по умолчанию, отключает валидацию SSH-ключей хостов (host_key_checking = False) и автоматически активирует повышение прав (become = True) через sudo для выполнения системных задач.
• inventory.yml — Файл инвентаря инфраструктуры. Содержит IP-адрес целевого хоста (192.168.1.10), имя пользователя для подключения и принудительно указывает использовать интерпретатор Python 3 на удаленной машине.
• install_vaultwarden.yml — Главный управляющий плейбук (Master Playbook). Не содержит прямых задач, а последовательно импортирует (import_playbook) 12 шагов сценария для сборки, установки и настройки системы.
• ./group_vars/all.yml — Централизованный файл конфигурационных переменных. Хранит фиксированные версии ПО (Vaultwarden 1.37.4, Web Vault v2026.4.1), пути к рабочим директориям (/opt/vaultwarden), реквизиты доступа к PostgreSQL, параметры токена администратора, настройки сетевых портов, доменное имя и флаги безопасности.

Файлы конфигурации и шаблоны (Templates)

• vaultwarden.env (Шаблон .j2) — Конфигурация параметров окружения. Передает бэкенд-приложению настройки из Ansible: строку подключения к СУБД, лимиты пула соединений, запрет на свободную регистрацию (SIGNUPS_ALLOWED=false), флаг работы инвайтов, путь к фронтенду и токен админки.
• vaultwarden.service (Шаблон Systemd .j2) — Конфигурационный юнит системной службы. Обеспечивает автозапуск Vaultwarden после инициализации сети и PostgreSQL. Включает строгие механизмы безопасности ядра Linux (NoNewPrivileges=true, ProtectSystem=strict, ProtectHome=true), разрешая запись данных исключительно в выделенный каталог хранения (ReadWritePaths).
• nginx.conf (Шаблон конфигурации хоста .j2) — Настройки веб-сервера. Конфигурирует Nginx на безусловный редирект с HTTP (80) на безопасный HTTPS (443), задает пути к TLS-сертификатам и ограничивает протоколы до современных TLSv1.2 и TLSv1.3. Организует проксирование запросов на бэкенд Vaultwarden (порт 8000) и выделенный WebSocket-канал (порт 3012) для мгновенной синхронизации клиентов.
Пошаговые сценарии (Нажмите, чтобы раскрыть полный список Playbooks)
• 00-check-system.yml — Валидация системных требований (Pre-flight checks). Проверяет соответствие ОС требованиям дистрибутива Debian (версии 12+), архитектуру процессора (x86_64 или aarch64), наличие как минимум 2 ГБ оперативной памяти и не менее 10 ГБ свободного пространства на корневом разделе /.
• 01-prepare-os.yml — Базовая подготовка операционной системы. Обновляет кэш пакетов apt, выполняет безопасное обновление установленных компонентов и инсталлирует пакеты утилит и библиотек (включая build-essential, dev-пакеты SSL, компрессии, а также библиотеки для PostgreSQL, SQLite и MariaDB), необходимые для сборки приложения.
• 02-create-user-and-directories.yml — Настройка прав доступа и файловой структуры. Создает выделенного системного пользователя и группу vaultwarden без возможности интерактивного входа в систему (/usr/sbin/nologin). Формирует структуру каталогов в /opt/vaultwarden для хранения бинарных файлов, данных, исходников и окружения компилятора Rust, выставляя строгие права доступа 0750.
• 03-install-dependencies.yml — Развертывание компилятора. Проверяет наличие установленного Rust в системе, при необходимости скачивает официальный скрипт rustup.sh и разворачивает стабильную (stable) версию инструментария разработки от имени системного пользователя vaultwarden.
• 04-install-postgresql.yml — Развертывание и изоляция СУБД. Устанавливает PostgreSQL, создает выделенную база данных и пользователя с типом шифрования паролей scram-sha-256. Модифицирует файлы postgresql.conf и pg_hba.conf, изолируя СУБД на локальном интерфейсе 127.0.0.1 для предотвращения доступа из сети.
• 05-download-sources.yml — Загрузка исходного кода. Очищает временные папки и выполняет клонирование официального репозитория Vaultwarden с помощью git clone, фиксируя исходный код на релизной версии 1.37.4.
• 06-build-vaultwarden.yml — Компиляция приложения. Запускает процесс сборки оптимизированного релиза через утилиту cargo build --release с флагом интеграции PostgreSQL. Проверяет успешность генерации бинарного файла и копирует готовый исполняемый файл в рабочую директорию /opt/vaultwarden/bin/ с правами 0755.

English (English Version)


Running Vaultwarden Installation on the Target Host

To deploy the project, execute the following command in your terminal:
bash
ansible-playbook -i inventory.yml install_vaultwarden.yml -k -K
Используйте код с осторожностью.
Note: During execution, you will be prompted to enter passwords twice: first for the SSH connection to the server, and second for privilege escalation (SUDO). The connecting user must have active SUDO privileges on the target host.

Brief Project Description

This Ansible playbook deploys Vaultwarden (a lightweight server application compatible with Bitwarden and KeePass APIs) in a highly secure and isolated corporate environment.
Instead of utilizing standard Docker containers, this project implements a bare-metal installation:
1. Compiles the Vaultwarden binary directly from official source codes on the target server.
2. Configures a local, network-isolated PostgreSQL database instance.
3. Isolates the process using a dedicated Systemd service unit (applying strict sandboxing and filesystem restrictions).
4. Exposes the service to the network via an Nginx web server acting as a Reverse Proxy with secure TLS encryption.

Important Before Deployment

• Host Configuration: Ensure you update the target host IP address and connection username inside the inventory.yml file.
• Security Settings: If performing a production deployment, carefully review and modify default passwords, DNS names, public signup permissions, and other crucial variables in the ./group_vars/all.yml file.
• OS Compatibility: This deployment framework has been successfully tested on Debian 12 and Debian 13. Operational stability on other Linux distributions is not guaranteed.

Getting Started After Deployment

1. Accessing Admin Panel: Open your browser and navigate to the admin interface at https://vaultwarden.local. The authorization token can be found inside your configuration variables (all.yml).
2. Email Configuration: Your first step should be navigating to the SMTP Email Settings section to configure your mail server integration. This is required for sending out invitations and verifying new user registrations.
3. Creating an Organization: Once the initial user is registered and logged in, create an Organization. The Organization acts as a top-level secure container for all enterprise credentials and records.
4. Access Management: As the organization Owner, you can now invite other registered system users into the organization to securely share credentials, passwords, and secrets.

Overview of the Playbook Directory Structure


System Files

• ansible.cfg — Primary Ansible configuration. Sets the default inventory path, disables SSH host key verification (host_key_checking = False), and automatically enables privilege escalation (become = True) via sudo for system management tasks.
• inventory.yml — Infrastructure inventory file. Defines the target host (192.168.1.10), the SSH connection user, and enforces the usage of Python 3 on the remote node.
• install_vaultwarden.yml — Master execution playbook. Contains no direct tasks; instead, it sequentially imports (import_playbook) 12 structured stages to build, install, and configure the overall system.
• ./group_vars/all.yml — Centralized variables repository. Stores static software deployment versions (Vaultwarden 1.37.4, Web Vault v2026.4.1), system directory paths (/opt/vaultwarden), PostgreSQL credentials, the administrator token, network ports, domain definitions, and security flags.

Configuration Files and Templates

• vaultwarden.env (Template .j2) — Environment parameters configuration. Passes Ansible configurations to the compiled backend application, including the database connection string, connection pool limits, public signup restrictions (SIGNUPS_ALLOWED=false), invitation toggles, web vault paths, and the admin token.
• vaultwarden.service (Template Systemd .j2) — System service unit. Manages Vaultwarden auto-start sequences after network and PostgreSQL service initialization. Enforces kernel-level process isolation features (NoNewPrivileges=true, ProtectSystem=strict, ProtectHome=true), strictly limiting write access to the dedicated data directory (ReadWritePaths).
• nginx.conf (Template Virtual Host .j2) — Web server reverse proxy configuration. Sets up Nginx to unconditionally redirect HTTP (80) requests to secure HTTPS (443), defines SSL certificate paths, and restricts protocols to modern TLSv1.2 and TLSv1.3. Proxies standard requests to the Vaultwarden backend (port 8000) and routes dedicated WebSocket connections (port 3012) for instant client synchronization.
Sequential Playbooks (Click to expand full step-by-step list)
• 00-check-system.yml — Environment validation (Pre-flight checks). Verifies that the host OS matches Debian (versions 12+), CPU architecture is supported (x86_64 or aarch64), and ensures at least 2 GB of RAM and 10 GB of free space on the root / partition are available.
• 01-prepare-os.yml — Base operating system preparation. Updates apt package caches, performs safe package upgrades, and installs necessary development utilities and system libraries (including build-essential, SSL dev packages, compression libraries, and dev packages for PostgreSQL, SQLite, and MariaDB) required for source compilation.
• 02-create-user-and-directories.yml — Access control and directory structuring. Creates a dedicated system user and group named vaultwarden with shell access disabled (/usr/sbin/nologin). Sets up the directory hierarchy within /opt/vaultwarden for binaries, data, sources, and the Rust/Cargo environment, applying restricted 0750 permissions.
• 03-install-dependencies.yml — Compiler deployment. Checks if Rust is present on the system, downloads the official rustup.sh installer when necessary, and initializes the stable toolchain framework on behalf of the vaultwarden system user.
• 04-install-postgresql.yml — Database deployment and isolation. Installs PostgreSQL, provisions a dedicated database instance, and configures a system user utilizing secure scram-sha-256 password hashing. Modifies postgresql.conf and pg_hba.conf to bind the database strictly to the local loopback interface 127.0.0.1 to prevent network exposure.
• 05-download-sources.yml — Source code ingestion. Clears historical source pathways and executes a shallow git clone from the official Vaultwarden repository, locking the checked-out codebase to the specific release version 1.37.4.
• 06-build-vaultwarden.yml — Application compilation. Launches the optimized release build process utilizing cargo build --release with explicit PostgreSQL support flags enabled. Confirms the successful compilation of the execution binary and copies it into the operational path /opt/vaultwarden/bin/ with 0755 executable permissions.
