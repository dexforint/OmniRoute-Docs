---
title: "OmniRoute — руководство по удалению"
version: 3.8.40
lastUpdated: 2026-06-28
---

# OmniRoute — руководство по удалению

🌐 **Языки:** 🇺🇸 [English](./UNINSTALL.md) | 🇧🇷 [Português (Brasil)](../i18n/pt-BR/docs/guides/UNINSTALL.md) | 🇪🇸 [Español](../i18n/es/docs/guides/UNINSTALL.md) | 🇫🇷 [Français](../i18n/fr/docs/guides/UNINSTALL.md) | 🇮🇹 [Italiano](../i18n/it/docs/guides/UNINSTALL.md) | 🇷🇺 [Русский](../i18n/ru/docs/guides/UNINSTALL.md) | 🇨🇳 [中文 (简体)](../i18n/zh-CN/docs/guides/UNINSTALL.md) | 🇩🇪 [Deutsch](../i18n/de/docs/guides/UNINSTALL.md) | 🇮🇳 [हिन्दी](../i18n/in/docs/guides/UNINSTALL.md) | 🇹🇭 [ไทย](../i18n/th/docs/guides/UNINSTALL.md) | 🇺🇦 [Українська](../i18n/uk-UA/docs/guides/UNINSTALL.md) | 🇸🇦 [العربية](../i18n/ar/docs/guides/UNINSTALL.md) | 🇯🇵 [日本語](../i18n/ja/docs/guides/UNINSTALL.md) | 🇻🇳 [Tiếng Việt](../i18n/vi/docs/guides/UNINSTALL.md) | 🇧🇬 [Български](../i18n/bg/docs/guides/UNINSTALL.md) | 🇩🇰 [Dansk](../i18n/da/docs/guides/UNINSTALL.md) | 🇫🇮 [Suomi](../i18n/fi/docs/guides/UNINSTALL.md) | 🇮🇱 [עברית](../i18n/he/docs/guides/UNINSTALL.md) | 🇭🇺 [Magyar](../i18n/hu/docs/guides/UNINSTALL.md) | 🇮🇩 [Bahasa Indonesia](../i18n/id/docs/guides/UNINSTALL.md) | 🇰🇷 [한국어](../i18n/ko/docs/guides/UNINSTALL.md) | 🇲🇾 [Bahasa Melayu](../i18n/ms/docs/guides/UNINSTALL.md) | 🇳🇱 [Nederlands](../i18n/nl/docs/guides/UNINSTALL.md) | 🇳🇴 [Norsk](../i18n/no/docs/guides/UNINSTALL.md) | 🇵🇹 [Português (Portugal)](../i18n/pt/docs/guides/UNINSTALL.md) | 🇷🇴 [Română](../i18n/ro/docs/guides/UNINSTALL.md) | 🇵🇱 [Polski](../i18n/pl/docs/guides/UNINSTALL.md) | 🇸🇰 [Slovenčina](../i18n/sk/docs/guides/UNINSTALL.md) | 🇸🇪 [Svenska](../i18n/sv/docs/guides/UNINSTALL.md) | 🇵🇭 [Filipino](../i18n/phi/docs/guides/UNINSTALL.md) | 🇨🇿 [Čeština](../i18n/cs/docs/guides/UNINSTALL.md)

Это руководство описывает, как полностью удалить OmniRoute из вашей системы.

---

## Быстрое удаление (v3.6.2+)

OmniRoute предоставляет два встроенных скрипта для чистого удаления:

### С сохранением ваших данных

```bash
npm run uninstall
```

Удаляет приложение OmniRoute, но **сохраняет** вашу базу данных, конфигурации, API-ключи и настройки провайдеров в `~/.omniroute/`. Используйте, если планируете переустановить позже и хотите сохранить настройки.

### Полное удаление

```bash
npm run uninstall:full
```

Удаляет приложение **и безвозвратно стирает** все данные:

- Базу данных (`storage.sqlite`)
- Конфигурации провайдеров и API-ключи
- Резервные копии
- Файлы логов
- Все файлы в каталоге `~/.omniroute/`

> ⚠️ **Предупреждение:** `npm run uninstall:full` необратимо. Все ваши подключения провайдеров, комбо, API-ключи и история использования будут удалены навсегда.

---

## Ручное удаление

### Глобальная установка NPM

```bash
# Удалить глобальный пакет
npm uninstall -g omniroute

# (Опционально) Удалить каталог данных
rm -rf ~/.omniroute
```

### Глобальная установка pnpm

```bash
pnpm uninstall -g omniroute
rm -rf ~/.omniroute
```

### Docker

```bash
# Остановить и удалить контейнер
docker stop omniroute
docker rm omniroute

# Удалить том (удаляет все данные)
docker volume rm omniroute-data

# (Опционально) Удалить образ
docker rmi diegosouzapw/omniroute:latest
```

### Docker Compose

```bash
# Остановить и удалить контейнеры
docker compose down

# Также удалить тома (удаляет все данные)
docker compose down -v
```

### Десктопное приложение Electron

**Windows:**

- Откройте `Параметры → Приложения → OmniRoute → Удалить`
- Или запустите деинсталлятор NSIS из каталога установки

**macOS:**

- Перетащите `OmniRoute.app` из `/Applications` в Корзину
- Удалите данные: `rm -rf ~/Library/Application Support/omniroute`

**Linux:**

- Удалите файл AppImage
- Удалите данные: `rm -rf ~/.omniroute`

### Установка из исходников (git clone)

```bash
# Удалить склонированный каталог
rm -rf /path/to/omniroute

# (Опционально) Удалить каталог данных
rm -rf ~/.omniroute
```

---

## Каталоги данных

По умолчанию OmniRoute хранит данные в следующих местах:

| Платформа     | Путь по умолчанию             | Переопределение           |
| ------------- | ----------------------------- | ------------------------- |
| Linux         | `~/.omniroute/`               | переменная `DATA_DIR`     |
| macOS         | `~/.omniroute/`               | переменная `DATA_DIR`     |
| Windows       | `%APPDATA%/omniroute/`        | переменная `DATA_DIR`     |
| Docker        | `/app/data/` (смонтированный том) | переменная `DATA_DIR` |
| XDG-совместимый | `$XDG_CONFIG_HOME/omniroute/` | переменная `XDG_CONFIG_HOME` |

### Файлы в каталоге данных

| Файл/Каталог         | Описание                                             |
| -------------------- | ---------------------------------------------------- |
| `storage.sqlite`     | Основная база данных (провайдеры, комбо, настройки, ключи) |
| `storage.sqlite-wal` | Журнал предзаписи SQLite (временный)                 |
| `storage.sqlite-shm` | Разделяемая память SQLite (временная)                |
| `call_logs/`         | Архивы полезных данных запросов                      |
| `backups/`           | Автоматические резервные копии базы данных           |
| `log.txt`            | Устаревший журнал запросов (опционально)             |

---

## Проверка полного удаления

После удаления убедитесь, что не осталось файлов:

```bash
# Проверить глобальный npm-пакет
npm list -g omniroute 2>/dev/null

# Проверить каталог данных
ls -la ~/.omniroute/ 2>/dev/null

# Проверить запущенные процессы
pgrep -f omniroute
```

Если какой-либо процесс ещё запущен, остановите его:

```bash
pkill -f omniroute
```
