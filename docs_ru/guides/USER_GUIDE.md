---
title: "Руководство пользователя"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Руководство пользователя

🌐 **Языки:** 🇺🇸 [English](./USER_GUIDE.md) | 🇧🇷 [Português (Brasil)](../i18n/pt-BR/docs/guides/USER_GUIDE.md) | 🇪🇸 [Español](../i18n/es/docs/guides/USER_GUIDE.md) | 🇫🇷 [Français](../i18n/fr/docs/guides/USER_GUIDE.md) | 🇮🇹 [Italiano](../i18n/it/docs/guides/USER_GUIDE.md) | 🇷🇺 [Русский](../i18n/ru/docs/guides/USER_GUIDE.md) | 🇨🇳 [中文 (简体)](../i18n/zh-CN/docs/guides/USER_GUIDE.md) | 🇩🇪 [Deutsch](../i18n/de/docs/guides/USER_GUIDE.md) | 🇮🇳 [हिन्दी](../i18n/in/docs/guides/USER_GUIDE.md) | 🇹🇭 [ไทย](../i18n/th/docs/guides/USER_GUIDE.md) | 🇺🇦 [Українська](../i18n/uk-UA/docs/guides/USER_GUIDE.md) | 🇸🇦 [العربية](../i18n/ar/docs/guides/USER_GUIDE.md) | 🇯🇵 [日本語](../i18n/ja/docs/guides/USER_GUIDE.md) | 🇻🇳 [Tiếng Việt](../i18n/vi/docs/guides/USER_GUIDE.md) | 🇧🇬 [Български](../i18n/bg/docs/guides/USER_GUIDE.md) | 🇩🇰 [Dansk](../i18n/da/docs/guides/USER_GUIDE.md) | 🇫🇮 [Suomi](../i18n/fi/docs/guides/USER_GUIDE.md) | 🇮🇱 [עברית](../i18n/he/docs/guides/USER_GUIDE.md) | 🇭🇺 [Magyar](../i18n/hu/docs/guides/USER_GUIDE.md) | 🇮🇩 [Bahasa Indonesia](../i18n/id/docs/guides/USER_GUIDE.md) | 🇰🇷 [한국어](../i18n/ko/docs/guides/USER_GUIDE.md) | 🇲🇾 [Bahasa Melayu](../i18n/ms/docs/guides/USER_GUIDE.md) | 🇳🇱 [Nederlands](../i18n/nl/docs/guides/USER_GUIDE.md) | 🇳🇴 [Norsk](../i18n/no/docs/guides/USER_GUIDE.md) | 🇵🇹 [Português (Portugal)](../i18n/pt/docs/guides/USER_GUIDE.md) | 🇷🇴 [Română](../i18n/ro/docs/guides/USER_GUIDE.md) | 🇵🇱 [Polski](../i18n/pl/docs/guides/USER_GUIDE.md) | 🇸🇰 [Slovenčina](../i18n/sk/docs/guides/USER_GUIDE.md) | 🇸🇪 [Svenska](../i18n/sv/docs/guides/USER_GUIDE.md) | 🇵🇭 [Filipino](../i18n/phi/docs/guides/USER_GUIDE.md) | 🇨🇿 [Čeština](../i18n/cs/docs/guides/USER_GUIDE.md)

Полное руководство по настройке провайдеров, созданию комбо, интеграции CLI-инструментов и развёртыванию OmniRoute.

---

## Содержание

