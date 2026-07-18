---
title: "Руководство по скрытности"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Руководство по скрытности (Stealth)

> **Источник истины:** `open-sse/utils/tlsClient.ts`, `open-sse/services/{chatgptTlsClient,claudeCodeCCH,claudeCodeFingerprint,claudeCodeObfuscation,claudeCodeCompatible,antigravityObfuscation}.ts`, `open-sse/config/cliFingerprints.ts`, `src/mitm/`
> **Последнее обновление:** 2026-06-28 — v3.8.40
> **Аудитория:** инженеры, поддерживающие провайдер-специфичные скрытые интеграции.

OmniRoute интегрируется с провайдерами, чьи точки присутствия активно снимают отпечатки неофициальных клиентов (TLS JA3/JA4, порядок заголовков, форма JSON-тела, токены целостности). Эта страница документирует поверхности скрытности, которые предоставляет OmniRoute, и где они реализованы.

## Юридическое и этическое уведомление

Функции скрытности существуют, чтобы OmniRoute мог выступать слоем совместимости между официальными аккаунтами, принадлежащими пользователям (Claude Code CLI, ChatGPT Desktop/Web, Antigravity, Cursor и т.д.), и унифицированным API OmniRoute. Они **не** предназначены для обхода фрод-детектирования, совместного использования учётных данных или нарушения условий использования провайдеров. Мейнтейнеры ожидают, что операторы будут соблюдать условия использования апстримов, которые они подписали при создании аккаунтов.

---

## Слой TLS-отпечатков

### `open-sse/utils/tlsClient.ts` — wreq-js (Chrome 124)

Лениво загружаемая сессия `wreq-js`, имитирующая **Chrome 124 на macOS**. Используется как универсальная обёртка JA3/JA4 для апстримов за Cloudflare. Откатывается на нативный fetch, когда `wreq-js` не установлен (`available = false`).

- Сессия-синглтон: `browser: "chrome_124", os: "macos"`
- Разрешение прокси (приоритет): `HTTPS_PROXY` → `HTTP_PROXY` → `ALL_PROXY` (также в нижнем регистре)
- Таймаут: `TLS_CLIENT_TIMEOUT_MS` (наследуется от `FETCH_TIMEOUT_MS`, по умолчанию 600000)
- Response `wreq-js` совместим с fetch (`headers`, `text()`, `json()`, `clone()`, `body`).

### `open-sse/services/chatgptTlsClient.ts` — tls-client-node (Firefox 148)

Выделенный имитатор TLS для `chatgpt.com`. Конфигурация Cloudflare ChatGPT привязывает `cf_clearance` к JA3/JA4 + порядку кадров HTTP/2 SETTINGS — рукопожатие undici получает `cf-mitigated: challenge` даже с валидными cookie.

- Профиль: `firefox_148` (должен совпадать с отправляемым `User-Agent` Firefox 148)
- Режим: `runtimeMode: "native"` (разделяемая библиотека, загружаемая через koffi; избегает управляемого HTTP-сайдкара)
- `withRandomTLSExtensionOrder: true`
- `tlsFetchChatGpt(url, options)` поддерживает стриминг (записывает тело во временный файл, читается как `ReadableStream`)
- Обнаружение зависаний: `raceWithTimeout` + `TlsClientHangError` запускает `resetClientCache()`, чтобы следующий вызов пересоздал привязку
- Разрешение прокси (приоритет): `proxyUrl` вызова → `OMNIROUTE_TLS_PROXY_URL` → `HTTPS_PROXY`/`HTTP_PROXY`/`ALL_PROXY` (нативная привязка сама **не** читает эти переменные; их нужно прокидывать)
- Ошибки: `TlsClientUnavailableError` (бинарник отсутствует), `TlsClientHangError` (привязка в дедлоке)

---

## Пакет скрытности Claude Code

Когда включён `cliCompatMode`, OmniRoute переформировывает исходящие запросы Claude, чтобы они были неотличимы от трафика `claude-cli`. Три модуля работают совместно:

### `claudeCodeFingerprint.ts`

Вычисляет 3-символьный отпечаток `cc_version`, встраиваемый в биллинговый заголовок:

```
SHA256(SALT + msg[4] + msg[7] + msg[20] + version)[:3]
```

- `FINGERPRINT_SALT = "59cf53e54c78"` (захардкожена; совпадает с официальным клиентом)
- Входы: символы по индексам 4, 7, 20 текста первого пользовательского сообщения + строка версии
- Выход: 3-символьный hex-префикс

### `claudeCodeCCH.ts` (Client Content Hash)

Серверная проверка целостности, которую официальный Claude Code CLI вычисляет через Bun/Zig. OmniRoute реализует заново с `xxhash-wasm`:

1. Сериализовать тело с заглушкой `cch=00000;`
2. `xxhash64(bytes, seed) & 0xFFFFF`
3. 5-символьный hex в нижнем регистре с нулями слева
4. Заменить `cch=00000;` вычисленным токеном

Константы:

- Сид: `0x6e52736ac806831e`
- Паттерн: `/\bcch=([0-9a-f]{5});/`

### `claudeCodeObfuscation.ts`

Вставляет Unicode **zero-width joiner** (`U+200D`) после первого символа «чувствительных» имён клиентов, чтобы апстрим-фильтры не могли их найти grep'ом. Список слов по умолчанию:

```
opencode, open-code, cline, roo-cline, roo_cline, cursor, windsurf,
aider, continue.dev, copilot, avante, codecompanion
```

Применяется к: блокам `system`, всем `messages[].content` и `tools[].description` / `tools[].function.description`. Переопределяется оператором через `setSensitiveWords()`.

### `claudeCodeCompatible.ts` — провайдеры `anthropic-compatible-cc-*`

Для сторонних ретрансляторов Anthropic, принимающих только «настоящий Claude Code» трафик:

- `CLAUDE_CODE_COMPATIBLE_USER_AGENT = "claude-cli/2.1.207 (external, sdk-cli)"`
- `CLAUDE_CODE_COMPATIBLE_STAINLESS_PACKAGE_VERSION = "0.94.0"`
- `CLAUDE_CODE_COMPATIBLE_STAINLESS_RUNTIME_VERSION = "v24.3.0"`
- `anthropic-beta = "claude-code-20250219,interleaved-thinking-2025-05-14,effort-2025-11-24"` по умолчанию
- Переключатель «Enable redact-thinking beta» на подключение добавляет `redact-thinking-2026-02-12`, когда апстрим CC Compatible особо требует потоки с редактированными рассуждениями
- Переключатель «Enable summarized thinking display» на подключение сохраняет `providerSpecificData.requestDefaults.summarizeThinking` и добавляет `display: "summarized"` к запросам CC Compatible thinking, у которых не задан режим отображения
- `CONTEXT_1M_BETA_HEADER = "context-1m-2025-08-07"` (семейство Opus/Sonnet 4.x)
- Путь по умолчанию: `/v1/messages?beta=true`

Родственные модули в том же пакете:

- `claudeCodeConstraints.ts` — правила temperature + cache-control
- `claudeCodeToolRemapper.ts` — переназначение имён инструментов
- `claudeCodeExtraRemap.ts` — дополнительная нормализация нагрузки

---

## Скрытность Antigravity

### `antigravityObfuscation.ts`

Тот же трюк с zero-width-joiner, что у Claude Code, но с расширенным списком слов, который также маскирует: `claude code`, `claude-code`, `kilo code`, `kilocode`, **`omniroute`**. Зеркалирует `ZEROGRAVITY_SENSITIVE_WORDS` от ZeroGravity и систему маскировки CLIProxyAPI.

### `antigravityHeaderScrub.ts`

Вырезает маркеры Stainless SDK (`x-stainless-lang`, `x-stainless-package-version`, `x-stainless-os`, `x-stainless-arch`, `x-stainless-runtime`, `x-stainless-runtime-version`, `x-stainless-timeout`, `x-stainless-retry-count`, `x-stainless-helper-method`) перед пересылкой.

