# DUSA Tablet - App Development Guide

> **PURPOSE**: AI guide for creating new tablet apps (both internal and external iframe apps).

---

## App Types Overview

| Type | Location | Escrow | Use Case |
|------|----------|--------|----------|
| **Built-in App** | `web/src/components/apps/` | LOCKED | Core tablet functionality |
| **External iframe App** | External resource | N/A | Third-party integrations |
| **Config-based POI** | `config/config.lua` | OPEN | Maps app locations |

---

## External App Registration (RECOMMENDED)

External apps run in iframes and are registered via exports. This is the **recommended** approach since it doesn't require modifying locked tablet code.

### Server-Side Registration

**Location**: External resource's server script

```lua
-- your_resource/server/main.lua

CreateThread(function()
    -- Wait for tablet to be ready
    while GetResourceState('dusa_tablet') ~= 'started' do
        Wait(500)
    end
    Wait(1000)  -- Extra safety delay

    -- Register app
    exports['dusa_tablet']:RegisterApp({
        id = 'myapp',                           -- Unique identifier
        name = 'My Application',                -- Display name
        icon = 'myapp.svg',                     -- Icon in tablet's app icons
        description = 'App description',        -- Optional tooltip
        ui = {
            type = 'iframe',
            url = 'https://cfx-nui-your_resource/dist/index.html',
        },
        permissions = {                         -- Optional restrictions
            job = 'police',                     -- Single job
            -- OR
            job = {'police', 'sheriff'},        -- Multiple jobs
            -- OR
            gang = 'ballas',                    -- Gang-based
        },
    })
end)

-- Cleanup on resource stop
AddEventHandler('onResourceStop', function(resourceName)
    if resourceName == GetCurrentResourceName() then
        exports['dusa_tablet']:UnregisterApp('myapp')
    end
end)
```

### Client-Side Registration

**Location**: External resource's client script

```lua
-- your_resource/client/main.lua

CreateThread(function()
    while GetResourceState('dusa_tablet') ~= 'started' do
        Wait(500)
    end
    Wait(1000)

    exports['dusa_tablet']:RegisterApp({
        id = 'myapp',
        name = 'My Application',
        icon = 'myapp.svg',
        ui = {
            type = 'iframe',
            url = 'https://cfx-nui-your_resource/dist/index.html',
        },
        permissions = {
            job = 'mechanic',
        },
        -- Client-side callbacks (optional)
        onOpen = function()
            print('App opened')
            -- Start any client-side logic
        end,
        onClose = function()
            print('App closed')
            -- Cleanup client-side logic
        end,
    })
end)
```

### App Data Structure

```lua
---@class AppRegistration
---@field id string Unique app identifier (required)
---@field name string Display name (required)
---@field icon string Icon filename - place in tablet's icons folder (required)
---@field description? string Tooltip description
---@field ui AppUI UI configuration (required for iframe apps)
---@field permissions? AppPermissions Access restrictions
---@field onOpen? function Client callback when app opens
---@field onClose? function Client callback when app closes

---@class AppUI
---@field type 'iframe' Currently only iframe supported
---@field url string Full NUI URL to app's index.html

---@class AppPermissions
---@field job? string|string[] Required job(s)
---@field gang? string|string[] Required gang(s)
```

---

## iframe App Communication

### Parent (Tablet) → iframe (App)

Tablet sends messages to iframe apps via `postMessage`:

```typescript
// In your app's React/JS code
window.addEventListener('message', (event) => {
    const { action, data } = event.data;

    switch (action) {
        case 'TABLET_APP_OPENED':
            // App was opened, initialize
            console.log('App opened with data:', data);
            break;

        case 'TABLET_APP_CLOSED':
            // App is closing, cleanup
            break;

        case 'TABLET_LANGUAGE_CHANGE':
            // Language changed, update i18n
            const newLocale = data.locale; // 'en', 'tr', etc.
            i18n.changeLanguage(newLocale);
            break;

        case 'TABLET_VISIBILITY_CHANGE':
            // Tablet hidden/shown (for minigames etc.)
            const isVisible = data.visible;
            break;
    }
});
```

### iframe (App) → Parent (Tablet)

App sends messages to tablet:

```typescript
// Request to close tablet
window.parent.postMessage({
    action: 'CLOSE_TABLET',
}, '*');

// Request current language
window.parent.postMessage({
    action: 'REQUEST_TABLET_LANGUAGE',
}, '*');

// Show notification
window.parent.postMessage({
    action: 'SHOW_NOTIFICATION',
    data: {
        app: 'My App',
        title: 'Success',
        description: 'Operation completed',
    },
}, '*');
```

### NUI Callback from iframe

For server communication, iframe apps should use their own resource's NUI callbacks:

```typescript
// your_resource/web/src/utils/fetchNui.ts
export async function fetchNui<T>(eventName: string, data?: any): Promise<T> {
    const resourceName = (window as any).GetParentResourceName?.() || 'your_resource';

    const response = await fetch(`https://${resourceName}/${eventName}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data || {}),
    });

    return response.json();
}

// Usage
const result = await fetchNui('myapp:getData', { id: 123 });
```

```lua
-- your_resource/client/nui.lua
RegisterNUICallback('myapp:getData', function(data, cb)
    local result = lib.callback.await('myapp:getData', false, data)
    cb(result)
end)
```

---

## Language Sync Hook

For external apps to sync with tablet's language setting:

```typescript
// your_resource/web/src/hooks/useTabletLanguage.ts
import { useEffect } from 'react';
import i18n from '../i18n';

export function useTabletLanguage() {
    useEffect(() => {
        // Request current language on mount
        window.parent.postMessage({ action: 'REQUEST_TABLET_LANGUAGE' }, '*');

        // Listen for language changes
        const handler = (event: MessageEvent) => {
            const { action, data } = event.data || {};

            if (action === 'TABLET_LANGUAGE_RESPONSE' || action === 'TABLET_LANGUAGE_CHANGE') {
                if (data?.locale) {
                    i18n.changeLanguage(data.locale);
                }
            }
        };

        window.addEventListener('message', handler);
        return () => window.removeEventListener('message', handler);
    }, []);
}

// Usage in App.tsx
function App() {
    useTabletLanguage();  // Auto-syncs language
    return <MyAppContent />;
}
```

---

## Icon Requirements

### Adding App Icon

1. **Format**: SVG preferred, PNG accepted
2. **Size**: 60x60px recommended
3. **Location**: Place in tablet's icon folder or use external URL
4. **Naming**: Match the `icon` field in registration

**For external apps**, you can:
- Use a full URL: `icon = 'https://your-cdn.com/myapp-icon.svg'`
- Reference tablet's built-in icons: `icon = 'settings.svg'`
- Add icon to tablet's `dist/icons/` folder (requires rebuild)

---

## Permission Examples

### Job-Based

```lua
-- Single job
permissions = { job = 'police' }

-- Multiple jobs (OR logic - any match grants access)
permissions = { job = {'police', 'sheriff', 'fib'} }

-- Job with grade check (requires custom implementation)
permissions = { job = 'police', minGrade = 3 }
```

### Gang-Based

```lua
-- Single gang
permissions = { gang = 'ballas' }

-- Multiple gangs
permissions = { gang = {'ballas', 'vagos', 'families'} }
```

### Combined (OR Logic)

```lua
-- Player needs job OR gang
permissions = {
    job = 'mechanic',
    gang = 'lostmc',
}
```

### No Restrictions

```lua
-- Available to all players
permissions = {}
-- or omit permissions field entirely
```

---

## Complete External App Example

### File Structure

```
your_mechanic_app/
├── fxmanifest.lua
├── client/
│   └── main.lua
├── server/
│   └── main.lua
└── web/
    ├── src/
    │   ├── App.tsx
    │   ├── hooks/
    │   │   └── useTabletLanguage.ts
    │   └── utils/
    │       └── fetchNui.ts
    └── dist/
        └── index.html
```

### fxmanifest.lua

```lua
fx_version 'cerulean'
game 'gta5'

author 'Your Name'
description 'Mechanic App for DUSA Tablet'
version '1.0.0'

lua54 'yes'

dependencies {
    'dusa_tablet',
    'ox_lib',
}

client_scripts {
    'client/main.lua',
}

server_scripts {
    '@oxmysql/lib/MySQL.lua',
    'server/main.lua',
}

ui_page 'web/dist/index.html'

files {
    'web/dist/index.html',
    'web/dist/assets/**/*',
}
```

### server/main.lua

```lua
local isRegistered = false

