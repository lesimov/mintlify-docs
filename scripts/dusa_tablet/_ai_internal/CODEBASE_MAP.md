# DUSA Tablet - AI Internal Codebase Map

> **PURPOSE**: This document is for AI assistants only. It maps the entire codebase with escrow status and provides guidance on which files can be modified.

## Escrow Status Legend

| Status | Meaning | AI Action |
|--------|---------|-----------|
| `OPEN` | escrow_ignore - Can be modified | Propose code changes directly |
| `LOCKED` | Escrowed - Cannot be modified | Report issue to user, suggest workaround |

---

## File Map by Module

### Configuration (OPEN)

```
config/
└── config.lua                    [OPEN] Main configuration
    - Config.General              - Locale, debug settings
    - Config.Telemetry            - Sentry DSN, rate limits
    - Config.Tablet               - Open method, keybinds, animation
    - Config.CustomApps           - Custom app limits
    - Config.Apps                 - Built-in app toggles
    - Config.Database             - Migration settings
    - Config.Dev                  - Dev mode, restart key
    - Config.ErrorCapture         - Error capture whitelist
    - Config.ImageUpload          - Camera upload provider (Discord/Fivemanage/Fivemerr)
    - Config.News                 - News channels with job mappings
    - Config.Messages             - Phone sync settings
    - Config.MapsPOI              - Points of interest data
```

### Shared Modules (LOCKED except locales)

```
shared/
├── constants.lua                 [LOCKED] Global constants
│   - TabletConstants.LOG_LEVELS
│   - TabletConstants.DEBUG_CATEGORIES
│   - TabletConstants.KVP_KEYS
│   - TabletConstants.NUI_EVENTS
│
├── debug_bridge.lua              [LOCKED] Cross-resource logging
│   - TabletDebugBridge.sendLog()
│   - TabletDebugBridge.createLogger()
│
└── locales/                      [OPEN] Translations
    ├── init.lua                  [OPEN] Locale loader
    ├── en.lua                    [OPEN] English translations
    └── tr.lua                    [OPEN] Turkish translations
```

### Server Modules (LOCKED except adapters)

```
server/
├── main.lua                      [LOCKED] Server entry point
│   - TabletServer.RegisterApp()
│   - TabletServer.UnregisterApp()
│   - TabletServer.GetRegisteredApps()
│   - Callbacks: dusa_tablet:getServerApps, tablet:getUserInfo, tablet:maps:getPOI
│
├── debug/
│   ├── logger.lua                [LOCKED] ServerLogger
│   ├── telemetry.lua             [LOCKED] ServerTelemetry
│   └── http.lua                  [LOCKED] HTTP debug endpoints
│
├── database/
│   ├── migration.lua             [LOCKED] TabletMigration
│   └── init.lua                  [LOCKED] TabletDatabase
│
└── apps/
    ├── mail.lua                  [LOCKED] Email system
    ├── news.lua                  [LOCKED] News/articles
    ├── camera.lua                [LOCKED] Screenshot upload
    └── messages/
        ├── init.lua              [LOCKED] Message system init
        ├── phoneDetector.lua     [LOCKED] Phone resource detection
        ├── phoneSyncCallbacks.lua [LOCKED] Callbacks
        └── adapters/             [OPEN] Phone adapters
            ├── baseAdapter.lua   [OPEN] Base class
            ├── lbPhoneAdapter.lua [OPEN] lb-phone support
            ├── qbPhoneAdapter.lua [OPEN] qb-phone support
            ├── qsSmartphoneAdapter.lua [OPEN] qs-smartphone support
            ├── roadphoneAdapter.lua [OPEN] roadphone support
            └── gksphoneAdapter.lua [OPEN] gksphone support
```

### Client Modules (LOCKED)

```
client/
├── main.lua                      [LOCKED] Client entry point
│   - TabletClient.Open()
│   - TabletClient.Close()
│   - TabletClient.Toggle()
│   - TabletClient.IsOpen()
│   - TabletClient.Hide() / TabletClient.Show()
│   - TabletClient.RegisterApp()
│   - TabletClient.ShowNotification()
│
└── debug/
    └── logger.lua                [LOCKED] TabletLogger
```

### Telemetry Module (OPEN config only)

