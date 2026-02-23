# Инструкция по установке Laravel Boilerplate

Этот boilerplate предназначен для быстрого развертывания Laravel-проекта с архитектурой **PHP-FPM + Httpd (TCP) + PostgreSQL + Redis + pgAdmin**.

## Для каких приложений подходит эта архитектура

### ✅ 1. Blade-based приложения
**Самый каноничный Laravel-кейс**
* Blade templates
* Server-side rendering
* Немного JS (Alpine, jQuery, vanilla)
* Tailwind / Bootstrap

**Примеры:** CRM / админки, корпоративные сайты, SaaS-панели, Internal tools.

---

### ✅ 2. Laravel + Livewire / Inertia
Frontend есть, но **не как отдельное приложение**
* Livewire
* Inertia + Vue/React (без SPA-архитектуры)
* JS живёт в `resources/`

**Почему всё в одном репо и контейнере:** Нет отдельного frontend-сервиса, Laravel — главный runtime, Vite используется **только для сборки**.

---

### ✅ 3. API-only backend
Даже если **нет UI вообще**:
* Laravel как REST / GraphQL API
* Клиенты: mobile / external frontend
* Swagger / OpenAPI

**Почему всё равно этот вариант:** Один сервис, простая деплой-модель, нет frontend runtime.

---

### Итог
Подходит для: **Blade, Livewire, Inertia, API-only, Admin panels, Small–medium SaaS**. 
👉 Это дефолтный Laravel-мир.

---

## 🚀 Процесс установки

### 1. Создание проекта Laravel
Создайте новый проект Laravel с помощью composer:
```bash
composer create-project laravel/laravel .
```
*(Или укажите имя папки вместо `.`)*

### 2. Копирование файлов boilerplate
Скопируйте следующие файлы и папки из данного boilerplate в корень вашего нового проекта Laravel:
* Папку `docker/` (включая все подпапки и файлы)
* Файлы `docker-compose.yml`, `docker-compose.dev.yml`, `docker-compose.prod.yml`
* Файл `Makefile`

### 3. Настройка окружения (.env)
Откройте созданный файл `.env` в корне Laravel и добавьте в его конец следующую секцию из файла `.env.docker`:

```dotenv
# --- Database Connection ---
# Важно! В основной секции .env замените DB_HOST=127.0.0.1
# на имя сервиса БД из вашего docker-compose.yml (по умолчанию: laravel-postgres-httpd-tcp)
# А REDIS_HOST=127.0.0.1 на laravel-redis-httpd-tcp

# --- pgAdmin Web Interface ---
# Доступ к pgAdmin: http://localhost:8080
PGADMIN_DEFAULT_EMAIL=admin@example.com
PGADMIN_DEFAULT_PASSWORD=admin

# --- Xdebug Configuration ---
# По умолчанию отключен для производительности
# Для включения установите: XDEBUG_MODE=debug и XDEBUG_START=yes
XDEBUG_MODE=off
XDEBUG_START=no
XDEBUG_CLIENT_HOST=host.docker.internal
```

**Важно:** 
1. В секции БД (в начале `.env`) обязательно замените `DB_HOST=127.0.0.1` на имя сервиса из `docker-compose.yml` (например, `laravel-postgres-httpd-tcp`).
2. Вы можете изменить имена сервисов, имя базы данных, а также все логины и пароли на свои собственные в файлах `docker-compose.yml` и `.env`.

### 4. Инициализация проекта
Запустите команду, которая соберет контейнеры, установит все зависимости и выполнит миграции:
```bash
make setup
```

После завершения проект будет доступен по адресу: **http://localhost**
Интерфейс pgAdmin доступен по адресу: **http://localhost:8080**


