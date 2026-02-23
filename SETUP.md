# SETUP — Laravel Boilerplate (PHP-FPM + Httpd TCP + External PostgreSQL/MySQL + External Redis Cluster)

Этот boilerplate предназначен для быстрого старта Laravel-проекта в Docker Compose, где:
- **внутри Compose живут только app-сервисы** (PHP-FPM + Httpd + Node/Vite в dev);
- **PHP-FPM и Httpd взаимодействуют по TCP** (порт 9000);
- **PostgreSQL/MySQL — внешний** (managed DB или отдельный сервер/кластер);
- **Redis Cluster — внешний** (managed Redis Cluster или отдельный кластер);
- миграции запускаются **one-off** сервисом `migrate` (prod-like подход).

---

## 0) Предпосылки (что нужно заранее)

### На машине разработчика
- Docker + Docker Compose (plugin)
- Make (обычно есть на macOS)
- Доступ к внешним сервисам:
    - PostgreSQL или MySQL: `host:port`, база, пользователь, пароль
    - Redis Cluster: список нод `host:port,host:port,...`, пароль/ACL (если есть), TLS (если требуется)

---

## 1) Как использовать boilerplate в новом Laravel-проекте

### Вариант A (рекомендуется): создаёте новый Laravel и копируете boilerplate
1) Создайте новый проект Laravel:
```
bash
composer create-project laravel/laravel .
```
2) Скопируйте из этого репозитория в корень вашего Laravel-проекта:
- папку `docker/`
- `docker-compose.yml`
- `docker-compose.dev.yml`
- `docker-compose.prod.yml`
- `Makefile`
- `.env.docker` (как подсказки/контракт переменных)

3) Проверьте, что в корне есть `.env` (Laravel создаёт его сам).

---

## 2) Настройка `.env` (самое важное)

> Важно: в external-модели **нет** контейнеров БД/Redis внутри Compose, поэтому `DB_HOST` и Redis-параметры **должны указывать на внешние endpoints**.

### 2.1 Выбор базы данных (PostgreSQL / MySQL) и переключение Dockerfile

В этом boilerplate есть два Dockerfile для PHP:

- `docker/php.pgsql.Dockerfile` — PHP с расширениями `pdo_pgsql/pgsql`
- `docker/php.mysql.Dockerfile` — PHP с расширениями `pdo_mysql/mysqli`

Чтобы переключиться, в `docker-compose.yml` у сервиса `laravel-php-httpd-tcp` поменяйте `build.dockerfile`:

```yml
services:
  laravel-php-httpd-tcp:
    build:
      context: ./docker
      dockerfile: php.pgsql.Dockerfile # или php.mysql.Dockerfile
      target: php-base
```

Также синхронизируйте `.env` Laravel:

- для PostgreSQL: `DB_CONNECTION=pgsql`, `DB_PORT=5432`
- для MySQL: `DB_CONNECTION=mysql`, `DB_PORT=3306`

Если БД находится в Docker (в другом compose-проекте), app-контейнер должен быть в той же сети. В шаблоне по умолчанию в `docker-compose.yml` подключена внешняя сеть `postgres-dev-network`. При переключении БД не забудьте переключить и сеть: либо `postgres-dev-network`, либо `mysql-dev-network` (или удалите этот network, если вы ходите в managed DB/наружу Docker).

Переключение делается прямо в `services.laravel-php-httpd-tcp.networks`:

```yml
services:
  laravel-php-httpd-tcp:
    networks:
      - laravel-httpd-tcp-network
      - postgres-dev-network
      # - mysql-dev-network
```

### 2.2 PostgreSQL (external)

В external-модели (внешняя БД) Laravel **не создаёт базы данных** автоматически. Команда `php artisan migrate` создаёт таблицы, но не делает `CREATE DATABASE`.

Базу для проекта нужно создать заранее вручную (через SQL или админку). У каждой базы своя схема `public`.

#### 1) Шаблон: “1 проект = 1 база + 1 пользователь”