```
telemetry/
├── config.lua                    [OPEN] Telemetry configuration
│   - DusaTraceConfig.Database
│   - DusaTraceConfig.Alerts (Discord webhook)
│   - DusaTraceConfig.Dashboard
│   - DusaTraceConfig.Tracing
│   - DusaTraceConfig.Thresholds
│   - DusaTraceConfig.Debug
│
├── shared/
│   └── constants.lua             [LOCKED] Trace constants
│
├── server/
│   ├── trace.lua                 [LOCKED] DusaTrace engine
│   ├── storage.lua               [LOCKED] MySQL storage
│   ├── hooks.lua                 [LOCKED] Callback hooks
│   ├── alerts.lua                [LOCKED] Discord alerts
│   ├── dashboard_api.lua         [LOCKED] Dashboard API
│   ├── init.lua                  [LOCKED] Initialization
│   └── json_exporter.lua         [LOCKED] JSON export
│
└── client/
    ├── trace_client.lua          [LOCKED] DusaTraceClient
    └── dashboard_nui.lua         [LOCKED] Dashboard NUI
```

### Web/NUI (Build Output - DO NOT EDIT)

```
web/                              [SOURCE - dev branch only]
└── src/                          React source code

dist/                             [BUILD OUTPUT - DO NOT EDIT]
├── index.html
└── assets/
```

### SQL Migrations

```
sql/
├── schema.sql                    [REFERENCE] Initial schema
└── migrations/                   [REFERENCE] Migration files
```

---

## Global Tables Reference

### Core Tables (LOCKED)

| Table | Side | Purpose |
|-------|------|---------|
| `TabletConstants` | Shared | Constants and enums |
| `TabletDebugBridge` | Shared | Cross-resource logging |
| `ServerLogger` | Server | Server-side logging |
| `TabletServer` | Server | App registration, callbacks |
| `TabletMigration` | Server | Database migrations |
| `TabletDatabase` | Server | Database operations |
| `TabletClient` | Client | Tablet UI control |
| `TabletLogger` | Client | Client-side logging |

### Configurable Tables (OPEN)

| Table | File | Purpose |
|-------|------|---------|
| `Config` | config/config.lua | Main configuration |
| `DusaTraceConfig` | telemetry/config.lua | Telemetry settings |
| `TabletLocale` | shared/locales/init.lua | Translation system |
| `PhoneAdapter` | server/apps/messages/adapters/baseAdapter.lua | Phone adapter base |

---

## Common Issue Categories

### 1. Configuration Issues (OPEN - Can Fix)

**Location**: `config/config.lua`

| Issue | Config Path | Fix |
|-------|-------------|-----|
| Wrong language | `Config.General.Locale` | Change to valid locale code |
| Tablet won't open | `Config.Tablet.OpenMethod` | Check keybind/command/item setup |
| Animation not working | `Config.Tablet.Animation.Enabled` | Toggle or adjust dict/anim |
| Camera upload fails | `Config.ImageUpload.Provider` | Check webhook/API key |
| News channel missing | `Config.News.Channels` | Add channel with job mapping |
| Phone sync not working | `Config.Messages.Enabled` | Enable and check adapter |

### 2. Translation Issues (OPEN - Can Fix)

**Location**: `shared/locales/`

| Issue | File | Fix |
|-------|------|-----|
| Missing translation | `en.lua` or `tr.lua` | Add missing key |
| Wrong translation | `en.lua` or `tr.lua` | Update value |
| New language needed | Create `{lang}.lua` | Copy en.lua and translate |

### 3. Phone Adapter Issues (OPEN - Can Fix)

**Location**: `server/apps/messages/adapters/`

| Issue | Fix |
|-------|-----|
| New phone resource | Create new adapter extending PhoneAdapter |
| Query returns wrong data | Update SQL in adapter |
| Transform format wrong | Fix TransformConversation/TransformMessage |

### 4. Telemetry Issues (OPEN config only)

**Location**: `telemetry/config.lua`

| Issue | Config Path | Fix |
|-------|-------------|-----|
| Discord alerts not working | `DusaTraceConfig.Alerts.DiscordWebhook` | Update webhook URL |
| Too many alerts | `DusaTraceConfig.Thresholds.*` | Adjust thresholds |
| Dashboard not opening | `DusaTraceConfig.Dashboard.Enabled` | Enable dashboard |

