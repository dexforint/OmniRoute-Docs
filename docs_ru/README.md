---
title: "Документация OmniRoute"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Документация OmniRoute

Навигационный указатель комплекта документации OmniRoute. Темы сгруппированы по назначению, чтобы вы могли быстро найти нужное.

> Ищете обзор проекта, инструкции по установке или заметки о релизах? См. корневые [README.md](../README.md), [CHANGELOG.md](../CHANGELOG.md) и [CONTRIBUTING.md](../CONTRIBUTING.md).

---

## Для нетехнических пользователей

Простые руководства по использованию OmniRoute — техническая подготовка не требуется.

### getting-started/

- [QUICK-START.md](getting-started/QUICK-START.md) — установка и запуск OmniRoute за 3 минуты.
- [AUTO-COMBO-GUIDE.md](getting-started/AUTO-COMBO-GUIDE.md) — позвольте OmniRoute выбирать лучший ИИ за вас.
- [PROVIDERS-GUIDE.md](getting-started/PROVIDERS-GUIDE.md) — как подключить ИИ-провайдеров.
- [FREE-TIERS-GUIDE.md](getting-started/FREE-TIERS-GUIDE.md) — бесплатный ИИ без банковской карты.
- [TROUBLESHOOTING.md](getting-started/TROUBLESHOOTING.md) — решение типичных проблем.

### guides/

- [SETUP_GUIDE.md](guides/SETUP_GUIDE.md) — первичная настройка OmniRoute.
- [USER_GUIDE.md](guides/USER_GUIDE.md) — повседневная работа с панелью управления и API.
- [FEATURES.md](guides/FEATURES.md) — обзор возможностей панели управления.
- [TIERS.md](guides/TIERS.md) — уровни (tiers) OmniRoute простыми словами (руководство пользователя).
- [USAGE_QUOTA_GUIDE.md](guides/USAGE_QUOTA_GUIDE.md) — отслеживание использования, квот и расходов.
- [COST_TRACKING.md](guides/COST_TRACKING.md) — учёт стоимости и расходов.
- [FREE_PROVIDER_RANKINGS.md](guides/FREE_PROVIDER_RANKINGS.md) — рейтинги бесплатных провайдеров (Arena ELO).
- [DOCKER_GUIDE.md](guides/DOCKER_GUIDE.md) — запуск OmniRoute в Docker.
- [ELECTRON_GUIDE.md](guides/ELECTRON_GUIDE.md) — десктопные сборки (Electron).
- [TERMUX_GUIDE.md](guides/TERMUX_GUIDE.md) — запуск на Android через Termux.
- [PWA_GUIDE.md](guides/PWA_GUIDE.md) — установка панели управления как PWA.
- [REMOTE-MODE.md](guides/REMOTE-MODE.md) — удалённый доступ к OmniRoute + токены с ограниченной областью действия.
- [CLI-INTEGRATIONS.md](guides/CLI-INTEGRATIONS.md) — сводная таблица CLI-интеграций `setup-*`.
- [CLAUDE-CODE-CONFIGURATION.md](guides/CLAUDE-CODE-CONFIGURATION.md) — Claude Code CLI с OmniRoute.
- [CODEX-CLI-CONFIGURATION.md](guides/CODEX-CLI-CONFIGURATION.md) — Codex CLI с OmniRoute.
- [KIRO_SETUP.md](guides/KIRO_SETUP.md) — настройка Kiro.
- [I18N.md](guides/I18N.md) — процесс перевода и работы с локалями.
- [TROUBLESHOOTING.md](guides/TROUBLESHOOTING.md) — подробный справочник по устранению неполадок.
- [UNINSTALL.md](guides/UNINSTALL.md) — полное удаление.

---

## Для технических пользователей

Техническая документация для разработчиков и контрибьюторов.

## architecture/

Как устроена система — читайте, чтобы понять среду выполнения, структуру кода и модель отказоустойчивости.

