---
title: "Удалённый режим — управляйте удалённым OmniRoute с вашего ноутбука"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Удалённый режим

Запускайте CLI `omniroute` на вашем ноутбуке, в то время как сам OmniRoute работает где-то ещё
(VPS, домашний сервер, другая машина в вашей Tailnet). Вы входите один раз через
`omniroute connect`, и с этого момента **каждая** команда CLI нацелена на этот удалённый
сервер — те же команды, тот же вывод, просто выполняется против удалённого сервера.

Никакого второго инструмента устанавливать не нужно: удалённый режим — это обычный CLI `omniroute`
плюс **токены доступа** с областью.

```bash
npm install -g omniroute                 # обычный CLI
omniroute connect 192.168.0.15           # вход (пароль → токен с областью)
omniroute models list                    # ← теперь перечисляет модели УДАЛЁННОГО сервера
omniroute configure codex                # ← пишет локальный профиль Codex из удалённого каталога
```

---

## Как это работает

```
ваш ноутбук                             удалённый OmniRoute (VPS)
┌────────────────────┐                   ┌───────────────────────────────┐
│ omniroute CLI      │  POST /api/cli/connect  (пароль → токен)           │
│  context: vps      │ ───────────────►  │ выпускает токен доступа с областью │
│  baseUrl, token    │  Authorization: Bearer oma_live_…                  │
│                    │ ───────────────►  │ каждый управленческий маршрут, │
│ пишет конфиги      │ ◄───────────────  │ проверка по области токена     │
│ ЛОКАЛЬНО           │                   └───────────────────────────────┘
└────────────────────┘
```

- **Контексты** хранят по одному серверу (`~/.omniroute/config.json`, `chmod 600`).
  `omniroute contexts use <name>` переключает активный сервер; `default` — локальный.
- **Токены доступа** (`oma_live_…`) авторизуют управленческие команды. Они
  отличаются от API-ключей инференса (`sk-…`, используются для `/v1/chat/completions`).
- На сервере хранится только SHA-256-хэш токена. Открытый текст показывается
  **один раз**, при создании.

---

## Подключение

### С управленческим паролем (начальная загрузка)

```bash
omniroute connect 192.168.0.15
# Management password for http://192.168.0.15:20128: ********
# ✔ Connected to http://192.168.0.15:20128 — context '192.168.0.15' (scope: admin)
```

Поток с паролем по умолчанию выпускает токен **admin** (у вас есть пароль, значит
у вас уже есть полный контроль). Понизьте область через `--scope`:

```bash
omniroute connect 192.168.0.15 --scope write
```

Опции: `--port <p>` (когда у хоста его нет), `--name <ctx>` (имя контекста),
`--scope read|write|admin`. Полный URL принимается как есть:
`omniroute connect https://omni.example.com`.

### С заранее сгенерированным токеном

Сгенерируйте токен с областью в панели управления (или через `omniroute tokens create`) и
вставьте его — пароль не нужен:

```bash
omniroute connect 192.168.0.15 --key oma_live_xxxxxxxx
```

CLI валидирует его через `GET /api/cli/whoami` и сохраняет как активный контекст.

---

## Области

Три уровня, иерархически (`admin ⊃ write ⊃ read`):

| Область | Что можно                                                                     |
| ------- | ----------------------------------------------------------------------------- |
| `read`  | список/просмотр — `models list`, `providers status`, `logs`, `usage`, `cost`  |
| `write` | read **+** настройка/применение — `setup-codex`, `keys add`, `config set`, комбо |
| `admin` | write **+** управление — CRUD `tokens`, добавление провайдеров, сервисы, политики, oauth |

Сервер выводит область, необходимую каждому маршруту, из HTTP-метода
(`GET`→read, мутации→write) плюс административный белый список для чувствительных поверхностей
(`/api/cli/tokens`, мутации `/api/providers`, `/api/oauth`, `/api/services`, …).
Токен с недостаточной областью получает `403` с понятным сообщением.

> Маршруты, порождающие процессы (`/api/services/*`, `/api/mcp/*`, …), остаются
> **доступными только с loopback** — удалённый токен никогда не сможет до них достучаться, независимо от области.

---

## Подключение Antigravity на удалённой установке

