---
title: "Руководство по десктопному приложению Electron"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Руководство по десктопному приложению Electron

> **Источник истины:** воркспейс `electron/`
> **Последнее обновление:** 2026-06-28 — v3.8.40

OmniRoute поставляет кросс-платформенное десктопное приложение (Windows / macOS / Linux), построенное на
**Electron 41** + **electron-builder 26.10**. Десктопное приложение запускает standalone-сервер Next.js
как дочерний процесс, направляет на него `BrowserWindow` и добавляет
системный трей, автообновление, IPC-мост и начальную загрузку секретов без конфигурации.

## Архитектура

```
┌──────────────────────────────────────────────┐
│ Главный процесс Electron (electron/main.js)  │
│ ├─ Блокировка единственного экземпляра        │
│ ├─ Дочерний процесс: standalone-сервер Next.js │
│ │   (запускается средой Node из Electron)    │
│ ├─ BrowserWindow → http://localhost:PORT     │
│ ├─ Системный трей + контекстное меню         │
│ ├─ Автообновление через electron-updater     │
│ ├─ Content Security Policy (заголовки session) │
│ └─ Начальная загрузка секретов (JWT / API_KEY_SECRET) │
└──────────────────────────────────────────────┘
            ↕ IPC-мост (electron/preload.js)
┌──────────────────────────────────────────────┐
│ Рендерер (панель Next.js)                     │
│   window.electronAPI.* (contextIsolation)     │
└──────────────────────────────────────────────┘
```

## Версии

Подтверждено из `electron/package.json`:

| Пакет              | Версия                     |
| ------------------ | -------------------------- |
| `electron`         | `^41.5.1`                  |
| `electron-builder` | `^26.10.0`                 |
| `electron-updater` | `^6.8.5`                   |
| `better-sqlite3`   | `^12.9.0`                  |
| Версия приложения  | `3.8.0`                    |
| ID приложения      | `online.omniroute.desktop` |
| Имя продукта       | `OmniRoute`                |

## Скрипты (корневой `package.json`)

| Скрипт                          | Назначение                                                                 |
| ------------------------------- | -------------------------------------------------------------------------- |
| `npm run electron:dev`          | Запускает `npm run dev` + ждёт `localhost:20128` + запускает Electron      |
| `npm run electron:build`        | Собирает Next.js, затем запускает `electron-builder` для текущей ОС        |
| `npm run electron:build:win`    | Собирает установщик Windows NSIS + portable (x64)                          |
| `npm run electron:build:mac`    | Собирает DMG для macOS (Intel + Apple Silicon)                             |
| `npm run electron:build:linux`  | Собирает AppImage + DEB для Linux (x64 + arm64)                            |
| `npm run electron:smoke:packaged` | Запускает собранный бинарник и проверяет `/login` на HTTP 200, затем завершает |

Воркспейс `electron/` также предоставляет:

- `npm run prepare:bundle` — запускает `scripts/build/prepare-electron-standalone.mjs`
- `npm run build:mac-x64` / `build:mac-arm64` — одноархитектурные сборки macOS
- `npm run pack` — сборка только каталога для локального тестирования (без установщика)

## Структура каталогов

```
electron/
├── package.json              # Зависимости Electron + конфиг electron-builder
├── main.js                   # Главный процесс (24 КБ — см. аннотации ниже)
├── preload.js                # IPC-мост contextBridge
├── types.d.ts                # Типы AppInfo / ServerStatus / ElectronAPI
├── README.md                 # Заметки внутри воркспейса
├── assets/                   # icon.png, icon.ico, icon.icns, tray-icon.png
└── dist-electron/            # Вывод electron-builder (в .gitignore)

scripts/
├── build/
│   └── prepare-electron-standalone.mjs   # Готовит бандл .next/electron-standalone
└── dev/
    └── smoke-electron-packaged.mjs       # Пост-сборочный дымовой тест
```

И `main.js`, и `preload.js` — это **файлы CommonJS `.js`**, а не TypeScript. Типизации
со стороны рендерера находятся в `electron/types.d.ts`.

## IPC-мост (`preload.js`)

Preload предоставляет внесённый в белый список API на `window.electronAPI` с помощью `contextBridge`
при `contextIsolation: true` и `nodeIntegration: false`.

```javascript
const VALID_CHANNELS = {
  invoke: [
    "get-app-info",
    "open-external",
    "get-data-dir",
    "restart-server",
    "check-for-updates",
    "download-update",
    "install-update",
    "get-app-version",
  ],
  send: ["window-minimize", "window-maximize", "window-close"],
  receive: ["server-status", "port-changed", "update-status"],
};
```

