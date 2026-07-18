---
title: "Контрольные точки цепочки поставок"
---

# Контрольные точки цепочки поставок (Phase 8 · Block A)

OmniRoute публикует артефакты npm + Docker. Эти контрольные точки обеспечивают происхождение
(provenance), инвентаризацию (SBOM) и сканирование CVE — всё на OSS, встроено в рабочие
процессы релизов. Позиция **«сначала рекомендации»** — сейчас они отчитываются, а станут
блокирующими после первого зелёного релиза.

| Точка                 | Инструмент                                     | Где                           | Блокирует?               | Результат                                     |
| --------------------- | ---------------------------------------------- | ----------------------------- | ------------------------ | --------------------------------------------- |
| SLSA provenance (npm) | `npm --provenance` (OIDC)                      | `npm-publish.yml`             | только если публикация падает | бейдж npmjs / `npm audit signatures`     |
| SBOM npm              | `@cyclonedx/cyclonedx-npm`                     | `npm-publish.yml`             | только если генерация падает | ресурс релиза + артефакт                  |
| SBOM образа           | `anchore/sbom-action` (syft)                   | `docker-publish.yml` (merge)  | рекомендательно          | артефакт CycloneDX                            |
| Trivy CVE (SARIF)     | `aquasecurity/trivy-action`                    | `docker-publish.yml` (merge)  | рекомендательно          | SARIF (HIGH+CRITICAL) → вкладка Security      |
| Trivy CRITICAL gate   | `aquasecurity/trivy-action`                    | `docker-publish.yml` (merge)  | **блокирующий**          | `exit-code: '1'` при исправимом CRITICAL      |
| osv vulnCount         | `osv-scanner` (`check:vuln-ratchet --ratchet`) | `ci.yml` (`quality-extended`) | **блокирующий**          | трещотка `metrics.vulnCount` (direction:down) |
| OpenSSF Scorecard     | `ossf/scorecard-action`                        | `scorecard.yml` (cron)        | рекомендательно          | SARIF → Security + бейдж                      |

Трещотка CVE образа использует **два шага** в `docker-publish.yml`: шаг SARIF
(`HIGH,CRITICAL`, `exit-code: 0`) держит HIGH+CRITICAL видимыми на вкладке Security,
не блокируя; шаг _CRITICAL gate_ (`severity: CRITICAL`, `ignore-unfixed: true`,
`exit-code: 1`) проваливает релиз при CRITICAL CVE **с доступным исправлением**. `ignore-unfixed`
предотвращает блокировку релиза из-за CVE базового образа, для которого нет патча апстрима.

## ⚠️ Вариативность CVE (блокирующие точки osv/Trivy)

osv и Trivy сравнивают зависимости с базами CVE, которые **постоянно растут**. PR,
**не трогающий зависимости**, может внезапно стать красным из-за нового раскрытого
CVE в существующей зависимости (osv: измеренный `vulnCount` > базовой линии; Trivy:
новый исправимый CRITICAL в образе). **Это ОЖИДАЕМОЕ эксплуатационное поведение
блокирующей точки CVE, а не регрессия продукта.**

Когда osv или Trivy становятся красными из-за свежераскрытого CVE, действия такие:

1. **Обновите затронутую зависимость** (предпочтительно) — перейдите на исправленную версию
   через `overrides` в `package.json` (транзитивные зависимости) или пересоберите образ
   на пропатченной базе.
2. **Если исправления апстрима нет:**
   - **osv:** перебазируйте `metrics.vulnCount` в `config/quality/quality-baseline.json`
     (`npm run quality:ratchet -- --update` не покрывает выделенные точки — отредактируйте
     значение вручную, `direction:down`) с примечанием-обоснованием + отслеживающим issue.
   - **Trivy:** добавьте запись в `.trivyignore` (CVE-ID на строку) с комментарием-обоснованием
     + отслеживающим issue. `ignore-unfixed: true` уже автоматически покрывает CVE без патчей.

Обе точки **корректно ПРОПУСКАЮТСЯ** (exit 0), когда инструмент отсутствует или измерение
не удалось (osv-scanner нет в PATH, osv.dev/сеть недоступны, невалидный JSON) —
**ошибка измерения** никогда не блокирует, блокирует только **измеренная** регрессия.

## Бэклог: Scorecard из рекомендательного → блокирующий

После первого зелёного релиза с отчётностью Scorecard:

- Scorecard: трещотка оценки (замораживает измеренный балл; не может снижаться).

Дополняет точки Phase 7 (osv-scanner, gitleaks, actionlint+zizmor): zizmor
аудирует сами рабочие процессы; Scorecard измеряет общую позицию репозитория.