Antigravity использует экран согласия Google firstparty/nativeapp. Google
выдаёт код авторизации только когда **loopback-редирект**
(`http://127.0.0.1:<port>/callback`) **доступен из браузера, который
подтверждает вход**. При удалённой установке на VPS этот loopback находится на
сервере, а не на вашей машине, поэтому экран согласия **зависает навсегда и никогда
не выдаёт код** — обычному откату «вставьте URL callback» нечего вставлять. (Это ограничение со стороны Google: то же зависание происходит в любом прокси,
использующем встроенный десктопный клиент Antigravity, не только в OmniRoute.)

Есть два поддерживаемых способа подключить Antigravity к удалённому OmniRoute.

### Вариант A — локальный помощник входа (рекомендуется)

Выполните OAuth на **вашем собственном компьютере**, где `127.0.0.1` доступен, и вставьте
результат в удалённую панель управления. Помощник общается только с Google — ему
**не** нужен сетевой доступ к вашему VPS, поэтому он работает даже за файрволами.

```bash
# На вашей ЛОКАЛЬНОЙ машине (нужны Node.js + браузер):
npx omniroute login antigravity
#   ↳ открывает экран согласия Google в вашем браузере, перехватывает callback на локальном
#     loopback-порту, обменивает его и печатает однострочный blob учётных данных:
#
#   omniroute-cred-v1.eyJ2IjoxLCJ...
```

Затем в **удалённой** панели управления: **Providers → Antigravity → Connect** и
вставьте blob `omniroute-cred-v1.…` в поле **Step 2** (оно принимает как
URL callback, так и blob учётных данных). OmniRoute декодирует его, выполняет
онбординг Cloud Code на стороне сервера и сохраняет подключение.

> Blob содержит refresh-токен — обращайтесь с ним как с паролем. Он передаётся один
> раз по вашему соединению с панелью и хранится в зашифрованном виде.

Флаги: `--no-browser` (вывести URL вместо автооткрытия), `--port <n>`
(зафиксировать loopback-порт), `--timeout <ms>`.

### Вариант B — SSH-туннель с локальной пересылкой

Если у вас есть SSH-доступ к VPS, пробросьте порт панели управления, чтобы
loopback-callback доходил обратно до сервера через туннель:

```bash
# На вашей ЛОКАЛЬНОЙ машине:
ssh -L 20128:localhost:20128 user@your-vps
# затем откройте http://localhost:20128 в вашем ЛОКАЛЬНОМ браузере и подключите Antigravity
# как обычно — редирект 127.0.0.1:20128/callback теперь достигает VPS через SSH.
```

Поскольку вы обращаетесь к панели как `localhost:20128`, согласие Google
завершается, а callback доставляется на сервер через тот же туннель —
blob не нужен. Держите туннель открытым, пока подключение не покажется активным.

> Полностью headless-альтернатива (без помощника, без туннеля) — настроить **собственные**
> веб-учётные данные Google OAuth + публичный base URL; см. переменные окружения OAuth
> провайдера. Два варианта выше не требуют дополнительной настройки Google.

---

## Управление токенами

```bash
omniroute tokens create --name "laptop" --scope write [--expires 30]
#   ↳ печатает секрет ОДИН РАЗ — скопируйте его сейчас
omniroute tokens list                 # маскированно: id, имя, область, префикс, статус, истечение
omniroute tokens revoke <id|prefix>   # отозвать немедленно
omniroute tokens scopes               # объяснить три области
```

Команды `tokens` требуют учётных данных **admin**. Токены также можно управлять в
панели управления в **Settings → Access Tokens** (создание, отзыв, одноразовое копирование).

---

## Настройка CLI для кодирования из удалённого каталога

`omniroute configure` читает живой каталог моделей **активного сервера** и пишет
конфиг на **вашей** машине.

```bash
omniroute configure codex
#   Providers: glm, kmc, ollamacloud, opencode-go, …
#   Provider: glm
#   Model id: glm/glm-5.2
#   ✔ Wrote ~/.codex/glm52.config.toml
#   Use it:  codex --profile glm52

# неинтерактивно
omniroute configure codex --provider glm --model glm/glm-5.2 --name glm52
```

Записанный профиль ссылается на ключ инференса через переменную окружения
(`OMNIROUTE_API_KEY`) — секрет никогда не записывается на диск. Разовую
базовую настройку Codex (блок `[model_providers.omniroute]`) см. в
[CODEX-CLI-CONFIGURATION.md](./CODEX-CLI-CONFIGURATION.md).