Пример для проекта `app1`:
- база: `app1_db`
- пользователь: `app1_user`
- пароль: `<PASSWORD_PLACEHOLDER>`

**Создание через SQL (рекомендуется):**

1. **Подключитесь суперпользователем** (обычно `postgres`) через psql/pgAdmin/DataGrip.
2. **Создайте пользователя**:
   ```sql
   CREATE ROLE app1_user WITH LOGIN PASSWORD '<PASSWORD_PLACEHOLDER>';
   ```
3. **Создайте базу** и назначьте владельца:
   ```sql
   CREATE DATABASE app1_db OWNER app1_user;
   ```
4. **Ограничьте доступ** (рекомендуется):
   ```sql
   REVOKE ALL ON DATABASE app1_db FROM PUBLIC;
   GRANT CONNECT ON DATABASE app1_db TO app1_user;
   ```

#### 2) Настройка `.env`

В вашем `.env` установите:
```dotenv
DB_CONNECTION=pgsql
DB_HOST=<postgres_service_or_container_name>
DB_PORT=5432
DB_DATABASE=app1_db
DB_USERNAME=app1_user
DB_PASSWORD=<PASSWORD_PLACEHOLDER>
```

После изменения `.env` обязательно сбросьте кэш конфигурации:
```bash
php artisan config:clear
php artisan cache:clear
```

Дальше запускайте миграции:
```bash
php artisan migrate
```

#### 🧰 Если в PhpStorm/DataGrip “миграции прошли, а таблиц не видно”
Это часто не баг, а настройка отображения.
1. Database tool window → ваш datasource → база `app1_db`.
2. Откройте **Schemas…** (или Properties → Schemas).
3. Отметьте схему **`public`**.
4. Нажмите **Apply / OK** и сделайте **Refresh**.

#### TLS для Postgres (если нужно)
В зависимости от провайдера/драйвера может использоваться `DB_SSLMODE`.
Если ваша инфраструктура требует TLS, добавьте (пример):
```
dotenv
DB_SSLMODE=require
```

### 2.3 MySQL (external)

Аналогично PostgreSQL: в external-модели Laravel **не создаёт** базу автоматически — её нужно создать заранее.

#### 1) Шаблон: “1 проект = 1 база + 1 пользователь”

Пример для проекта `app1`:
- база: `app1_db`
- пользователь: `app1_user`
- пароль: `<PASSWORD_PLACEHOLDER>`

**Создание через SQL (пример):**

```sql
CREATE DATABASE app1_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'app1_user'@'%' IDENTIFIED BY '<PASSWORD_PLACEHOLDER>';
GRANT ALL PRIVILEGES ON app1_db.* TO 'app1_user'@'%';
FLUSH PRIVILEGES;
```

#### 2) Настройка `.env`

В вашем `.env` установите:

```dotenv
DB_CONNECTION=mysql
DB_HOST=<mysql_service_or_host>
DB_PORT=3306
DB_DATABASE=app1_db
DB_USERNAME=app1_user
DB_PASSWORD=<PASSWORD_PLACEHOLDER>
```

После изменения `.env` обязательно сбросьте кэш конфигурации:

```bash
php artisan config:clear
php artisan cache:clear
```

Дальше запускайте миграции:

```bash
php artisan migrate
```
---

### 2.4 Redis Cluster (external)
Этот boilerplate предполагает контракт через переменные окружения.

Добавьте в `.env` (если используете Redis):
```
dotenv
REDIS_CLIENT=phpredis
REDIS_CLUSTER=redis
REDIS_CLUSTER_NODES=<host1>:<port1>,<host2>:<port2>,<host3>:<port3>
REDIS_PASSWORD=<redis_password_or_empty>
REDIS_TLS=false
```
Примечания:
- `REDIS_CLUSTER_NODES` — CSV строка нод кластера.
- `REDIS_PASSWORD` — оставьте пустым, если пароль не нужен.
- `REDIS_TLS` — `true`, если ваш managed Redis Cluster требует TLS.

---

### 2.5 Docker/dev-переменные
В dev обычно достаточно порта Httpd и опционально Xdebug.