- [ARCHITECTURE.md](architecture/ARCHITECTURE.md) — общая архитектура системы (конвейер запросов, слои, модули).
- [CODEBASE_DOCUMENTATION.md](architecture/CODEBASE_DOCUMENTATION.md) — инженерный справочник по кодовой базе.
- [REPOSITORY_MAP.md](architecture/REPOSITORY_MAP.md) — навигaция по каталогам репозитория.
- [AUTHZ_GUIDE.md](architecture/AUTHZ_GUIDE.md) — конвейер авторизации (классификатор маршрутов + движок политик).
- [RESILIENCE_GUIDE.md](architecture/RESILIENCE_GUIDE.md) — circuit breaker для провайдеров, охлаждение соединений и блокировка моделей.
- [QUALITY_GATES.md](architecture/QUALITY_GATES.md) — перечень скриптов и CI-заданий контроля качества.
- [MONITORING_SECTIONS.md](architecture/MONITORING_SECTIONS.md) — навигация по панели мониторинга и расходов.
- [cluster-decisions.md](architecture/cluster-decisions.md) — решения по опциональному профилю sidecar/кластера.

## reference/

Справочные материалы — поверхность API, переменные окружения, флаги CLI, каталог провайдеров.

- [API_REFERENCE.md](reference/API_REFERENCE.md) — эндпоинты REST API и их форматы.
- [PROVIDER_REFERENCE.md](reference/PROVIDER_REFERENCE.md) — автоматически сгенерированный каталог провайдеров (не редактировать вручную).
- [PROVIDER_PLUGIN_MANIFEST.md](reference/PROVIDER_PLUGIN_MANIFEST.md) — sidecar-совместимый контракт плагина провайдера для миграции Bifrost и CLIProxyAPI.
- [openapi.yaml](openapi.yaml) — спецификация OpenAPI для публичного API.
- [ENVIRONMENT.md](reference/ENVIRONMENT.md) — справочник переменных окружения.
- [FEATURE_FLAGS.md](reference/FEATURE_FLAGS.md) — флаги функций и их значения по умолчанию.
- [CLI-TOOLS.md](reference/CLI-TOOLS.md) — встроенные CLI-команды.
- [FREE_TIERS.md](reference/FREE_TIERS.md) — каталог LLM-провайдеров с бесплатным тарифом.

## frameworks/

Подключаемые подсистемы, доступные клиентам, агентам и операторам.

- [MCP-SERVER.md](frameworks/MCP-SERVER.md) — сервер Model Context Protocol.
- [A2A-SERVER.md](frameworks/A2A-SERVER.md) — JSON-RPC сервер Agent-to-Agent (A2A).
- [ACP.md](frameworks/ACP.md) — Agent Client Protocol.
- [AGENT_PROTOCOLS_GUIDE.md](frameworks/AGENT_PROTOCOLS_GUIDE.md) — обзор A2A / ACP / облачного агента.
- [AGENTBRIDGE.md](frameworks/AGENTBRIDGE.md) — мост агента для IDE.
- [AGENT-SKILLS.md](frameworks/AGENT-SKILLS.md) — каталог навыков агента.
- [CLOUD_AGENT.md](frameworks/CLOUD_AGENT.md) — среда выполнения облачного агента и провайдеры.
- [SKILLS.md](frameworks/SKILLS.md) — фреймворк Skills (изолированные расширения).
- [MEMORY.md](frameworks/MEMORY.md) — постоянная память (FTS5 + Qdrant).
- [WEBHOOKS.md](frameworks/WEBHOOKS.md) — события вебхуков и их доставка.
- [EVALS.md](frameworks/EVALS.md) — наборы оценочных тестов (evals).
- [GAMIFICATION.md](frameworks/GAMIFICATION.md) — система геймификации и таблица лидеров.
- [EMBEDDED-SERVICES.md](frameworks/EMBEDDED-SERVICES.md) — встроенные sidecar-сервисы (9Router, CLIProxyAPI).
- [NOTION_CONTEXT.md](frameworks/NOTION_CONTEXT.md) — источник контекста Notion.
- [OBSIDIAN_CONTEXT.md](frameworks/OBSIDIAN_CONTEXT.md) — источник контекста Obsidian.
- [OPENCODE.md](frameworks/OPENCODE.md) — интеграция OpenCode.
- [OPEN_SSE_ARCHITECTURE.md](frameworks/OPEN_SSE_ARCHITECTURE.md) — внутреннее устройство движка потоковой передачи open-sse.
- [PLAYGROUND_STUDIO.md](frameworks/PLAYGROUND_STUDIO.md) — интерфейс Playground Studio.
- [SEARCH_TOOLS_STUDIO.md](frameworks/SEARCH_TOOLS_STUDIO.md) — интерфейс Search Tools Studio.
- [TRAFFIC_INSPECTOR.md](frameworks/TRAFFIC_INSPECTOR.md) — инспектор трафика (MITM).
- [PLUGINS.md](frameworks/PLUGINS.md) — обзор системы CLI-плагинов.
- [PLUGIN_SDK.md](frameworks/PLUGIN_SDK.md) — справочник по SDK плагинов.
- [PLUGIN_MARKETPLACE.md](frameworks/PLUGIN_MARKETPLACE.md) — маркетплейс плагинов.

