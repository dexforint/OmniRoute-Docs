---
title: "Политика семейства исходящих IP (IPv4/IPv6)"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Политика семейства исходящих IP (IPv4/IPv6)

> **Закрепите исходящий трафик за одним семейством IP — `auto`, `ipv4` или `ipv6` — для каждого прокси, чтобы IPv6-only исход никогда молча не утекал обратно в IPv4.**

> **Источник истины:** `open-sse/utils/proxyFamily.ts`, `open-sse/utils/proxyDispatcher.ts`, `open-sse/utils/proxyFetch.ts`, `open-sse/utils/socksConnectorWithFamily.ts`, `open-sse/utils/proxyFamilyResolve.ts`, `src/shared/validation/schemas.ts`, `src/lib/db/proxies.ts`, `src/lib/db/upstreamProxy.ts`, `src/lib/db/migrations/099_proxy_family.sql`

OmniRoute позволяет каждому прокси нести **директиву семейства адресов для исходящего трафика**. По умолчанию ОС выбирает IPv4 или IPv6 (dual-stack, «Happy Eyeballs»). Когда вы устанавливаете директиву в `ipv4` или `ipv6`, OmniRoute закрепляет каждое соединение через этот прокси за выбранным семейством и **отказывается (fail-closed)** вместо отката к другому семейству.

Эта страница документирует, что такое директива, зачем она существует, где её настроить и как рантайм её резолвит.

---

## Содержание