Доступные методы:

| Вызов рендерера                                                   | Тип                        |
| ----------------------------------------------------------------- | -------------------------- |
| `getAppInfo()` → `{ name, version, platform, isDev, port }`       | invoke                     |
| `openExternal(url)`                                               | invoke                     |
| `getDataDir()`                                                    | invoke                     |
| `restartServer()`                                                 | invoke                     |
| `getAppVersion()`                                                 | invoke                     |
| `checkForUpdates()` / `downloadUpdate()` / `installUpdate()`      | invoke                     |
| `minimizeWindow()` / `maximizeWindow()` / `closeWindow()`         | send                       |
| `onServerStatus(cb)` / `onPortChanged(cb)` / `onUpdateStatus(cb)` | receive (возвращает disposer) |

Хелперы receive возвращают **функцию-disposer**, а не полагаются на
`removeAllListeners` — это предотвращает накопление слушателей при перемонтировании компонентов React.

## Жизненный цикл сервера

`main.js` запускает standalone-бандл Next.js напрямую средой выполнения Node из
Electron, чтобы избежать несоответствия ABI нативных модулей с системным Node:

```js
spawn(process.execPath, [serverScript], {
  cwd: NEXT_SERVER_PATH,
  env: { ...serverEnv, PORT, NODE_ENV: "production", ELECTRON_RUN_AS_NODE: "1", NODE_PATH },
  stdio: "pipe",
});
```

Основные моменты:

- `waitForServer()` опрашивает URL до 30 с перед показом окна (нет пустого экрана при холодном старте).
- `stdio: "pipe"` захватывает stdout/stderr; фразы готовности (`Ready` / `listening`) генерируют `server-status: running` по IPC.
- `before-quit` ждёт до 5 с на корректный SIGTERM (checkpoint WAL), затем шлёт SIGKILL.
- Переключатель порта в трее (`20128`, `3000`, `8080`) останавливает и перезапускает сервер, затем перезагружает BrowserWindow.

## Начальная загрузка секретов без конфигурации

При первом запуске главный процесс автоматически генерирует и сохраняет недостающие секреты:

| Секрет                   | Источник                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------- |
| `JWT_SECRET`             | `crypto.randomBytes(64).toString("hex")`                                            |
| `STORAGE_ENCRYPTION_KEY` | `crypto.randomBytes(32).toString("hex")` (отказывает, если зашифрованные учётные данные уже есть) |
| `API_KEY_SECRET`         | `crypto.randomBytes(32).toString("hex")`                                            |

Сохраняются в `<DATA_DIR>/server.env`. `DATA_DIR` разрешается в:

- Windows: `%APPDATA%\omniroute`
- Linux: `$XDG_CONFIG_HOME/omniroute` или `~/.omniroute`
- macOS: `~/.omniroute`

## Окно и трей

- `BrowserWindow`: 1400×900 (мин. 1024×700), `backgroundColor: "#0a0a0a"`.
- macOS: `titleBarStyle: "hiddenInset"`, кнопки traffic-light в `{ x: 16, y: 16 }`.
- Windows/Linux: нативная строка заголовка.
- Кнопка закрытия сворачивает в трей; в меню трея: **Open OmniRoute**, **Open Dashboard** (внешний браузер), подменю **Server Port**, **Check for Updates**, **Quit**.

## Content Security Policy

Задаётся через `session.defaultSession.webRequest.onHeadersReceived`. Заметные директивы:

- `frame-ancestors 'none'`, `object-src 'none'`, `child-src 'none'`
- `connect-src 'self' http://localhost:* http://127.0.0.1:* ws://localhost:* ws://127.0.0.1:* https://*.omniroute.online https://*.omniroute.dev`
- В режиме разработки добавляет `'unsafe-eval'` только в `script-src`

## Автообновление

Используется `electron-updater` с провайдером GitHub (`diegosouzapw/OmniRoute`).

- `autoDownload = false`, `autoInstallOnAppQuit = true`
- События пересылаются в рендерер через IPC `update-status`:
  `checking`, `available`, `not-available`, `downloading` (с `percent`), `downloaded`, `error`
- `installUpdate()` завершает сервер, затем вызывает `autoUpdater.quitAndInstall()`
- Пропускается в режиме разработки (`!app.isPackaged`)

## Конвейер сборки

1. `npm run build` → standalone Next.js в `.next/standalone`.
2. `prepare-electron-standalone.mjs` → пересобирает в `.next/electron-standalone` и переписывает абсолютные пути внутри `server.js` + `required-server-files.json`, чтобы бандл был перемещаемым.
3. `electron-builder` упаковывает `main.js`, `preload.js`, `node_modules` и `extraResources: { ../.next/electron-standalone → app }`.

