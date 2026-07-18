---
title: Обнаружение бана аккаунта / запрещённых ключевых слов
---

# Обнаружение бана аккаунта / запрещённых ключевых слов

OmniRoute сканирует ошибки апстрима на сигналы, указывающие, что аккаунт провайдера
**окончательно мёртв** (приостановлен / деактивирован / заблокирован за нарушение ToS), и при
совпадении переводит это подключение в **терминальное состояние `banned`**, чтобы оно
больше не выбиралось для запросов. Именно это настраивает карточка **Security → Banned Keywords**
в настройках («Additional keywords that trigger permanent account ban detection. Built-in
keywords always apply.»).

Эта страница документирует встроенный список, процесс обнаружения, его область действия,
как безопасно добавлять свои ключевые слова и как восстановить помеченное подключение.
Само терминальное состояние — часть модели устойчивости — см.
[RESILIENCE_GUIDE](../architecture/RESILIENCE_GUIDE.md) («Terminal states»).

**Источник истины:** `open-sse/services/accountFallback.ts`
(`ACCOUNT_DEACTIVATED_SIGNALS`, `getMergedBannedSignals()`, `isAccountDeactivated()`).

## Встроенные ключевые слова

Эти 8 подстрок применяются всегда (без учёта регистра), независимо от пользовательского списка:

```
account_deactivated
account has been deactivated
account has been disabled
your account has been suspended
this account is deactivated
verify your account to continue                                 (Antigravity / Google Cloud Code)
this service has been disabled in this account for violation    (Antigravity)
this service has been disabled in this account                  (Antigravity)
```

> Этот список развивается по мере того, как провайдеры меняют формулировки банов.
> Авторитетный вариант — `ACCOUNT_DEACTIVATED_SIGNALS` в `open-sse/services/accountFallback.ts`;
> считайте блок выше снимком состояния.

Две смежные, **отдельные** таблицы сигналов живут в том же файле и *не* являются частью
обнаружения запрещённых слов:

- `CREDITS_EXHAUSTED_SIGNALS` — биллинг/квота исчерпаны (`insufficient_quota`,
  `credit_balance_too_low`, `payment required`, …) → терминальный `credits_exhausted`.
- `OAUTH_INVALID_TOKEN_SIGNALS` — **не терминальный**; может восстановиться обновлением токена.

Примечание: частые временные фразы вроде **`rate limit`** / `429` обрабатываются путём
rate-limit / охлаждения подключения и **не** являются сигналами бана.

## Процесс обнаружения

```
ошибка апстрима
  → тело приводится к строке в нижнем регистре
  → isAccountDeactivated(body): getMergedBannedSignals().some(sig => body.includes(sig))   [поиск подстроки]
  → совпадение?
      → connection testStatus = "banned"      (навсегда — охлаждение 1 год, никогда не восстанавливается автоматически)
      → если настройка `autoDisableBannedAccounts` включена → также isActive = false
      → подключение пропускается при выборе аккаунта (комбо-статусы QUOTA_BLOCKING)
```

- Совпадение — это **поиск подстроки без учёта регистра** по **телу** ответа
  (`isAccountDeactivated`, `accountFallback.ts`).
- Постоянная терминализация в `banned` срабатывает на теле с сигналом бана при **любом
  HTTP-статусе** (через `markAccountUnavailable` → `checkFallbackError`). Более узкая
  метка **`deactivated`** (`isActive=false`, когда у подключения нет запасных API-ключей)
  записывается встроенным путём `chatCore.ts` при **HTTP 401 / 403**
  (классификация через `classifyProviderError` → `ACCOUNT_DEACTIVATED`). Обратите внимание:
  путь `markAccountUnavailable()` записывает *другой* терминальный статус —
  **`expired`** — для того же сигнала `ACCOUNT_DEACTIVATED` (через
  `resolveTerminalConnectionStatus`), поэтому один и тот же бан может проявиться как
  `deactivated` или `expired` в зависимости от того, какой путь обработал ответ. (Старый
  комментарий в коде говорит «когда тело 401 содержит эти строки» — это занижает
  текущее поведение.)
