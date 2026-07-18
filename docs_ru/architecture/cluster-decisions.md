---
title: "Кластерные решения"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Кластерные решения — опциональные профили сайдкаров

**Статус:** предложение (ожидает ревью @diegosouzapw)
**Дата:** 2026-06-20
**Ссылки:** [#3932](https://github.com/diegosouzapw/OmniRoute/issues/3932), PR #4381

## TL;DR

Два опциональных compose-профиля (`memory`, `bifrost`) для существующего деплоя из 8 сервисов в [`docker-compose.yml`](../../docker-compose.yml). Поведение по умолчанию **не изменено**: 3 × реплики `omniroute` + Caddy + Redis + CliproxyAPI. Два новых профиля добавляют Qdrant и Bifrost как опциональные сайдкары, включаемые через `docker compose --profile <name> up`. **Ни один существующий сервис не удаляется и не заменяется.**

## Почему это консервативно

Существующая форма деплоя OmniRoute уже компактна и проверена:

- **`redis:7-alpine`** справляется с нагрузкой rate-limit/кэша в проде.
- **SQLite + sqlite-vec + FTS5** покрывают локальную память + вектор + полнотекстовый поиск (см. [`src/lib/memory/vectorStore.ts:108`](../../src/lib/memory/vectorStore.ts)).
- **Caddy** уже является LB + TLS-терминатором ([`docker-compose.yml`](../../docker-compose.yml)).
- **Bifrost** уже интегрирован как роутер Tier-1 в [`src/app/api/v1/relay/chat/completions/bifrost/route.ts`](../../src/app/api/v1/relay/chat/completions/bifrost/route.ts) (сайдкар-прокси с kill switch через env `BIFROST_ENABLED` — установите `=0`, чтобы обойти сайдкар и провалиться на TS-путь).

Два профиля здесь — это **варианты масштабирования для деплоев, упирающихся в потолок SQLite** — не миграции. Оба по умолчанию выключены.

## Два профиля

### `memory` — сайдкар векторной памяти Qdrant

**Когда включать:**

- > 1M эмбеддингов на деплой (sqlite-vec начинает тормозить в масштабе).
- Мульти-реплика деплой, которому нужно общее векторное состояние между `omniroute-1/2/3`.
- У вас уже есть внешний кластер Qdrant (Qdrant Cloud, on-prem).

**Что добавляет:**

| Сервис | Образ                     | Порты       | Примечания                                                  |
| ------ | ------------------------- | ----------- | ----------------------------------------------------------- |
| `qdrant` | `qdrant/qdrant:v1.12.4` | `6333` HTTP | HNSW индекс; постоянный том `omniroute_qdrant_data` |

**Активация:** переключите `qdrantEnabled = true` в UI настроек **или** установите `QDRANT_HOST=qdrant` env. См. [`src/lib/memory/qdrant.ts:60`](../../src/lib/memory/qdrant.ts) для правил приоритета (таблица настроек → env var → default).

**Переменные окружения:** `QDRANT_HOST`, `QDRANT_PORT`, `QDRANT_API_KEY`, `QDRANT_COLLECTION`, `QDRANT_VECTOR_SIZE`, `QDRANT_HNSW_EF_CONSTRUCT` (см. `.env.example` строки 1672-1683).

### `bifrost` — сайдкар роутера Bifrost Tier-1

**Когда включать:**

- Вы запускаете ≥3 реплики `omniroute` и хотите централизовать ротацию провайдеров в одном Go-процессе.
- Вы хотите единую поверхность аудита/логирования для запросов к апстрим-провайдерам по всем репликам.
- Вы хотите горизонтальное масштабирование уровня маршрутизации Tier-1 независимо от реплик OmniRoute.

**Что добавляет:**

| Сервис  | Образ                              | Порты  | Примечания                                                                    |
| ------- | ---------------------------------- | ------ | ----------------------------------------------------------------------------- |
| `bifrost` | `ghcr.io/maximhq/bifrost:1.5.21` | `8080` | Go-роутер Tier-1; том логов `omniroute_bifrost_logs` |

**Активация:** установите `BIFROST_BASE_URL=http://bifrost:8080` в `.env.example`. Существующий маршрут сайдкар-прокси в [`src/app/api/v1/relay/chat/completions/bifrost/route.ts`](../../src/app/api/v1/relay/chat/completions/bifrost/route.ts) (добавлен в PR #4381) подхватит это автоматически.

**Env vars:** `BIFROST_BASE_URL`, `BIFROST_API_KEY`, `BIFROST_STREAMING_ENABLED`, `BIFROST_TIMEOUT_MS` (см. `.env.example` строки 1685-1695).

## Что этот PR явно НЕ делает

Исходная ветка issue предлагала более крупную переработку кластера. После аудита реальной формы нагрузки следующее **отклонено** по указанным причинам:

| Компонент                              | Вердикт   | Причина                                                                                                 |
| -------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------- |
| **Dragonfly**                          | **DROP**  | `redis:7-alpine` уже годится для нагрузки rate-limit в проде; потолка нет.                               |
| **NATS**                               | **DROP**  | Каждая реплика `omniroute` — один Node.js процесс; нет многопроцессной pub/sub нагрузки.                 |
| **PostgreSQL**                         | **DROP**  | SQLite + sqlite-vec + FTS5 покрывают все 3 кейса; 97 миграций + упаковка Electron блокируют миграцию.    |
| **Neo4j**                              | **DROP**  | Маршрутизация — это join 5 таблиц; рекурсивный CTE на SQLite достаточен.                                 |
| **MinIO**                              | **DROP**  | Нет нагрузки blob-ов в несколько МБ; изображения/аудио — прокси passthrough.                              |
| **pgvector / pg_ai / pg_textsearch**   | **DROP**  | Та же причина потолка SQLite, что и PostgreSQL; экосистема pgvector фрагментирована.                     |
| **HAProxy / Envoy**                    | **DROP**  | Caddy уже делает LB + TLS; оба были явно отклонены как Tier-1 роутеры (см. `AGENTS.md`).                 |

Если будущий кейс докажет один из этих компонентов, это документ — место для поправки.

## 4-недельный rollout (если одобрено)

1. **Нед 1** — Приземлить этот PR + верификация опциональных профилей со стеком compose из 3 реплик.
2. **Нед 2** — Полная активация Bifrost для OpenAI/Claude/Gemini/Ollama (4 из 14+ провайдеров) с использованием маршрута sidecar proxy в [`src/app/api/v1/relay/chat/completions/bifrost/route.ts`](../../src/app/api/v1/relay/chat/completions/bifrost/route.ts) (под гейтом `BIFROST_ENABLED`, kill-switchable в рантайме).
3. **Нед 3** — Профиль памяти Qdrant включён в одном тестовом деплое; измерить дельту задержки vs sqlite-vec.
4. **Нед 4** — Healthcheck наблюдаемости (`docker compose ps` коды выхода + `wget` smoke-тесты); 71-pillar refresh по ADR-041.

## Файлы, изменённые в этом PR

| Файл                                                     | Изменение                                                                                                                                                                                                      |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docker-compose.yml`                                     | +30 строк: профиль `memory` (Qdrant), профиль `bifrost` (Bifrost), постоянные тома, healthcheck-и.                                                                                                               |
| `.env.example`                                           | +24 строки: `QDRANT_*` (6 vars), `BIFROST_*` (4 vars).                                                                                                                                                          |
| `docs/reference/ENVIRONMENT.md`                          | +6 строк в разделе 25 для env vars `QDRANT_*`.                                                                                                                                                                 |
| `src/lib/memory/qdrant.ts`                               | +33 строки: цепочка fallback env-var (settings → env → default) для `QDRANT_HOST`/`QDRANT_PORT`/`QDRANT_API_KEY`/`QDRANT_COLLECTION`/`QDRANT_VECTOR_SIZE`/`QDRANT_HNSW_EF_CONSTRUCT`/`QDRANT_EMBEDDING_MODEL`. |
| `src/lib/memory/__tests__/qdrant-wiring.test.ts`         | +88 строк: 9 новых тест-кейсов, закрепляющих приоритет fallback env-var.                                                                                                                                       |
| `docs/architecture/cluster-decisions.md` (этот файл)     | НОВЫЙ — запись решения для опциональных профилей.                                                                                                                                                              |
| `AGENTS.md`                                              | +1 строка: указатель на этот док в таблице справочной документации.                                                                                                                                            |

**Чистый затронутый код:** 4 продакшн файла (`docker-compose.yml`, `qdrant.ts`, `.env.example`, `ENVIRONMENT.md`), 1 тестовый файл (`qdrant-wiring.test.ts`), 2 док файла (`cluster-decisions.md`, `AGENTS.md`).