### Цели сборки

| ОС      | Цели                                       |
| ------- | ------------------------------------------ |
| Windows | Установщик NSIS + portable (x64)           |
| macOS   | DMG (Intel + arm64, перетаскивание в Applications) |
| Linux   | AppImage + DEB (x64 + arm64)               |

Настройки NSIS: `oneClick: false`, позволяет пользователю выбрать каталог установки, создаёт ярлыки на рабочем столе и в меню «Пуск».

## Дымовое тестирование собранной сборки

```bash
npm run electron:smoke:packaged
```

`scripts/dev/smoke-electron-packaged.mjs`:

- Автоматически находит собранный бинарник в `electron/dist-electron/` для текущей платформы.
- Запускает с изолированными каталогами `HOME`/`APPDATA`/`XDG_*`, чтобы не трогать данные разработчика.
- Опрашивает `http://127.0.0.1:20128/login` на HTTP 200 в течение 45 с.
- Следит за stderr/stdout на предмет фатальных паттернов (`Cannot find module`, `MODULE_NOT_FOUND`, `ERR_DLOPEN_FAILED`, `Failed to start server` и т.д.).
- Ждёт 2 с стабильной работы после готовности, затем посылает SIGTERM и ждёт освобождения порта.
- В CI автоматически передаёт `--no-sandbox --disable-gpu` (и `--disable-dev-shm-usage` на Linux).

Переопределения через окружение: `ELECTRON_SMOKE_APP_EXECUTABLE`, `ELECTRON_SMOKE_URL`, `ELECTRON_SMOKE_TIMEOUT_MS`, `ELECTRON_SMOKE_SETTLE_MS`, `ELECTRON_SMOKE_DATA_DIR`, `ELECTRON_SMOKE_KEEP_DATA`, `ELECTRON_SMOKE_STREAM_LOGS`.

## Подписание кода

`electron/package.json` **не** подключает учётные данные подписи напрямую. Передавайте их через переменные окружения в `electron-builder`:

### macOS

```bash
export APPLE_ID=<email>
export APPLE_APP_SPECIFIC_PASSWORD=<password>
export APPLE_TEAM_ID=<id>
export CSC_LINK=path/to/cert.p12
export CSC_KEY_PASSWORD=<cert-password>
npm run electron:build:mac
```

### Windows

```bash
export CSC_LINK=path/to/cert.pfx
export CSC_KEY_PASSWORD=<cert-password>
npm run electron:build:win
```

### Linux

Подписание AppImage опционально — задайте `LINUX_GPG_KEY` при подписании.

## Дистрибуция

Артефакты попадают в `electron/dist-electron/`:

- `OmniRoute Setup X.Y.Z.exe`, `OmniRoute-X.Y.Z-portable.exe` (Windows)
- `OmniRoute-X.Y.Z-mac.dmg`, `OmniRoute-X.Y.Z-arm64-mac.dmg` (macOS)
- `OmniRoute-X.Y.Z.AppImage`, `omniroute-desktop_X.Y.Z_amd64.deb` (Linux)

Релизы публикуются в GitHub Releases (`diegosouzapw/OmniRoute`) — там же `electron-updater` проверяет наличие новых версий.

## Устранение неполадок

| Симптом                                                         | Решение                                                                     |
| --------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `Cannot find module 'better-sqlite3'` после major-обновления Electron | `cd electron && npm rebuild`                                            |
| `ERR_DLOPEN_FAILED` для нативного модуля                        | Повторно запустите `prepare:bundle` и проверьте соответствие ABI Node из Electron |
| Окно появляется пустым на Linux                                 | Убедитесь, что сервер Next.js действительно привязался к PORT (проверьте логи `[Server]`) |
| Нотаризация macOS зависает                                      | Убедитесь, что переменные `APPLE_*` экспортированы, а не только в `.env`    |
| Предупреждение Windows SmartScreen                              | Подпишите EV-сертификатом, или пользователи: правый клик → «Run anyway»     |
| Дымовой тест падает с занятым портом                            | Остановите любой локальный dev-сервер на 20128 перед запуском `electron:smoke:packaged` |

## См. также

- [SETUP_GUIDE.md](./SETUP_GUIDE.md)
- [RELEASE_CHECKLIST.md](../ops/RELEASE_CHECKLIST.md)
- Исходники: `electron/main.js`, `electron/preload.js`, `electron/package.json`
- Хелперы: `scripts/build/prepare-electron-standalone.mjs`, `scripts/dev/smoke-electron-packaged.mjs`