- [Что это такое](#what-it-is)
- [Зачем это существует](#why-it-exists)
- [Три значения](#the-three-values)
- [Как настроить](#how-to-configure-it)
- [Как резолвится `auto`](#how-auto-resolves)
- [Как применяются `ipv4` / `ipv6`](#how-ipv4--ipv6-are-enforced)
- [Совместимость с SOCKS5](#socks5-compatibility)
- [Поведение fail-closed](#fail-closed-behavior)
- [Модель данных](#data-model)
- [Связанная документация](#related-documentation)

---

## Что это такое

У каждого прокси в реестре есть поле `family` с тремя возможными значениями, валидируемое перечислением Zod:

```ts
// src/shared/validation/schemas.ts
family: z.enum(["auto", "ipv4", "ipv6"]).optional().default("auto"),
```

Поле по умолчанию равно `"auto"`, что сохраняет прежнее dual-stack поведение. Установка в `ipv4` или `ipv6` закрепляет семейство соединений для этого прокси.

Директива нормализуется везде через один помощник, так что любое неизвестное значение сводится к `auto`:

```ts
// open-sse/utils/proxyFamily.ts
export type ProxyFamily = "auto" | "ipv4" | "ipv6";

export function parseProxyFamily(value: unknown): ProxyFamily {
  return value === "ipv4" || value === "ipv6" ? value : "auto";
}
```

---

## Зачем это существует

Введено в PR [#3777](https://github.com/diegosouzapw/OmniRoute/pull/3777). Мотивирующие проблемы:

| Проблема                                        | Что исправляет директива                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Утечка IPv6-only трафика в IPv4**             | Когда у хоста прокси есть записи и A, и AAAA (или ОС предпочитает IPv4), Happy Eyeballs может выйти по IPv4, даже если вы намеревались использовать путь только IPv6. Закрепление `ipv6` устраняет эту утечку.                                                                                                                                                                                       |
| **Отзыв при аномалии общего egress**            | Ротирующие провайдеры (codex/openai) отзывают токены, когда много аккаунтов выходит через **один и тот же** IP на высоком объёме. Управление семейством egress — часть удержания аккаунтов на разных, предсказуемых путях выхода (см. [`src/lib/proxyEgress.ts`](../../src/lib/proxyEgress.ts) для парной диагностики egress-IP).                                                                        |
| **Детерминированный egress для комплаенса/тестов** | Когда нужно гарантировать, что трафик выходит по конкретному семейству, `auto` недостаточно.                                                                                                                                                                                                                                                                                                        |

Директива намеренно **на прокси**, а не глобальная — разные прокси в вашем пуле могут иметь разные политики.

---

## Три значения

| Значение | Метка в UI          | Поведение                                                                                                                                                 |
| -------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `auto`   | `Auto (dual-stack)` | ОС выбирает семейство. Для прокси-хоста с IP-литералом семейство присуще литералу; для имени хоста допустимы оба семейства. Это поведение по умолчанию.    |
| `ipv4`   | `IPv4 only`         | Закрепляет соединение за IPv4. Отказывает (fail-closed), если у хоста прокси нет записи IPv4 (A).                                                          |
| `ipv6`   | `IPv6 only`         | Закрепляет соединение за IPv6. Отказывает (fail-closed), если у хоста прокси нет записи IPv6 (AAAA).                                                       |

Строки UI находятся в `src/i18n/messages/en.json` (`labelFamily`, `familyAuto`, `familyIpv4`, `familyIpv6`, `familyHint`).

---

## Как настроить

### Панель управления

Селектор находится в форме прокси на вкладке **Proxy Pool**:

1. Откройте **Dashboard → Settings → Proxy → Proxy Pool**
2. Добавьте или отредактируйте прокси
3. Установите выпадающий список **IP family** в `Auto (dual-stack)`, `IPv4 only` или `IPv6 only`
4. Сохраните

Элемент управления отрисовывается `ProxyRegistryManager.tsx` (монтируется в `proxy/ProxyPoolTab.tsx`).

### API

Поле `family` является частью полезных нагрузок создания/обновления реестра прокси, валидируется `createProxyRegistrySchema` / `updateProxyRegistrySchema` (`src/shared/validation/schemas.ts`) и обрабатывается `POST` / `PATCH /api/v1/management/proxies`:

```bash
# Создать прокси только IPv6
curl -X POST http://localhost:20128/api/v1/management/proxies \
  -H "Content-Type: application/json" \
  -d '{
    "name": "IPv6 egress",
    "type": "socks5",
    "host": "proxy.example.com",
    "port": 1080,
    "family": "ipv6"
  }'

# Изменить существующий прокси на только IPv4
curl -X PATCH http://localhost:20128/api/v1/management/proxies \
  -H "Content-Type: application/json" \
  -d '{ "id": "proxy-uuid-here", "family": "ipv4" }'
```

То же поле также принимается встроенным объектом конфигурации прокси, используемым для записей апстрим-прокси (`upstream_proxy_config.family`, см. [Модель данных](#data-model)).

Про остальную часть API CRUD/назначения прокси см. [PROXY_GUIDE.md](../ops/PROXY_GUIDE.md).

---

## Как резолвится `auto`

Когда `family` равен `auto`, OmniRoute **не** добавляет никакой директивы — URL прокси используется как есть, и семейство соединения определяется присущим образом.

Во время построения URL (`proxyConfigToUrl` / `normalizeProxyUrl` в `open-sse/utils/proxyDispatcher.ts`) прокси `auto` даёт обычный URL без маркера:

```ts
// open-sse/utils/proxyDispatcher.ts
const fam = parseProxyFamily(config.family);
const normalized = normalizeProxyUrl(proxyUrlStr, "context proxy", { allowSocks5 });
return fam === "auto" ? normalized : `${normalized}?family=${fam}`;
```

Во время диспетчеризации (`resolveDispatcherFamily`) `auto` резолвится в присущее семейство хоста с IP-литералом или в `null` (пусть ОС решает) для имени хоста:

```ts
// open-sse/utils/proxyDispatcher.ts
function resolveDispatcherFamily(parsed: URL): 4 | 6 | null {
  const directive = parseProxyFamily(parsed.searchParams.get("family") ?? undefined);
  const literal = detectIpLiteralFamily(parsed.hostname);
  if (directive === "auto") return literal; // null для имени хоста → выбирает ОС
  // ...
}
```

Итак:

- `auto` + IP-литерал хоста (`192.0.2.1` / `[2001:db8::1]`) → семейство этого литерала.
- `auto` + имя хоста → `null` → стандартное dual-stack разрешение ОС.

---

## Как применяются `ipv4` / `ipv6`

Директива не-`auto` путешествует как один синтетический query-маркер — `?family=ipv4` или `?family=ipv6`, — добавленный ровно один раз к нормализованному URL прокси. `normalizeProxyUrl` аккуратно убирает и повторно добавляет этот маркер ровно один раз, чтобы он никогда не портил разбор порта.

Когда строится диспетчер, маркер читается и преобразуется в конкретное семейство соединения. Если хост — IP-литерал **противоположного** семейства, OmniRoute выбрасывает ошибку (противоречие — fail-closed):

```ts
// open-sse/utils/proxyDispatcher.ts
const want = directive === "ipv6" ? 6 : 4;
if (literal !== null && literal !== want) {
  throw new Error(
    `[ProxyDispatcher] Proxy family directive ${directive} contradicts ${literal === 6 ? "IPv6" : "IPv4"} literal host`
  );
}
```

Конкретное семейство затем закрепляется на коннекторе:

- **HTTP/HTTPS-прокси** (`ProxyAgent`): `proxyTls: { family, autoSelectFamily: false }` — отключает Happy Eyeballs, чтобы выбранное семейство было единственным набираемым.
- **SOCKS5-прокси**: пользовательский коннектор прокидывает `socket_options: { family, autoSelectFamily: false }` в SOCKS-клиент (см. [Совместимость с SOCKS5](#socks5-compatibility)).

---

## Совместимость с SOCKS5

Закрепление семейства работает с SOCKS5-прокси, но стандартный `fetch-socks` не выставляет опции сокета, необходимые для закрепления семейства прокси-hop. OmniRoute поставляет собственный коннектор для этого:

```ts
// open-sse/utils/socksConnectorWithFamily.ts
export function buildSocksFamilySocketOptions(family: 4 | 6 | null): Record<string, unknown> {
  if (family === 6) return { family: 6, autoSelectFamily: false };
  if (family === 4) return { family: 4, autoSelectFamily: false };
  return {};
}
```

`createProxyDispatcher` выбирает коннектор в зависимости от того, закреплено ли семейство:

- `family === null` (т.е. `auto` по имени хоста) → стандартный `socksDispatcher` из `fetch-socks`.
- `family === 4 | 6` → `createSocksDispatcherWithFamily`, который прокидывает `socket_options` в `SocksClient.createConnection`, чтобы Happy Eyeballs не мог выбрать IPv4 для политики выхода только IPv6.

Поддержка SOCKS5 сама по себе включена по умолчанию (отказ через `ENABLE_SOCKS5_PROXY=false`); см. [PROXY_GUIDE.md → Переменные окружения](../ops/PROXY_GUIDE.md#environment-variables).

---

## Поведение fail-closed

Весь смысл директивы — **отказывать**, а не молча откатываться на неправильное семейство. Две защиты обеспечивают это:

1. **Противоречие литералу** — директива, противоречащая IP-литералу хоста, выбрасывает ошибку во время построения диспетчера (`resolveDispatcherFamily`, показано выше).

2. **Предполётная DNS-проверка имени хоста** — для прокси-имени с закреплённым семейством `proxyFetch.ts` проверяет, что у имени действительно есть запись в требуемом семействе **до** выхода, через `assertHostnameSupportsFamily`:

   ```ts
   // open-sse/utils/proxyFamilyResolve.ts
   const hasFamily = records.some((r) => r.family === family);
   if (!hasFamily) {
     throw new Error(
       `[ProxyFamily] Proxy host ${host} has no ${family === 6 ? "IPv6 (AAAA)" : "IPv4 (A)"} record; ` +
         `refusing ${family === 6 ? "IPv6" : "IPv4"}-only egress (fail-closed)`
     );
   }
   ```

   При сбое `proxyFetch.ts` помечает ошибку как `code = "PROXY_FAMILY_UNAVAILABLE"` и `statusCode = 503`. Ошибка DNS-разрешения аналогично трактуется как fail-closed (отказ от выхода).

Хосты с IP-литералами — no-op для предполётной DNS-проверки: их семейство присуще и не требует поиска.

---

## Модель данных

Колонка `family` была добавлена миграцией `099_proxy_family.sql` в **две** таблицы:

```sql
-- src/lib/db/migrations/099_proxy_family.sql
ALTER TABLE proxy_registry ADD COLUMN family TEXT NOT NULL DEFAULT 'auto';
ALTER TABLE upstream_proxy_config ADD COLUMN family TEXT NOT NULL DEFAULT 'auto';
```

- `proxy_registry.family` — директива на прокси для записей реестра (`src/lib/db/proxies.ts`). Запросы разрешения выбирают `family` вместе с другими колонками прокси, а отсутствующее/не-строковое значение приводится к `"auto"`.
- `upstream_proxy_config.family` — директива для записей апстрим-прокси (`src/lib/db/upstreamProxy.ts`) с тем же значением по умолчанию `"auto"`.

Когда разрешённый объект прокси несёт `family` не-`auto`, `proxyConfigToUrl` добавляет маркер `?family=`, чтобы закрепление сохранялось вплоть до диспетчера.

---

## Связанная документация

> 📖 **Связанная документация:**
>
> - [Руководство по прокси](../ops/PROXY_GUIDE.md) — полная система прокси: CRUD реестра, 4-уровневое разрешение, ротация, проверка здоровья, справочник API
> - [Руководство по скрытности](./STEALTH_GUIDE.md) — слои отпечатков TLS и CLI, работающие поверх прокси
> - [Уровни защиты маршрутов](./ROUTE_GUARD_TIERS.md) — принуждение к loopback для локальных маршрутов