CreateThread(function()
    while GetResourceState('dusa_tablet') ~= 'started' do
        Wait(500)
    end
    Wait(1000)

    local success = exports['dusa_tablet']:RegisterApp({
        id = 'mechanic-panel',
        name = 'Mechanic Panel',
        icon = 'wrench.svg',  -- Use tablet's built-in icon
        description = 'Vehicle repair and tuning management',
        ui = {
            type = 'iframe',
            url = 'https://cfx-nui-your_mechanic_app/web/dist/index.html',
        },
        permissions = {
            job = 'mechanic',
        },
    })

    if success then
        isRegistered = true
        print('[MechanicApp] Registered with tablet')
    else
        print('[MechanicApp] Failed to register')
    end
end)

AddEventHandler('onResourceStop', function(resourceName)
    if resourceName == GetCurrentResourceName() and isRegistered then
        exports['dusa_tablet']:UnregisterApp('mechanic-panel')
    end
end)

-- Your server callbacks
lib.callback.register('mechanic:getWorkOrders', function(source)
    -- Return work orders from database
    return MySQL.query.await('SELECT * FROM mechanic_orders WHERE status = ?', {'pending'})
end)
```

### client/main.lua

```lua
-- NUI Callbacks for your app
RegisterNUICallback('mechanic:getWorkOrders', function(data, cb)
    local result = lib.callback.await('mechanic:getWorkOrders', false, data)
    cb(result or {})
end)

RegisterNUICallback('mechanic:createOrder', function(data, cb)
    local result = lib.callback.await('mechanic:createOrder', false, data)
    cb(result or { success = false })
end)
```

---

## Config-Based Apps (Maps POI)

For adding locations to the Maps app, modify `config/config.lua`:

```lua
-- config/config.lua [OPEN]

Config.MapsPOI = {
    -- Add your custom locations
    {
        id = 'mechanic-shop-1',
        name = 'Benny\'s Custom Shop',
        category = 'garage',           -- hospital, police, gas_station, garage, store, bank, restaurant
        coords = { x = -211.55, y = -1320.56, z = 30.89 },
        address = 'Strawberry, Los Santos',
        phone = '555-BENNY',
        hours = '09:00 - 21:00',
        description = 'Custom lowrider modifications',
    },
}
```

**Categories with built-in icons**:
- `hospital` - Medical cross
- `police` - Badge
- `gas_station` - Fuel pump
- `garage` - Wrench
- `store` - Shopping bag
- `bank` - Building
- `restaurant` - Utensils

---

## Troubleshooting App Registration

### App Not Appearing

1. **Check resource state**: Is `dusa_tablet` started?
2. **Check permissions**: Does player have required job/gang?
3. **Check console**: Any registration errors?
4. **Check timing**: Add delay after tablet starts

```lua
-- Debug registration
local success = exports['dusa_tablet']:RegisterApp(...)
print('Registration result:', success)

-- List registered apps
local apps = exports['dusa_tablet']:GetRegisteredApps()
print('Registered apps:', json.encode(apps))
```

### iframe Not Loading

1. **Check URL**: Is the NUI URL correct?
2. **Check CORS**: External URLs may be blocked
3. **Check build**: Is `web/dist/index.html` present?
4. **Check files**: Are files listed in `fxmanifest.lua`?

### Communication Not Working

1. **Check postMessage**: Are you using correct action names?
2. **Check origin**: iframe must be from FiveM NUI
3. **Check listener**: Is event listener added on mount?

---

## AI Resolution Notes

### When User Asks for New App

1. **External app** (recommended): Provide complete registration code
2. **Config POI**: Add to `Config.MapsPOI` in `config/config.lua` [OPEN]
3. **Built-in app**: Report as LOCKED, suggest external iframe approach

### Response for Built-in App Request

```markdown
## Built-in App Oluşturma

Built-in app'ler `web/src/components/apps/` klasöründe ve **LOCKED** (escrowed).

### Önerilen Alternatif: External iframe App

Aynı işlevselliği external resource olarak oluşturabilirsiniz:

1. Yeni FiveM resource oluşturun
2. React/Vue/plain HTML ile UI yapın
3. `exports['dusa_tablet']:RegisterApp()` ile kaydedin

Bu yaklaşım:
- Tablet koduna dokunmaz
- Bağımsız güncelleme yapılabilir
- Escrow sorunu olmaz

[Örnek kod sağla...]
```