### Команды настройки для каждого CLI

У каждого поддерживаемого CLI есть команда настройки с поддержкой удалённого режима (все учитывают активный
контекст или `--remote <url> --api-key <key>`):

| CLI | Команда | Что записывает |
|-----|---------|----------------|
| Codex | `omniroute setup-codex` | Профили `~/.codex/<name>.config.toml` (на каждую модель) |
| Claude Code | `omniroute setup-claude` | `~/.claude/profiles/<name>/settings.json` (на каждую модель) |
| OpenCode | `omniroute setup-opencode` | `~/.config/opencode/opencode.json` — провайдер `omniroute` openai-совместимый со всеми моделями каталога (запуск `opencode -m omniroute/<model>`) |
| Cline | `omniroute setup-cline` | `~/.cline/data/{globalState,secrets}.json` (режим CLI) + печатает настройки расширения VS Code для вставки (OpenAI-совместимый, Base URL **без** `/v1`) |
| Kilo Code | `omniroute setup-kilo` | `~/.local/share/kilo/auth.json` (CLI) + настройки VS Code `kilocode.*` — OpenAI-совместимый, Base URL **с** `/v1` |
| Continue | `omniroute setup-continue` | `~/.continue/config.yaml` (VS Code/JetBrains + CLI `cn`) — `provider: openai`, `apiBase` **с** `/v1`, ключ через `${{ secrets.OMNIROUTE_API_KEY }}` |
| Cursor | `omniroute setup-cursor` | печатает шаги внутри приложения (Settings → Models → Override OpenAI Base URL **с** `/v1` + ключ + модель). Конфиг Cursor — непрозрачный SQLite — только панель чата |
| Roo Code | `omniroute setup-roo` | пишет JSON для импорта Roo (`~/.omniroute/roo-settings.json`) + задаёт `roo-cline.autoImportSettingsPath` + печатает шаги UI (OpenAI-совместимый, Base URL **с** `/v1`) |
| Crush | `omniroute setup-crush` | `~/.config/crush/crush.json` — провайдер `openai-compat`, `base_url` **с** `/v1`, ключ через `$OMNIROUTE_API_KEY` |
| Goose | `omniroute setup-goose` | `~/.config/goose/config.yaml` (`GOOSE_PROVIDER=openai` + `OPENAI_HOST` **без** `/v1` + `GOOSE_MODEL`) + рецепт окружения |
| Qwen Code | `omniroute setup-qwen` | `~/.qwen/settings.json` — openai `modelProvider`, `baseUrl` **с** `/v1`, ключ через `envKey` (OMNIROUTE_API_KEY) |
| Aider | `omniroute setup-aider` | `~/.aider.conf.yml` (`openai-api-base` **без** `/v1` + `model: openai/<id>`) + рецепт окружения (`aider --message --yes`) |

```bash
# OpenCode (openai-совместимый провайдер, все модели каталога, удалённый VPS)
omniroute setup-opencode --remote http://192.168.0.15:20128 --api-key oma_live_xxx
omniroute setup-opencode --only glm,kimi        # оставить только совпадающие модели
opencode -m omniroute/glm/glm-5.2 "..."          # сначала export OMNIROUTE_API_KEY
```

> У OpenCode также есть более богатая **плагинная** интеграция: `omniroute setup opencode`
> (теперь с поддержкой удалённого режима через `--remote`) устанавливает `@omniroute/opencode-plugin`.
> `setup-opencode` — облегчённая openai-совместимая альтернатива. API-ключ
> указывается через `{env:OMNIROUTE_API_KEY}` — никогда не записывается на диск.

---

## Управление контекстами (переключение между серверами)

**Контекст** — это сохранённый сервер (baseUrl + учётные данные + область). `omniroute connect`
создаёт его и делает активным; с этого момента каждая команда нацелена на него. Управляйте ими и
переключайтесь между ними через `omniroute contexts`:

```bash
omniroute contexts list            # все контексты; активный отмечен ●
omniroute contexts current         # активный сервер, статус авторизации, область
```

```text
  | Name    | Base URL                  | Auth  | Scope | Description
● | vps     | http://100.67.86.91:20128 | token | admin | Remote OmniRoute (…)
  | default | http://localhost:20128    | ✗     |       |
```

**Переключение серверов** — каждая последующая команда следует активному контексту:

