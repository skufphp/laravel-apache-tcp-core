# Laravel Boilerplate (PHP-FPM + Httpd TCP + External PostgreSQL/MySQL + External Redis Cluster)

Этот репозиторий представляет собой **boilerplate** для быстрого развертывания Laravel-проектов с оптимизированной архитектурой взаимодействия сервисов и **внешней инфраструктурой** (БД и Redis не поднимаются в Docker Compose).

## Особенности

*   **Производительность:** связь между Httpd и PHP-FPM настроена через **TCP (порт 9000)**.
*   **External infra friendly:** **PostgreSQL/MySQL и Redis Cluster — внешние**, подключение только через переменные окружения.
*   **Автоматизация:** набор команд в `Makefile` для управления контейнерами и Laravel.
*   **Xdebug:** готов к включению одной переменной в `.env`.

## Структура проекта

*   `docker/` — Docker-файлы и конфигурации для PHP и Httpd.
*   `docker-compose.yml` — базовый app-скелет (php+httpd).
*   `docker-compose.dev.yml` — dev-надстройки (ports, bind-mount, node/vite).
*   `docker-compose.prod.yml` — prod-надстройки (image + migrate one-off).
*   `SETUP.md` — инструкция по интеграции boilerplate в ваш Laravel проект.

## Быстрый старт

1.  Создайте проект Laravel: `composer create-project laravel/laravel .`
2.  Скопируйте файлы boilerplate в корень проекта.
3.  Настройте `.env` (подключение к **внешнему PostgreSQL/MySQL** и **внешнему Redis Cluster**).
4.  Запустите:
    ```bash
    make setup
    ```

После завершения:
*   Сайт: [http://localhost](http://localhost)

---

## 📂 База данных: PostgreSQL / MySQL

В этом boilerplate есть два Dockerfile для PHP:

- `docker/php.pgsql.Dockerfile` — PHP с расширениями `pdo_pgsql/pgsql`
- `docker/php.mysql.Dockerfile` — PHP с расширениями `pdo_mysql/mysqli`

### 🔁 Как переключаться между PostgreSQL и MySQL

1) В `docker-compose.yml` у сервиса `laravel-php-httpd-tcp` переключите `build.dockerfile`:

```yml
services:
  laravel-php-httpd-tcp:
    build:
      context: ./docker
      dockerfile: php.pgsql.Dockerfile # или php.mysql.Dockerfile
      target: php-base
```

2) В `.env` Laravel выставьте корректный драйвер и порт:

```dotenv
# PostgreSQL
DB_CONNECTION=pgsql
DB_PORT=5432

# MySQL
# DB_CONNECTION=mysql
# DB_PORT=3306
```

3) Если вы подключаетесь к БД-контейнеру в Docker (а не к managed DB), убедитесь что app-сервис находится в той же сети.
   В текущем шаблоне в `docker-compose.yml` по умолчанию подключена внешняя сеть `postgres-dev-network`.
   При переключении БД не забудьте переключить и сеть у сервиса `laravel-php-httpd-tcp`: либо `postgres-dev-network`, либо `mysql-dev-network` (или удалите этот network, если БД вне Docker).

   Пример (переключение делается прямо в `services.laravel-php-httpd-tcp.networks`):
   ```yml
   services:
     laravel-php-httpd-tcp:
       networks:
         - laravel-httpd-tcp-network
         - postgres-dev-network
         # - mysql-dev-network
   ```

### PostgreSQL: создание базы и подключение

В external-модели (внешняя БД) Laravel **не создаёт базы данных** автоматически. Команда `php artisan migrate` создаёт таблицы, но не делает `CREATE DATABASE`.

Базу для проекта нужно создать заранее вручную (через SQL или админку).

### 🧱 Шаблон: “1 проект = 1 база + 1 пользователь”

Пример для проекта `app1`:
- база: `app1_db`
- пользователь: `app1_user`
- пароль: `<PASSWORD_PLACEHOLDER>`

#### 1) Создание через SQL (рекомендуется)

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

### 🔌 Настройка Laravel (.env)

В `.env` проекта выставьте:
```dotenv
DB_CONNECTION=pgsql
DB_HOST=<postgres_service_or_container_name>
DB_PORT=5432
DB_DATABASE=app1_db
DB_USERNAME=app1_user
DB_PASSWORD=<PASSWORD_PLACEHOLDER>
```

После изменения `.env` сбросьте кэш и запустите миграции:
```bash
php artisan config:clear
php artisan migrate
```

### 🧰 Если в PhpStorm/DataGrip не видно таблиц

Если миграции прошли, но в дереве базы пусто:
1. Database tool window → ваш datasource → база `app1_db`.
2. Откройте **Schemas…** (или Properties → Schemas).
3. Отметьте схему **`public`**.
4. Нажмите **Apply / OK** и сделайте **Refresh**.

---
Подробная инструкция по установке находится в файле [SETUP.md](SETUP.md).