Вы можете:
- либо **добавить** нужные переменные в `.env`,
- либо использовать `.env.docker` как “шпаргалку”.

Минимальный набор для dev:
```
dotenv
HTTPD_PORT=80

XDEBUG_MODE=off
XDEBUG_START=no
XDEBUG_CLIENT_HOST=host.docker.internal
```
---

## 3) Запуск DEV окружения

### 3.1 Быстрый старт
```
bash
make setup
```
Что делает `make setup` (логика может отличаться, если вы её меняли):
- сборка образов
- запуск контейнеров
- установка зависимостей (composer + npm)
- генерация ключа приложения
- запуск миграций
- фиксация прав на `storage/` и `bootstrap/cache`

### 3.2 Обычные команды на каждый день
Поднять/остановить:
```
bash
make up
make down
```
Логи:
```
bash
make logs
make logs-php
make logs-httpd
make logs-node
```
Войти в контейнер:
```
bash
make shell-php
make shell-httpd
make shell-node
```
Artisan/Composer:
```
bash
make artisan CMD="about"
make artisan CMD="migrate"
make composer CMD="install"
```
Открыть приложение:
- http://localhost (или порт из `HTTPD_PORT`)

---

## 4) Запуск PROD-like (локально или на сервере)

> В production overlay обычно используются **готовые образы из Registry** (`docker-compose.prod.yml`).

Поднять прод-стек:
```
bash
make up-prod
```
Запустить миграции one-off:
```
bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml run --rm migrate
```
---

## 5) Как работает `migrate` в external-модели

Сервис `migrate`:
- использует тот же PHP-образ, что и приложение;
- подключается к **внешней** PostgreSQL через переменные `.env`;
- выполняет:
```
bash
php artisan migrate --force
```
Рекомендации для CI/CD:
- миграции должны выполняться **последовательно** (не параллельно несколькими деплоями)
- порядок обычно такой:
    1) pull новых образов
    2) `up -d` приложения
    3) `run --rm migrate`

---

## 6) Типичные проблемы и быстрые проверки

### 6.1 Миграции не подключаются к БД
Проверьте:
- `DB_HOST/DB_PORT` доступны из контейнера (сетевой доступ/фаервол/VPC).
- База данных и пользователь созданы (см. раздел 2.1).
- Креды верные (`DB_DATABASE/DB_USERNAME/DB_PASSWORD`).
- `DB_CONNECTION=pgsql`.

### 6.2 В PhpStorm/DataGrip не видно таблиц
Если миграции прошли успешно, но в IDE дерево таблиц пустое:
1. В окне Database выберите ваш datasource → база `app1_db`.
2. Откройте **Schemas…** (или Properties → Schemas).
3. Отметьте схему **`public`**.
4. Нажмите **Apply / OK** и сделайте **Refresh**.

### 6.3 Redis Cluster “не виден” или таймауты
Проверьте:
- `REDIS_CLUSTER_NODES` в формате `host:port,host:port,...` без лишних пробелов
- `REDIS_TLS=true`, если ваш кластер требует TLS
- пароль/ACL (если включены)

### 6.4 В dev не открывается сайт
Проверьте:
- `HTTPD_PORT` не занят другим процессом
- контейнеры в статусе `healthy`:
```
bash
make status
```
---

## 7) Что этот boilerplate *не делает намеренно*
- Не поднимает Postgres/Redis/pgAdmin внутри Docker Compose (это “external infra” версия).
- Не хранит секреты в репозитории: пароли/ключи должны приходить из `.env` (dev) или секретов CI/CD (prod).

---

## 8) Мини-чеклист “готово к работе”
- [ ] `docker compose ...` поднимает `php` + `httpd` (+ `node` в dev)
- [ ] `.env` содержит валидные `DB_*` для внешнего Postgres
- [ ] (опционально) `.env` содержит `REDIS_*` для внешнего Redis Cluster
- [ ] `make setup` проходит и `php artisan migrate` выполняется
- [ ] сайт открывается на `http://localhost` (или вашем порту)

---