```bash
omniroute contexts use vps         # → все команды теперь идут на удалённый VPS
omniroute tokens list              #   (выполняется против VPS)

omniroute contexts use default     # → обратно на localhost
omniroute tokens list              #   (выполняется против локального сервера)
```

**Добавить контекст вручную** (вместо `connect`), просмотреть или переименовать:

```bash
omniroute contexts add staging --url https://staging.example.com:20128 \
  --access-token oma_live_xxxx --scope write --description "staging box"
omniroute contexts show staging    # полные детали одного контекста
omniroute contexts rename staging stg
```

**Удалить контекст** — запрашивает подтверждение; передайте `--yes`, чтобы пропустить
(обязательно для скриптов / неинтерактивных оболочек, которые иначе безопасно отклоняются):

```bash
omniroute contexts remove stg --yes
```

> `default` (localhost) удалить нельзя. Удаление активного контекста откатывает к
> `default`. Совет: удаление контекста лишь отбрасывает **локально** сохранённые учётные данные —
> отзовите токен на сервере через `omniroute tokens revoke <id>`, чтобы действительно
> прекратить доступ.

**Экспорт / импорт** контекстов (напр. для переноса между машинами — секреты включены,
поэтому обращайтесь с файлом осторожно):

```bash
omniroute contexts export --out contexts.json     # по умолчанию: stdout
omniroute contexts import contexts.json            # перезапись; --merge — сохранить существующие
```

---

## Быстрая сквозная проверка

Готовый к копированию жизненный цикл для проверки удалённой настройки с нуля — подключиться, выпустить
токен с областью, выполнить команду, переключиться обратно и демонтировать. Замените
`192.168.0.15` на хост/IP вашего сервера (Tailscale, LAN или публичный
URL `https://…`).

```bash
# 1. Подключение (пароль → токен admin, сохраняется как контекст, который становится активным)
omniroute connect 192.168.0.15                 # или: --key oma_live_xxxx  (без пароля)
omniroute contexts current                     # показывает удалённый сервер + область

# 2. Использование — управленческие команды теперь выполняются против удалённого
omniroute tokens create --name laptop --scope read   # выпустить более узкий токен
omniroute tokens list                                 # маскированный список с удалённого

# 3. Переключение туда-обратно
omniroute contexts use default                 # → локальный
omniroute contexts use 192-168-0-15            # → снова удалённый (имя из `contexts list`)

# 4. Демонтаж. ВНИМАНИЕ: `contexts remove` удаляет только ЛОКАЛЬНЫЕ учётные данные —
#    токен на сервере оно НЕ отзывает. Сначала отзовите на стороне сервера, если
#    хотите действительно прекратить доступ.
omniroute tokens revoke <id|prefix>            # прекращает доступ на сервере
omniroute contexts remove 192-168-0-15 --yes   # удалить локальный контекст (даже активный → откат к default), без запроса
```

> `--yes` делает `contexts remove` неинтерактивным (обязательно в скриптах/CI; без него
> неинтерактивная оболочка безопасно отклоняет вместо зависания). Удаление
> **активного** контекста автоматически откатывает к `default`.

---

## Замечания по безопасности

- Открытый текст токена показывается один раз; сохраняется только SHA-256-хэш (как и для API-ключей).
- `omniroute connect` переиспользует защиту от перебора при входе + аудит-логирование.
- Предпочитайте HTTPS или Tailnet для транспорта; голый хост по умолчанию использует `http://`
  для удобства в LAN/Tailscale — передайте полный URL `https://…` для TLS.
- Локальный файл контекста — `~/.omniroute/config.json` (`chmod 600`); токены
  никогда не печатаются в логах (маскируются до префикса).

---

## Эндпоинты API (справочник)

| Метод  | Маршрут               | Авторизация         | Область                    |
| ------ | --------------------- | ------------------- | -------------------------- |
| POST   | `/api/cli/connect`    | управленческий пароль | — (публичный, защищён паролем) |
| GET    | `/api/cli/whoami`     | токен доступа       | read                       |
| GET    | `/api/cli/tokens`     | токен доступа       | admin                      |
| POST   | `/api/cli/tokens`     | токен доступа       | admin                      |
| DELETE | `/api/cli/tokens/:id` | токен доступа       | admin                      |

Полные схемы см. в [openapi.yaml](../openapi.yaml).