## routing/

Маршрутизация комбо, скоринг и replay.

- [AUTO-COMBO.md](routing/AUTO-COMBO.md) — Auto-Combo (многофакторный скоринг, 17 стратегий).
- [QUOTA_SHARE.md](routing/QUOTA_SHARE.md) — движок распределения квот.
- [REASONING_REPLAY.md](routing/REASONING_REPLAY.md) — кэш воспроизведения рассуждений.

## security/

Защитные механизмы, соответствие требованиям, скрытность и обязательные паттерны работы с публичными учётными данными и сообщениями об ошибках.

- [GUARDRAILS.md](security/GUARDRAILS.md) — защита PII, от инъекций в промпты, визуальные ограничения.
- [COMPLIANCE.md](security/COMPLIANCE.md) — аудит и соответствие требованиям.
- [STEALTH_GUIDE.md](security/STEALTH_GUIDE.md) — скрытность TLS / отпечатков.
- [PUBLIC_CREDS.md](security/PUBLIC_CREDS.md) — **обязательный** паттерн встраивания публичных OAuth client_id/secret апстримов и веб-ключей Firebase без срабатывания сканеров секретов.
- [ERROR_SANITIZATION.md](security/ERROR_SANITIZATION.md) — **обязательный** паттерн пропускания каждого ответа об ошибке через `sanitizeErrorMessage` для предотвращения утечки стектрейсов.
- [ROUTE_GUARD_TIERS.md](security/ROUTE_GUARD_TIERS.md) — уровни классификации защиты маршрутов.
- [CLI_TOKEN.md](security/CLI_TOKEN.md) — CLI-токен по machine-ID (HMAC + устаревший SHA-256).
- [EGRESS_POLICY.md](security/EGRESS_POLICY.md) — политика исходящих IP-семейств (IPv4/IPv6).
- [MITM-TPROXY-DECRYPT.md](security/MITM-TPROXY-DECRYPT.md) — прозрачная MITM-расшифровка.
- [SUPPLY_CHAIN.md](security/SUPPLY_CHAIN.md) — контроль цепочки поставки (SLSA, SBOM, Trivy, osv-scanner, Scorecard).
- [SOCKET_DEV_FINDINGS.md](security/SOCKET_DEV_FINDINGS.md) — аттестации находок цепочки поставки.

## compression/

Движки сжатия промптов, правила и языковые пакеты.