### ⚠️ Риск: `ANTIGRAVITY_CREDITS=always` (горячая точка банов аккаунтов)

`ANTIGRAVITY_CREDITS=always` (потребляется `open-sse/executors/antigravity.ts`) направляет **каждый** запрос через Antigravity AI Credit Overages (платные кредиты Google) вместо того, чтобы позволить бесплатной квоте Google ограничивать поток. Это задокументировано как функция, но это **самый частый отчёт о нарушении ToS, который мы видим** — несколько аккаунтов Google Ultra были забанены с `403 / "service disabled for ToS violation" / insufficient_quota` после работы нескольких часов с `=always`.

Принуждение происходит на **стороне Google**, и OmniRoute не может его предотвратить. Название переменной и существующая документация создают впечатление безопасного рычага; это не так.

**Почему это сильнее привлекает детектирование злоупотреблений, чем использование только бесплатного тарифа:**

- Устойчивые автоматизированные траты на одном аккаунте Google помечаются иначе, чем «упёрся в квоту бесплатного тарифа и остановился».
- У перерасхода кредитов нет потолка частоты, поэтому неправильно настроенный клиент может сжечь несколько сотен долларов за минуты и выглядеть как перепродажа API-ключей или бот-трафик.
- Несколько пользователей OmniRoute, кратковременно расходующих кредиты параллельно с одного внешнего IP, усиливают сигнал.

**Рекомендуемая позиция:**

1. **По умолчанию `ANTIGRAVITY_CREDITS=retry`** — перерасходы используются только когда бесплатный тариф возвращает 429, а не на каждом запросе. Это более безопасный из двух ненулевых режимов.
2. **Распределяйте нагрузку между провайдерами через Auto-Combo** (`model: "auto"` или комбо `kr/glm/etc`) вместо насыщения одного аккаунта Antigravity.
3. **Установите ограничения RPM на подключение** на странице редактирования провайдера Antigravity (Dashboard → Providers → Antigravity → connection → rate limit). 30–60 RPM — защищаемая верхняя граница для устойчивого использования.
4. **Используйте разные апстрим-IP** для каждого аккаунта Antigravity, когда возможно (резидентные прокси, направленные на один аккаунт от многих пользователей, усиливают сигнал злоупотребления).
5. **Если забанили**: подавайте апелляцию через `support.google.com` → «Restore Workspace/Account access» с точным телом ответа `quota_exceeded` / `service disabled`, которое прислал Google. Восстановление не гарантируется.

Это предупреждение также отображается в панели рядом с экраном редактирования провайдера Antigravity, когда `ANTIGRAVITY_CREDITS` установлен в `always` (или появится в v3.8.0; отслеживается отдельно).

Точки касания:

- `open-sse/executors/antigravity.ts` — читает `process.env.ANTIGRAVITY_CREDITS`
- `src/lib/oauth/providers/antigravity.ts` — проводка учётных данных
- Первоначальный отчёт об инциденте: Discussion [#1183](https://github.com/diegosouzapw/OmniRoute/discussions/1183)

---

## Реестр CLI-отпечатков — `open-sse/config/cliFingerprints.ts`

Таблица по провайдерам, закрепляющая **точный** порядок заголовков и порядок полей JSON-тела, снятый с трассировок mitmproxy официальных CLI. В настоящее время зарегистрированы: `codex`, `claude`, плюс профили, производные во время выполнения, в `providerHeaderProfiles.ts` для `antigravity`, `qwen`, `github`.

```ts
interface CliFingerprint {
  headerOrder: string[]; // с учётом регистра
  bodyFieldOrder: string[]; // ключи JSON верхнего уровня
  userAgent?: string | (() => string);
  extraHeaders?: Record<string, string>;
}
```

Переключается по провайдерам через env (см. ниже). Когда выключено, заголовки/ключи тела появляются в том порядке, в котором их дал Node/JSON — легко отпечатывается.

---

## MITM-прокси (Antigravity, Linux/macOS/Windows)

Для CLI, чьи бинарники не могут быть перенаправлены через `OPENAI_BASE_URL`, OmniRoute запускает локальный прокси с завершением TLS. Эндпоинты находятся под `src/app/api/cli-tools/antigravity-mitm/`.

| Метод  | Эндпоинт                                | Назначение                                       |
| ------ | --------------------------------------- | ------------------------------------------------ |
| GET    | `/api/cli-tools/antigravity-mitm`       | Статус — running, pid, dnsConfigured, certExists |
| POST   | `/api/cli-tools/antigravity-mitm`       | Запустить MITM (требует `apiKey` + `sudoPassword`) |
| DELETE | `/api/cli-tools/antigravity-mitm`       | Остановить MITM                                  |
| GET    | `/api/cli-tools/antigravity-mitm/alias` | Список псевдонимов моделей                       |
| PUT    | `/api/cli-tools/antigravity-mitm/alias` | Сохранить псевдонимы моделей для инструмента     |

Целевой перехватываемый хост: **`daily-cloudcode-pa.googleapis.com`** (апстрим Antigravity).

### Последовательность запуска (`src/mitm/manager.ts::startMitm`)

1. Сгенерировать самоподписанный сертификат через `selfsigned` (RSA-2048, SHA-256, 1 год) — `cert/generate.ts`
2. Установить сертификат в системное хранилище доверия — `cert/install.ts`
3. Добавить запись hosts `127.0.0.1 daily-cloudcode-pa.googleapis.com` — `dns/dnsConfig.ts`
4. Запустить `src/mitm/server.cjs` с `ROUTER_API_KEY` + `MITM_LOCAL_PORT` (по умолчанию `443`)
5. Сохранить PID в `<DATA_DIR>/mitm/.mitm.pid`

### Динамическое определение хранилища доверия в Linux — `cert/install.ts`

`getLinuxCertConfig()` проходит приоритетный список и выбирает первый существующий каталог:

| Семейство дистрибутивов    | Каталог                                     | Команда обновления       |
| -------------------------- | ------------------------------------------- | ------------------------ |
| Debian / Ubuntu            | `/usr/local/share/ca-certificates`          | `update-ca-certificates` |
| Arch / CachyOS / Manjaro   | `/etc/ca-certificates/trust-source/anchors` | `update-ca-trust`        |
| Fedora / RHEL / CentOS     | `/etc/pki/ca-trust/source/anchors`          | `update-ca-trust`        |
| openSUSE                   | `/etc/pki/trust/anchors`                    | `update-ca-certificates` |

Имя файла сертификата: `omniroute-mitm.crt`. Совпадение отпечатка через `getCertFingerprint()` (SHA-1 от DER).

Дополнительно `updateNssDatabases()` устанавливает в пользовательские базы NSS, когда доступен `certutil`: `~/.pki/nssdb`, `~/snap/chromium/.../nssdb`, все профили Firefox (включая snap), под псевдонимом **`OmniRoute MITM Root CA`**.

### macOS / Windows

- **macOS:** `security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain`
- **Windows:** привилегированный PowerShell → `certutil -addstore Root`

### Аутентификация

Все эндпоинты MITM требуют управляющей аутентификации (`requireCliToolsAuth`). Пароль sudo кэшируется в области модуля (никогда в `globalThis`) и очищается при `stopMitm()`.

---

## Переопределения User-Agent — переменные окружения (`.env.example` раздел 12)

| Переменная               | По умолчанию                                                    |
| ------------------------ | --------------------------------------------------------------- |
| `CLAUDE_USER_AGENT`      | `claude-cli/2.1.207 (external, cli)`                            |
| `CODEX_USER_AGENT`       | `codex-cli/0.142.0 (Windows 10.0.26200; x64)`                   |
| `GITHUB_USER_AGENT`      | `GitHubCopilotChat/0.54.0`                                      |
| `ANTIGRAVITY_USER_AGENT` | `antigravity/2.0.1 linux/arm64 google-api-nodejs-client/10.3.0` |
| `KIRO_USER_AGENT`        | `AWS-SDK-JS/3.0.0 kiro-ide/1.0.0`                               |
| `QODER_USER_AGENT`       | `Qoder-Cli`                                                     |
| `QWEN_USER_AGENT`        | `QwenCode/0.19.3 (linux; x64)`                                  |
| `CURSOR_USER_AGENT`      | `Cursor/3.4`                                                    |

Потребляются `open-sse/executors/base.ts::buildHeaders()` через динамический поиск. **Обновляйте их, когда провайдеры выпускают новые версии CLI** — устаревшие строки UA начинают отклоняться как устаревшие клиенты.

## Переключатели режима совместимости CLI (`.env.example` раздел 13)

| Переменная                 | Эффект                          |
| -------------------------- | ------------------------------- |
| `CLI_COMPAT_CODEX=1`       | Отпечаток Codex                 |
| `CLI_COMPAT_CLAUDE=1`      | Отпечаток claude-cli            |
| `CLI_COMPAT_GITHUB=1`      | Отпечаток GitHub Copilot Chat   |
| `CLI_COMPAT_ANTIGRAVITY=1` | Отпечаток Antigravity           |
| `CLI_COMPAT_KIRO=1`        | Kiro                            |
| `CLI_COMPAT_CURSOR=1`      | Cursor                          |
| `CLI_COMPAT_KIMI_CODING=1` | Kimi Coding                     |
| `CLI_COMPAT_KILOCODE=1`    | KiloCode                        |
| `CLI_COMPAT_CLINE=1`       | Cline                           |
| `CLI_COMPAT_QWEN=1`        | Qwen Code                       |
| `CLI_COMPAT_ALL=1`         | Включить все вышеперечисленные  |

IP провайдера **всегда сохраняется** — переключатель только переформировывает сетевой образ запроса, он не переключает выход по IP.

---

## Санитизация входящих заголовков

OmniRoute очищает входящие заголовки клиентов перед пересылкой, чтобы запрос, пришедший от Cursor, не утёк `User-Agent: Cursor/X.Y.Z` в апстрим Claude. См. `src/shared/constants/upstreamHeaders.ts` для списка запрещённых, синхронизированного со схемами Zod и модульными тестами.

---

## Обновление отпечатков при ротации провайдера

1. Захватите трафик официального CLI с `mitmproxy` (TLS-перехват + дамп)
2. Извлеките JA3/JA4 и буквальный порядок заголовков
3. Обновите соответствующую запись `CLI_FINGERPRINTS[...]`
4. Поднимите соответствующее значение по умолчанию `*_USER_AGENT` в `.env.example`
5. Если изменилось само TLS-рукопожатие: обновите `chatgptTlsClient.ts::CHATGPT_PROFILE` или опцию `browser:` у wreq-js
6. Запустите `chatgptTlsClient.test.ts` и ручной канареечный тест против живого провайдера
7. Выпустите в патч-релизе; задокументируйте в `CHANGELOG.md`

---

## Тесты

- `open-sse/services/__tests__/chatgptTlsClient.test.ts` — приоритет разрешения прокси, обработка abort, восстановление от зависаний
- `tests/unit/anthropic-cache-fingerprint.test.ts` — детерминизм отпечатка
- `tests/unit/chatgpt-web.test.ts` — сквозной путь скрытности для ChatGPT

---

## См. также

- [RESILIENCE_GUIDE.md](../architecture/RESILIENCE_GUIDE.md) — что происходит, когда путь скрытности получает `403`
- [TROUBLESHOOTING.md](../guides/TROUBLESHOOTING.md)
- [ENVIRONMENT.md](../reference/ENVIRONMENT.md) — полный справочник переменных окружения
- [CLI-TOOLS.md](../reference/CLI-TOOLS.md) — взгляд оператора на рабочий процесс MITM