- [Цены в двух словах](#-pricing-at-a-glance)
- [Сценарии использования](#-use-cases)
- [Настройка провайдеров](#-provider-setup)
- [Интеграция CLI](#-cli-integration)
- [Развёртывание](#-deployment)
- [Доступные модели](#-available-models)
- [Расширенные возможности](#-advanced-features)
- [Автомаршрутизация (zero-config)](#-auto-routing-zero-config)
- [Интеграция MCP и A2A](#-mcp--a2a-integration)
- [Система навыков](#-skills-system)
- [Система памяти](#-memory-system)
- [Вебхуки](#-webhooks)
- [Облачные агенты](#-cloud-agents)
- [Программное управление](#-programmatic-management)
- [Встроенный CLI](#-internal-cli)
- [Десктопное приложение (Electron)](#-desktop-application-electron)

---

## 💰 Цены в двух словах

| Уровень             | Провайдер         | Стоимость   | Сброс квоты    | Лучше всего для      |
| ------------------- | ----------------- | ----------- | -------------- | -------------------- |
| **💳 ПОДПИСКА**     | Claude Code (Pro) | $20/мес.    | 5ч + еженедельно | Уже есть подписка   |
|                     | Codex (Plus/Pro)  | $20-200/мес.| 5ч + еженедельно | Пользователи OpenAI |
|                     | GitHub Copilot    | $10-19/мес. | Ежемесячно     | Пользователи GitHub |
| **🔑 API-КЛЮЧ**     | DeepSeek          | По использованию | Нет       | Дешёвые рассуждения |
|                     | Groq              | По использованию | Нет       | Ультрабыстрый инференс |
|                     | xAI (Grok)        | По использованию | Нет       | Рассуждения Grok 4  |
|                     | Mistral           | По использованию | Нет       | Модели в ЕС         |
|                     | Perplexity        | По использованию | Нет       | С поиском           |
|                     | Together AI       | По использованию | Нет       | Открытые модели     |
|                     | Fireworks AI      | По использованию | Нет       | Быстрые изображения FLUX |
|                     | Cerebras          | По использованию | Нет       | Wafer-scale скорость |
|                     | Cohere            | По использованию | Нет       | Command R+ RAG      |
|                     | NVIDIA NIM        | По использованию | Нет       | Корпоративные модели |
|                     | Baidu Qianfan     | По использованию | Нет       | Модели ERNIE        |
| **💰 ДЕШЁВЫЕ**      | GLM-4.7           | $0.6/1M     | Ежедневно в 10:00 | Бюджетный резерв  |
|                     | MiniMax M2.1      | $0.2/1M     | Скользящее 5-часовое | Самый дешёвый вариант |
|                     | Kimi K2           | $9/мес. фикс | 10M токенов/мес. | Предсказуемая стоимость |
| **🆓 БЕСПЛАТНЫЕ**   | Qoder             | $0          | Безлимитно     | 8 моделей бесплатно |
|                     | Qwen              | $0          | Безлимитно     | 3 модели бесплатно  |
|                     | Kiro              | $0          | ~50 кредитов/мес. | Claude бесплатно |

---

## 🎯 Сценарии использования

### Сценарий 1: «У меня подписка Claude Pro»

**Проблема:** квота истекает неиспользованной, ограничения частоты при интенсивном кодировании

```
Комбо: "maximize-claude"
  1. cc/claude-opus-4-7        (использовать подписку полностью)
  2. glm/glm-4.7               (дешёвый резерв, когда квота кончилась)
  3. if/kimi-k2       (бесплатный аварийный фолбэк)

Стоимость в месяц: $20 (подписка) + ~$5 (резерв) = $25 итог
против $20 + упрётесь в лимиты = разочарование
```

### Сценарий 2: «Хочу нулевые расходы»

**Проблема:** не можете позволить подписки, нужен надёжный ИИ для кодирования

```
Комбо: "free-forever"
  1. if/kimi-k2       (бесплатно безлимитно)
  2. qw/qwen3-coder-plus       (бесплатно безлимитно)

Стоимость в месяц: $0
Качество: модели продакшен-уровня
```

### Сценарий 3: «Нужно кодирование 24/7, без перерывов»

**Проблема:** дедлайны, простой недопустим

```
Комбо: "always-on"
  1. cc/claude-opus-4-7        (лучшее качество)
  2. cx/gpt-5.5                (вторая подписка)
  3. glm/glm-4.7               (дёшево, сброс ежедневно)
  4. minimax/MiniMax-M2.1      (самое дешёвое, сброс 5ч)
  5. if/kimi-k2       (бесплатно безлимитно)

Результат: 5 слоёв фолбэка = нулевой простой
Стоимость в месяц: $20-200 (подписки) + $10-20 (резерв)
```

### Сценарий 4: «Хочу БЕСПЛАТНЫЙ ИИ в OpenClaw»

**Проблема:** нужен ИИ-ассистент в мессенджерах, полностью бесплатно

```
Комбо: "openclaw-free"
  1. if/qwen3-coder-plus       (бесплатно безлимитно)
  2. if/deepseek-r1            (бесплатно безлимитно)
  3. if/kimi-k2                (бесплатно безлимитно)

Стоимость в месяц: $0
Доступ через: WhatsApp, Telegram, Slack, Discord, iMessage, Signal...
```

---

## 📖 Настройка провайдеров

### 🔐 Провайдеры по подписке

#### Claude Code (Pro/Max)

```bash
Панель → Providers → Connect Claude Code
→ OAuth-вход → автоматическое обновление токена
→ отслеживание квоты 5-часовой + еженедельной

Модели:
  cc/claude-opus-4-7
  cc/claude-sonnet-4-6
  cc/claude-haiku-4-5-20251001
```

**Совет:** используйте Opus для сложных задач, Sonnet — для скорости. OmniRoute отслеживает квоту по каждой модели!

Маршруты Claude и Claude Code-совместимых сохраняют усилие thinking `max` для моделей
Opus и Sonnet. Модели Haiku не принимают уровень усилия `max`, поэтому OmniRoute понижает такой
запрос до высокого бюджета thinking перед отправкой в апстрим.

#### OpenAI Codex (Plus/Pro)

```bash
Панель → Providers → Connect Codex
→ OAuth-вход (порт 1455)
→ сброс 5-часовой + еженедельный

Модели:
  cx/gpt-5.5
  cx/gpt-5.4
  cx/gpt-5.3-codex
  cx/gpt-5.3-codex-spark
```

#### GitHub Copilot

```bash
Панель → Providers → Connect GitHub
→ OAuth через GitHub
→ ежемесячный сброс (1-го числа)

Модели:
  gh/gpt-5.5
  gh/gpt-5.4
  gh/claude-sonnet-4.6
  gh/claude-opus-4.7
  gh/gemini-3.1-pro-preview
```

### 💰 Дешёвые провайдеры

#### GLM-4.7 (ежедневный сброс, $0.6/1M)

1. Регистрация: [Zhipu AI](https://open.bigmodel.cn)
2. Получите API-ключ в Coding Plan
3. Панель → Add API Key: Provider: `glm`, API Key: `your-key`

**Используйте:** `glm/glm-4.7` — **Совет:** Coding Plan предлагает 3× квоту за 1/7 стоимости! Сброс ежедневно в 10:00.

#### MiniMax M2.1 (сброс 5ч, $0.20/1M)

1. Регистрация: [MiniMax](https://www.minimax.io)
2. Получите API-ключ → Панель → Add API Key

**Используйте:** `minimax/MiniMax-M2.1` — **Совет:** самый дешёвый вариант для длинного контекста (1M токенов)!

#### Kimi K2 ($9/мес. фикс)

1. Подписка: [Moonshot AI](https://platform.moonshot.ai)
2. Получите API-ключ → Панель → Add API Key

**Используйте:** `kimi/kimi-k2.5` — **Совет:** фиксированные $9/мес. за 10M токенов = эффективная стоимость $0.90/1M!

#### Baidu Qianfan / ERNIE

1. Регистрация: [Baidu AI Cloud Qianfan](https://cloud.baidu.com/product/wenxinworkshop)
2. Создайте API-ключ Qianfan → Панель → Add API Key: Provider: `qianfan`

**Используйте:** `qianfan/ernie-5.1`, `qianfan/ernie-x1.1` или другой OpenAI-совместимый ID модели Qianfan.

### 🆓 БЕСПЛАТНЫЕ провайдеры

У бесплатных провайдеров без авторизации есть переключатель рядом с **No authentication required** на странице провайдера.
Его отключение деактивирует провайдера, убирает его из представлений Providers configured/compact и
убирает его модели из `/v1/models`.

#### Qoder (8 БЕСПЛАТНЫХ моделей)

```bash
Панель → Connect Qoder → OAuth-вход → безлимитное использование

Модели: if/kimi-k2, if/qwen3-coder-plus, if/qwen3-max, if/qwen3-235b, if/deepseek-r1, if/deepseek-v3.2
```

#### Kiro (Claude БЕСПЛАТНО)

```bash
Панель → Connect Kiro → AWS Builder ID или Google/GitHub → ~50 кредитов/мес.

Модели: kr/claude-sonnet-4.5, kr/claude-haiku-4.5
```

---

## 🎨 Комбо

Карточки комбо можно переупорядочивать прямо в **Панель → Combos**, перетаскивая за ручку на каждой карточке. Порядок хранится в SQLite и восстанавливается при перезагрузке.

### Пример 1: максимум подписки → дешёвый резерв

```
Панель → Combos → Create New

Название: premium-coding
Модели:
  1. cc/claude-opus-4-7 (основная по подписке)
  2. glm/glm-4.7 (дешёвый резерв, $0.6/1M)
  3. minimax/MiniMax-M2.7 (самый дешёвый фолбэк, $0.3/1M)

Использовать в CLI: premium-coding
```

### Пример 2: только бесплатные (нулевая стоимость)

```
Название: free-combo
Модели:
  1. if/kimi-k2 (безлимитно)
  2. qw/coder-model (безлимитно)

Стоимость: $0 навсегда!
```

---

## 🔧 Интеграция CLI

### Cursor IDE

```
Settings → Models → Advanced:
  OpenAI API Base URL: http://localhost:20128/v1
  OpenAI API Key: [из панели omniroute]
  Model: cc/claude-opus-4-7
```

### Claude Code

Отредактируйте `~/.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:20128",
    "ANTHROPIC_AUTH_TOKEN": "your-omniroute-api-key"
  }
}
```

Используйте здесь Claude-совместимый корневой эндпоинт. Не добавляйте `/v1` к `ANTHROPIC_BASE_URL`.

### Codex CLI

```bash
export OPENAI_BASE_URL="http://localhost:20128"
export OPENAI_API_KEY="your-omniroute-api-key"
codex "your prompt"
```

### OpenClaw

Отредактируйте `~/.openclaw/openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "omniroute/if/kimi-k2" }
    }
  },
  "models": {
    "providers": {
      "omniroute": {
        "baseUrl": "http://localhost:20128/v1",
        "apiKey": "your-omniroute-api-key",
        "api": "openai-completions",
        "models": [{ "id": "if/kimi-k2", "name": "kimi-k2" }]
      }
    }
  }
}
```

**Или через панель:** CLI Tools → OpenClaw → Auto-config

### Cline / Continue / RooCode

```
Provider: OpenAI Compatible
Base URL: http://localhost:20128/v1
API Key: [из панели]
Model: cc/claude-opus-4-7
```

---

## 🚀 Развёртывание

### Глобальная установка npm (рекомендуется)

```bash
npm install -g omniroute

# Создать каталог конфигурации
mkdir -p ~/.omniroute

# Создать файл .env (см. .env.example)
cp .env.example ~/.omniroute/.env

# Запустить сервер
omniroute
# Или с пользовательским портом:
omniroute --port 3000
```

CLI автоматически загружает `.env` из `~/.omniroute/.env` или `./.env`.

### Удаление

Когда OmniRoute больше не нужен, мы предоставляем два быстрых скрипта для чистого удаления:

| Команда                  | Действие                                                                                |
| ------------------------ | --------------------------------------------------------------------------------------- |
| `npm run uninstall`      | Удаляет системное приложение, но **сохраняет вашу БД и конфигурации** в `~/.omniroute`. |
| `npm run uninstall:full` | Удаляет приложение И безвозвратно **стирает все конфигурации, ключи и базы данных**.    |

> Примечание: чтобы выполнить эти команды, перейдите в папку проекта OmniRoute (если вы его клонировали) и выполните их там. Как вариант, при глобальной установке можно просто выполнить `npm uninstall -g omniroute`.

### Развёртывание на VPS

```bash
git clone https://github.com/diegosouzapw/OmniRoute.git
cd OmniRoute && npm install && npm run build

export JWT_SECRET="your-secure-secret-change-this"
export INITIAL_PASSWORD="your-password"
export DATA_DIR="/var/lib/omniroute"
export PORT="20128"
export HOSTNAME="0.0.0.0"
export NODE_ENV="production"
export NEXT_PUBLIC_BASE_URL="http://localhost:20128"
export API_KEY_SECRET="endpoint-proxy-api-key-secret"

npm run start
# Или: pm2 start npm --name omniroute -- start
```

### Развёртывание через PM2 (мало памяти)

Для серверов с ограниченной RAM используйте опцию лимита памяти:

```bash
# С лимитом 512MB (по умолчанию)
pm2 start npm --name omniroute -- start

# Или с пользовательским лимитом памяти
OMNIROUTE_MEMORY_MB=512 pm2 start npm --name omniroute -- start

# Или используя ecosystem.config.js
pm2 start ecosystem.config.js
```

Создайте `ecosystem.config.js`:

```javascript
module.exports = {
  apps: [
    {
      name: "omniroute",
      script: "npm",
      args: "start",
      env: {
        NODE_ENV: "production",
        OMNIROUTE_MEMORY_MB: "512",
        JWT_SECRET: "your-secret",
        INITIAL_PASSWORD: "your-password",
      },
      node_args: "--max-old-space-size=512",
      max_memory_restart: "300M",
    },
  ],
};
```

### Docker

```bash
# Собрать образ (по умолчанию = runner-cli с предустановленными codex/claude/droid)
docker build -t omniroute:cli .

# Портативный режим (рекомендуется)
docker run -d --name omniroute -p 20128:20128 --env-file ./.env -v omniroute-data:/app/data omniroute:cli
```

Для режима интеграции с хостом с бинарниками CLI см. раздел Docker в основной документации.

### Void Linux (xbps-src)

Пользователи Void Linux могут собрать и установить OmniRoute нативно с помощью кросс-компиляционного фреймворка `xbps-src`. Он автоматизирует standalone-сборку Node.js вместе с необходимыми нативными биндингами `better-sqlite3`.

<details>
<summary><b>Показать шаблон xbps-src</b></summary>

```bash
# Файл шаблона для 'omniroute'
pkgname=omniroute
version=3.8.0
revision=1
hostmakedepends="nodejs python3 make"
depends="openssl"
short_desc="Universal AI gateway with smart routing for multiple LLM providers"
maintainer="zenobit <zenobit@disroot.org>"
license="MIT"
homepage="https://github.com/diegosouzapw/OmniRoute"
distfiles="https://github.com/diegosouzapw/OmniRoute/archive/refs/tags/v${version}.tar.gz"
checksum=009400afee90a9f32599d8fe734145cfd84098140b7287990183dde45ae2245b
system_accounts="_omniroute"
omniroute_homedir="/var/lib/omniroute"
export NODE_ENV=production
export npm_config_engine_strict=false
export npm_config_loglevel=error
export npm_config_fund=false
export npm_config_audit=false

do_build() {
	# Определить целевую архитектуру CPU для node-gyp
	local _gyp_arch
	case "$XBPS_TARGET_MACHINE" in
		aarch64*) _gyp_arch=arm64 ;;
		armv7*|armv6*) _gyp_arch=arm ;;
		i686*) _gyp_arch=ia32 ;;
		*) _gyp_arch=x64 ;;
	esac

	# 1) Установить все зависимости – пропустить скрипты
	NODE_ENV=development npm ci --ignore-scripts

	# 2) Собрать standalone-бандл Next.js
	npm run build

	# 3) Скопировать статические ресурсы в standalone
	cp -r .next/static .next/standalone/.next/static
	[ -d public ] && cp -r public .next/standalone/public || true

	# 4) Скомпилировать нативный биндинг better-sqlite3
	local _node_gyp=/usr/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js
	(cd node_modules/better-sqlite3 && node "$_node_gyp" rebuild --arch="$_gyp_arch")

	# 5) Поместить скомпилированный биндинг в standalone-бандл
	local _bs3_release=.next/standalone/node_modules/better-sqlite3/build/Release
	mkdir -p "$_bs3_release"
	cp node_modules/better-sqlite3/build/Release/better_sqlite3.node "$_bs3_release/"

	# 6) Удалить архитектурно-специфичные бандлы sharp
	rm -rf .next/standalone/node_modules/@img

	# 7) Скопировать runtime-зависимости pino, пропущенные статическим анализом Next.js:
	for _mod in pino-abstract-transport split2 process-warning; do
		cp -r "node_modules/$_mod" .next/standalone/node_modules/
	done
}

do_check() {
	npm run test:unit
}

do_install() {
	vmkdir usr/lib/omniroute/.next
	vcopy .next/standalone/. usr/lib/omniroute/.next/standalone

	# Предотвратить удаление пустых каталогов app router Next.js хуком post-install
	for _d in \
		.next/standalone/.next/server/app/dashboard \
		.next/standalone/.next/server/app/dashboard/settings \
		.next/standalone/.next/server/app/dashboard/providers; do
		touch "${DESTDIR}/usr/lib/omniroute/${_d}/.keep"
	done

	cat > "${WRKDIR}/omniroute" <<'EOF'
#!/bin/sh
export PORT="${PORT:-20128}"
export DATA_DIR="${DATA_DIR:-${XDG_DATA_HOME:-${HOME}/.local/share}/omniroute}"
export APP_LOG_TO_FILE="${APP_LOG_TO_FILE:-false}"
mkdir -p "${DATA_DIR}"
exec node /usr/lib/omniroute/.next/standalone/server.js "$@"
EOF
	vbin "${WRKDIR}/omniroute"
}

post_install() {
	vlicense LICENSE
}
```

</details>

### Переменные окружения

| Переменная                              | По умолчанию                         | Описание                                                                                                  |
| --------------------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| `JWT_SECRET`                            | `omniroute-default-secret-change-me` | Секрет подписи JWT (**смените в продакшене**)                                                             |
| `INITIAL_PASSWORD`                      | `CHANGEME`                           | Пароль первого входа                                                                                      |
| `DATA_DIR`                              | `~/.omniroute`                       | Каталог данных (БД, использование, логи)                                                                  |
| `PORT`                                  | значение фреймворка                  | Порт сервиса (`20128` в примерах)                                                                         |
| `HOSTNAME`                              | значение фреймворка                  | Хост привязки (в Docker по умолчанию `0.0.0.0`)                                                           |
| `NODE_ENV`                              | значение среды                       | Установите `production` для развёртывания                                                                 |
| `NEXT_PUBLIC_BASE_URL`                  | `http://localhost:20128`             | Публичный base URL, отображаемый на панели и доступный серверу (заменяет устаревший `BASE_URL`)           |
| `NEXT_PUBLIC_CLOUD_URL`                 | `https://omniroute.dev`              | Base URL эндпоинта облачной синхронизации (заменяет устаревший `CLOUD_URL`)                               |
| `API_KEY_SECRET`                        | `endpoint-proxy-api-key-secret`      | Секрет HMAC для сгенерированных API-ключей                                                                |
| `REQUIRE_API_KEY`                       | `false`                              | Требовать Bearer API-ключ на `/v1/*`                                                                      |
| `ALLOW_API_KEY_REVEAL`                  | `false`                              | Разрешить аутентифицированным пользователям панели раскрывать полные сохранённые значения API-ключей по запросу |
| `PROVIDER_LIMITS_SYNC_INTERVAL_MINUTES` | `70`                                 | Серверная периодичность обновления кэшированных данных Provider Limits; кнопки обновления в UI по-прежнему запускают синхронизацию вручную |
| `DISABLE_SQLITE_AUTO_BACKUP`            | `false`                              | Отключить автоматические снимки SQLite перед записью/импортом/восстановлением; ручные копии по-прежнему работают |
| `APP_LOG_TO_FILE`                       | `true`                               | Включает запись логов приложения и аудита на диск                                                         |
| `AUTH_COOKIE_SECURE`                    | `false`                              | Принудительный cookie `Secure` для авторизации (за обратным прокси HTTPS)                                 |
| `CLOUDFLARED_BIN`                       | не задан                             | Использовать существующий бинарник `cloudflared` вместо управляемого скачивания                           |
| `CLOUDFLARED_PROTOCOL`                  | `http2`                              | Транспорт для управляемых Quick Tunnels (`http2`, `quic` или `auto`)                                      |
| `OMNIROUTE_MEMORY_MB`                   | `512`                                | Лимит кучи Node.js в МБ                                                                                   |
| `PROMPT_CACHE_MAX_SIZE`                 | `50`                                 | Макс. записей кэша промптов                                                                               |
| `SEMANTIC_CACHE_MAX_SIZE`               | `100`                                | Макс. записей семантического кэша                                                                         |

Полный справочник переменных окружения см. в [README](../README.md).

---

## 📊 Доступные модели

<details>
<summary><b>Показать все доступные модели</b></summary>

> Список ниже составлен из `open-sse/config/providerRegistry.ts` для v3.8.0. Облачные каталоги (Gemini, OpenRouter и т.д.) синхронизируются динамически — полный живой каталог см. в **Панель → Providers → [провайдер] → Available Models** или вызовите `GET /api/models/catalog`.

**Claude Code (`cc/`)** — Pro/Max OAuth: `cc/claude-opus-4-8`, `cc/claude-opus-4-7`, `cc/claude-opus-4-6`, `cc/claude-opus-4-5-20251101`, `cc/claude-sonnet-4-6`, `cc/claude-sonnet-4-5-20250929`, `cc/claude-haiku-4-5-20251001`

**Codex (`cx/`)** — Plus/Pro OAuth: `cx/gpt-5.5` (+ уровни усилия: `gpt-5.5-xhigh`, `gpt-5.5-high`, `gpt-5.5-medium`, `gpt-5.5-low`), `cx/gpt-5.4`, `cx/gpt-5.4-mini`, `cx/gpt-5.3-codex`, `cx/gpt-5.3-codex-spark`

**GitHub Copilot (`gh/`)** — OAuth: `gh/gpt-5.5`, `gh/gpt-5.4`, `gh/gpt-5.4-mini`, `gh/gpt-5-mini`, `gh/gpt-5.3-codex`, `gh/claude-opus-4.7`, `gh/claude-opus-4.6`, `gh/claude-opus-4-5-20251101`, `gh/claude-sonnet-4.6`, `gh/claude-sonnet-4.5`, `gh/claude-haiku-4.5`, `gh/gemini-3.1-pro-preview`, `gh/gemini-3-flash-preview`, `gh/oswe-vscode-prime`

**Kiro (`kr/`)** — БЕСПЛАТНЫЙ OAuth: `kr/auto-kiro`, `kr/claude-opus-4.7`, `kr/claude-opus-4.6`, `kr/claude-sonnet-4.6`, `kr/claude-sonnet-4.5`, `kr/claude-haiku-4.5`, `kr/deepseek-3.2`, `kr/minimax-m2.5`, `kr/minimax-m2.1`, `kr/glm-5`, `kr/qwen3-coder-next`

**Qoder (`if/`)** — БЕСПЛАТНЫЙ OAuth: `if/kimi-k2-0905`, `if/kimi-k2`, `if/qwen3-coder-plus`, `if/qwen3-max`, `if/qwen3-max-preview`, `if/qwen3-vl-plus`, `if/qwen3-32b`, `if/qwen3-235b-a22b-thinking-2507`, `if/qwen3-235b-a22b-instruct`, `if/qwen3-235b`, `if/deepseek-v3.2`, `if/deepseek-v3`, `if/deepseek-r1`, `if/qoder-rome-30ba3b`

**Qwen (`qw/`)** — БЕСПЛАТНЫЙ OAuth (chat.qwen.ai): `qw/coder-model`, `qw/vision-model`

**GLM (`glm/`, `glm-cn/`, `zai/`, `glmt/`)** — $0.2–0.6/1M: `glm/glm-5.1`, `glm/glm-5`, `glm/glm-5-turbo`, `glm/glm-4.7`, `glm/glm-4.7-flash`, `glm/glm-4.6`, `glm/glm-4.6v`, `glm/glm-4.5`, `glm/glm-4.5v`, `glm/glm-4.5-air`

**MiniMax (`minimax/`, `minimax-cn/`)** — $0.2/1M: `minimax/MiniMax-M2.7`, `minimax/MiniMax-M2.7-highspeed`, `minimax/MiniMax-M2.5`, `minimax/MiniMax-M2.5-highspeed`

**Kimi (`kimi/`, `kimi-coding/`, `kimi-coding-apikey/`)** — $9/мес. фикс или по использованию: `kimi/kimi-k2.6`, `kimi/kimi-k2.5`

**DeepSeek (`ds/`)** — API-ключ: `ds/deepseek-v4-pro`, `ds/deepseek-v4-flash`

**Groq (`groq/`)** — ультрабыстрый: `groq/llama-3.3-70b-versatile`, `groq/meta-llama/llama-4-maverick-17b-128e-instruct`, `groq/qwen/qwen3-32b`, `groq/openai/gpt-oss-120b`

**xAI (`xai/`)** — нативный Grok: `xai/grok-4.3`, `xai/grok-4.20-multi-agent-0309`, `xai/grok-4.20-0309-reasoning`, `xai/grok-4.20-0309-non-reasoning`

**Mistral (`mistral/`)** — размещён в ЕС: `mistral/mistral-large-latest`, `mistral/mistral-medium-3-5`, `mistral/mistral-small-latest`, `mistral/devstral-latest`, `mistral/codestral-latest`

**Perplexity (`pplx/`)** — с поиском: `pplx/sonar-deep-research`, `pplx/sonar-reasoning-pro`, `pplx/sonar-pro`, `pplx/sonar`

**Together AI (`together/`)** — открытые модели: `together/meta-llama/Llama-3.3-70B-Instruct-Turbo-Free` (бесплатно), `together/meta-llama/Llama-Vision-Free`, `together/deepseek-ai/DeepSeek-R1-Distill-Llama-70B-Free`, `together/deepseek-ai/DeepSeek-R1`, `together/Qwen/Qwen3-235B-A22B`, `together/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8`

**Fireworks AI (`fireworks/`)** — быстрый инференс: `fireworks/accounts/fireworks/models/kimi-k2p6`, `fireworks/accounts/fireworks/models/minimax-m2p7`, `fireworks/accounts/fireworks/models/qwen3p6-plus`, `fireworks/accounts/fireworks/models/glm-5p1`, `fireworks/accounts/fireworks/models/deepseek-v4-pro`

**Cerebras (`cerebras/`)** — wafer-scale: `cerebras/zai-glm-4.7`, `cerebras/gpt-oss-120b`

**Cohere (`cohere/`)** — ориентирован на RAG: `cohere/command-a-reasoning-08-2025`, `cohere/command-a-vision-07-2025`, `cohere/command-a-03-2025`, `cohere/command-r-08-2024`

**NVIDIA NIM (`nvidia/`)** — корпоративный: `nvidia/z-ai/glm-5.1`, `nvidia/minimaxai/minimax-m2.7`, `nvidia/google/gemma-4-31b-it`, `nvidia/mistralai/mistral-small-4-119b-2603`, `nvidia/mistralai/mistral-large-3-675b-instruct-2512`, `nvidia/qwen/qwen3.5-397b-a17b`, `nvidia/deepseek-ai/deepseek-v4-pro`, `nvidia/openai/gpt-oss-120b`, `nvidia/nvidia/nemotron-3-super-120b-a12b`

**Baidu Qianfan (`qianfan/`)** — ERNIE: `qianfan/ernie-5.1`, `qianfan/ernie-5.0-thinking-latest`, `qianfan/ernie-x1.1`

**Ollama Cloud (`ollama-cloud/`)**: `ollama-cloud/deepseek-v4-pro`, `ollama-cloud/deepseek-v4-flash`, `ollama-cloud/kimi-k2.6`, `ollama-cloud/glm-5.1`, `ollama-cloud/minimax-m2.7`, `ollama-cloud/gemma4:31b`, `ollama-cloud/qwen3.5:397b`

**Gemini (Google Cloud `gemini/`)**: синхронизируется живьём для каждого API-ключа из Google — статического списка нет. Подключите ключ в **Панель → Providers**, затем используйте **Available Models**, чтобы импортировать текущий каталог (напр. `gemini/gemini-3-pro`, `gemini/gemini-3-flash`).

**Другие совместимые провайдеры** (выборочно): `cohere`, `databricks`, `snowflake`, `together`, `vertex`, `alibaba`, `alibaba-cn`, `bedrock` (через `aws-bedrock`), `azure-ai`, `openrouter` (сквозной каталог), `siliconflow`, `hyperbolic`, `huggingface`, `featherless-ai`, `cloudflare-ai`, `scaleway`, `deepinfra`, `vercel-ai-gateway`, `bazaarlink`, `friendliai`, `nous-research`, `reka`, `volcengine`, `ai21`, `gigachat`. Каждый ведёт собственный список моделей в `providerRegistry.ts` и может автосинхронизироваться, когда провайдер предоставляет эндпоинт `/models`.

**Замечание об ID моделей:** OmniRoute использует нативные для провайдеров ID (`claude-opus-4-8`, `gpt-5.5`, `glm-5.1`, `MiniMax-M2.7`, `kimi-k2.5`, `grok-4.20-0309-reasoning`). Некоторые ID содержат точечные версии, потому что именно так их ожидает апстрим API. Если модели нет в списке выше, выполните `omniroute models --search <term>` или обратитесь к `GET /api/models/catalog`, чтобы подтвердить доступность.

</details>

---