- Подключение в статусе `banned` исключается из выбора везде, где фильтруются терминальные
  статусы (`isTerminalConnectionStatus`, комбо `QUOTA_BLOCKING_CONNECTION_STATUSES`).

## Область — какие провайдеры сканируются

**Все провайдеры.** Проверка выполняется в общем конвейере обработки ошибок, через который
проходит каждый неудачный апстрим-запрос — она **не** ограничена скрейперами
OAuth/подписок. Результирующее терминальное состояние — на уровне **подключения**,
а не провайдера.

При этом встроенные *строки* ориентированы на провайдеров подписок/OAuth с реальным
риском бана (ChatGPT Web, Claude Web, Codex, Muse Spark, Antigravity). Провайдер
с API-ключом сработает на детекторе, только если тело его ошибки буквально содержит
одну из подстрок.

## Пользовательские запрещённые ключевые слова

Добавляйте или удаляйте ключевые слова в **Security → Banned Keywords** (сохраняется как
глобальная настройка `customBannedSignals` через `PATCH /api/settings`). Они **добавляются** к
встроенному списку — никогда не заменяют его — и применяются на лету при сохранении (и при
запуске) через `setCustomBannedSignals()`. Каждое ключевое слово ограничено 200 символами;
ограничения на длину массива нет.

**⚠ Риск ложных срабатываний — выбирайте специфичные фразы.** Обнаружение — это прямой
поиск подстроки по всему телу ответа, и совпадение **постоянно** (охлаждение 1 год,
ручное восстановление). Широкое ключевое слово может заблокировать совершенно здоровое
подключение:

- **Плохо:** `quota`, `limit`, `error`, `denied` — встречаются во многих временных ошибках.
- **Хорошо:** полные фразы бана, например `your account has been suspended for`,
  `account permanently banned`, `violation of our terms`.

Предпочитайте самую длинную однозначную фразу, которую провайдер возвращает при реальном
бане. Если сомневаетесь, сначала понаблюдайте за `lastError` подключения, затем добавьте
точную формулировку.

## Восстановление помеченного подключения

Терминальные состояния `banned` / `deactivated` **никогда не восстанавливаются автоматически**
(они исключены из такта проактивного восстановления — сами собой восстанавливаются только
охлаждения `unavailable`). Оператор должен снять их явно:

1. **Повторно протестируйте подключение** — действие **Test** в панели
   (`POST /api/providers/{id}/test`); успешная проба сбрасывает `testStatus` в
   `active` и очищает поля ошибок.
2. **Переаутентифицируйтесь / отредактируйте учётные данные** — для OAuth-провайдеров
   повторите поток входа / обновления токена; маршруты создания/импорта провайдера
   устанавливают `isActive = true`.
3. **Повторно включите подключение** — если `autoDisableBannedAccounts` установил
   `isActive = false`, включите его обратно после исправления аккаунта.

Отдельной кнопки «снять флаг бана» нет — восстановление — это повторный тест,
переаутентификация или повторное включение, что соответствует общему правилу терминальных
состояний в [RESILIENCE_GUIDE](../architecture/RESILIENCE_GUIDE.md).

## Исходные файлы

| Задача | Файл |
| --- | --- |
| Таблицы сигналов + совпадение | `open-sse/services/accountFallback.ts` |
| Терминализация / сохранение | `src/sse/services/auth.ts` (`markAccountUnavailable`, `resolveTerminalConnectionStatus`, `clearAccountError`) |
| Встроенная классификация | `open-sse/handlers/chatCore.ts`, `open-sse/services/errorClassifier.ts` |
| Исключение терминальных состояний из восстановления | `src/lib/quota/connectionRecovery.ts` |
| Загрузка пользовательских ключевых слов в рантайме | `src/lib/config/runtimeSettings.ts` (`setCustomBannedSignals`) |
| UI настроек | `src/app/(dashboard)/dashboard/settings/components/SecurityTab.tsx` |