### 5. Core Logic Issues (LOCKED - Report to User)

**Affected Files**: All non-escrow_ignore files

**AI Response Template**:
```
Bu sorun kilitli (escrowed) bir dosyada:
- Dosya: [file_path]
- Sorun: [description]
- Geçici çözüm: [workaround if any]

Bu sorunu çözmek için geliştirici ile iletişime geçmeniz gerekiyor.
```

---

## Database Tables

### Tablet Core Tables

| Table | Purpose | Created By |
|-------|---------|------------|
| `tablet_migrations` | Migration tracking | migration.lua |
| `tablet_mail_users` | Email users | mail.lua |
| `tablet_mail_threads` | Email threads | mail.lua |
| `tablet_mail_thread_participants` | Thread participants | mail.lua |
| `tablet_mail_messages` | Email messages | mail.lua |
| `tablet_news_channels` | News categories | news.lua |
| `tablet_news_articles` | News articles | news.lua |

### Telemetry Tables

| Table | Purpose | Created By |
|-------|---------|------------|
| `dusatrace_migrations` | Telemetry migrations | storage.lua |
| `dusatrace_traces` | Historical traces | storage.lua |

---

## Callback Reference

### Server Callbacks (lib.callback)

| Callback | File | Purpose |
|----------|------|---------|
| `dusa_tablet:getServerApps` | server/main.lua | Get registered apps |
| `dusa_tablet:checkPermission` | server/main.lua | Check app permission |
| `tablet:getUserInfo` | server/main.lua | Get player name |
| `tablet:maps:getPOI` | server/main.lua | Get POI data |
| `tablet:mail:*` | server/apps/mail.lua | Email operations |
| `tablet:news:*` | server/apps/news.lua | News operations |
| `tablet:camera:getConfig` | server/apps/camera.lua | Camera config |
| `tablet:messages:*` | server/apps/messages/phoneSyncCallbacks.lua | Message sync |
| `tablet:youtube:*` | server/main.lua | YouTube proxy |

---

## Exports Reference

### Server Exports

```lua
exports['dusa_tablet']:RegisterApp(appData)
exports['dusa_tablet']:UnregisterApp(appId)
exports['dusa_tablet']:GetRegisteredApps()
exports['dusa_tablet']:getDebugBridge()
exports['dusa_tablet']:runMigration(migrationName)
```

### Client Exports

```lua
exports['dusa_tablet']:Open()
exports['dusa_tablet']:Close()
exports['dusa_tablet']:Toggle()
exports['dusa_tablet']:IsOpen()
exports['dusa_tablet']:HideTablet()
exports['dusa_tablet']:ShowTablet()
exports['dusa_tablet']:GetTabletState()
exports['dusa_tablet']:RegisterApp(appData)
exports['dusa_tablet']:UnregisterApp(appId)
exports['dusa_tablet']:ShowNotification(app, title, desc, time)
exports['dusa_tablet']:GetCurrentLocale()
```

---

## AI Decision Tree

```
User reports issue
    │
    ├─> Is it configuration related?
    │   └─> YES: Check config/config.lua [OPEN]
    │       └─> Propose config change
    │
    ├─> Is it translation related?
    │   └─> YES: Check shared/locales/*.lua [OPEN]
    │       └─> Propose translation fix
    │
    ├─> Is it phone adapter related?
    │   └─> YES: Check server/apps/messages/adapters/*.lua [OPEN]
    │       └─> Propose adapter fix or new adapter
    │
    ├─> Is it telemetry config related?
    │   └─> YES: Check telemetry/config.lua [OPEN]
    │       └─> Propose config change
    │
    └─> Is it core logic?
        └─> YES: File is [LOCKED]
            └─> Report to user:
                "Bu sorun escrowed dosyada. Geliştirici iletişimi gerekli."
                + Suggest workaround if possible
```

---

## Quick Reference Commands

```lua
-- Debug level control (in-game)
/tablet_debug level 2    -- Set to DEBUG
/tablet_debug level 3    -- Set to INFO (default)
/tablet_debug level 5    -- Set to ERROR only

-- Telemetry dashboard (admin)
/telemetry               -- Open dashboard

-- Resource restart (dev mode)
Press CAPS LOCK          -- Restart resource (if Config.Dev.Enabled)
```