- [COMPRESSION_GUIDE.md](compression/COMPRESSION_GUIDE.md) — общий обзор сжатия.
- [COMPRESSION_ENGINES.md](compression/COMPRESSION_ENGINES.md) — доступные движки сжатия.
- [COMPRESSION_RULES_FORMAT.md](compression/COMPRESSION_RULES_FORMAT.md) — формат файлов правил.
- [COMPRESSION_LANGUAGE_PACKS.md](compression/COMPRESSION_LANGUAGE_PACKS.md) — языковые пакеты.
- [RTK_COMPRESSION.md](compression/RTK_COMPRESSION.md) — подробный разбор движка RTK.
- [CONTEXT_EDITING.md](compression/CONTEXT_EDITING.md) — делегированное редактирование контекста (Anthropic).
- [EXTENDING_COMPRESSION.md](compression/EXTENDING_COMPRESSION.md) — добавление собственного движка сжатия.

## providers/

Руководства по интеграции конкретных провайдеров.

- [CLAUDE_WEB.md](providers/CLAUDE_WEB.md) — провайдер Claude Web (аутентификация по cookie).
- [AGENTROUTER.md](providers/AGENTROUTER.md) — настройка AgentRouter.
- [ZED-DOCKER.md](providers/ZED-DOCKER.md) — интеграция Zed IDE в Docker.

## comparison/

- [OMNIROUTE_VS_ALTERNATIVES.md](comparison/OMNIROUTE_VS_ALTERNATIVES.md) — сравнение OmniRoute с альтернативами.

## ops/

Релизы, развёртывание, прокси, туннели, покрытие, база данных, мониторинг.

- [RELEASE_CHECKLIST.md](ops/RELEASE_CHECKLIST.md) — чек-лист релизного процесса.
- [RELEASE_GREEN.md](ops/RELEASE_GREEN.md) — поддержание очереди PR и релизной ветки «зелёными».
- [QUALITY_GATE_PLAYBOOK.md](ops/QUALITY_GATE_PLAYBOOK.md) — playbook контроля качества.
- [BRANCH_PROTECTION_MAIN.md](ops/BRANCH_PROTECTION_MAIN.md) — защита ветки `main`.
- [COVERAGE_PLAN.md](ops/COVERAGE_PLAN.md) — план тестового покрытия.
- [DATABASE_GUIDE.md](ops/DATABASE_GUIDE.md) — схема БД и операции.
- [SQLITE_RUNTIME.md](ops/SQLITE_RUNTIME.md) — цепочка разрешения драйвера SQLite.
- [MONITORING_GUIDE.md](ops/MONITORING_GUIDE.md) — мониторинг и наблюдаемость.
- [FLY_IO_DEPLOYMENT_GUIDE.md](ops/FLY_IO_DEPLOYMENT_GUIDE.md) — развёртывание на Fly.io.
- [VM_DEPLOYMENT_GUIDE.md](ops/VM_DEPLOYMENT_GUIDE.md) — развёртывание на универсальной ВМ.
- [PROXY_GUIDE.md](ops/PROXY_GUIDE.md) — настройка исходящего прокси.
- [TUNNELS_GUIDE.md](ops/TUNNELS_GUIDE.md) — Cloudflare Tunnel и аналоги.

## diagrams/

Исходники Mermaid и экспортированные диаграммы SVG/PNG, на которые ссылаются документы выше. См. [diagrams/README.md](diagrams/README.md).

## i18n/

Переведённые зеркала документации на 43 локали. Список поддерживаемых языков см. в [i18n/README.md](i18n/README.md).

## screenshots/

Статические скриншоты, используемые панелью управления и README. Не являются частью основного текста документации.

---

## Автогенерируемые артефакты

- [reference/PROVIDER_REFERENCE.md](reference/PROVIDER_REFERENCE.md) генерируется скриптом `scripts/docs/gen-provider-reference.ts` из `src/shared/constants/providers.ts`. Не редактировать вручную.
- Интерфейс `/docs` работает на генерации исходников Fumadocs MDX из перечисленных выше подпапок.
