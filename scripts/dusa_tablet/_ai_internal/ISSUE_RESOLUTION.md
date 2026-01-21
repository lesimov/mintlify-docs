# DUSA Tablet - AI Issue Resolution Guide

> **PURPOSE**: Structured approach for AI to diagnose and resolve issues while respecting escrow boundaries.

---

## Resolution Protocol

### Step 1: Identify Issue Category

```
┌─────────────────────────────────────────────────────────────┐
│                    ISSUE CATEGORIES                          │
├─────────────────────────────────────────────────────────────┤
│ A. CONFIG      - Settings, toggles, values                   │
│ B. TRANSLATION - Missing/wrong text                          │
│ C. ADAPTER     - Phone system integration                    │
│ D. TELEMETRY   - Alerts, tracing, dashboard                  │
│ E. CORE        - Business logic, callbacks, database         │
│ F. NUI         - React UI, styling, components               │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Check Escrow Status

| Category | Files | Status | AI Action |
|----------|-------|--------|-----------|
| A. CONFIG | `config/config.lua` | **OPEN** | Fix directly |
| B. TRANSLATION | `shared/locales/*.lua` | **OPEN** | Fix directly |
| C. ADAPTER | `server/apps/messages/adapters/*.lua` | **OPEN** | Fix directly |
| D. TELEMETRY | `telemetry/config.lua` | **OPEN** | Fix directly |
| E. CORE | `server/*.lua`, `client/*.lua`, `shared/*.lua` (except locales) | **LOCKED** | Report to user |
| F. NUI | `web/src/**/*`, `dist/**/*` | **LOCKED** | Report to user |

---

## Category A: Configuration Issues

### A1. Tablet Won't Open

**Symptoms**: Player presses key/command but nothing happens

**Check Order**:
1. `Config.Tablet.OpenMethod` - Is it 'keybind', 'command', or 'item'?
2. If 'keybind': Check `Config.Tablet.Keybind.Key`
3. If 'command': Check `Config.Tablet.Command.Name`
4. If 'item': Check `Config.Tablet.Item.Name` and inventory integration
5. `Config.Tablet.AllowInVehicle` - Is player in vehicle?
6. `Config.Tablet.AllowWhileDead` - Is player dead?

**Fix Template**:
```lua
-- config/config.lua
Config.Tablet = {
    OpenMethod = 'keybind',  -- or 'command' or 'item'
    Keybind = {
        Key = 'TAB',  -- Change key here
    },
    AllowInVehicle = true,  -- Allow in vehicle
    AllowWhileDead = false,
}
```

### A2. Animation Issues

**Symptoms**: No tablet prop, wrong animation, prop floating

**Check Order**:
1. `Config.Tablet.Animation.Enabled` - Is animation enabled?
2. `Config.Tablet.Animation.Prop` - Is prop model valid?
3. `Config.Tablet.Animation.Offset` / `Rotation` - Position correct?

**Fix Template**:
```lua
Config.Tablet.Animation = {
    Enabled = true,
    Dict = 'amb@code_human_in_bus_passenger_idles@female@tablet@base',
    Anim = 'base',
    Prop = 'prop_cs_tablet',
    Bone = 28422,
    Offset = { x = 0.03, y = 0.002, z = -0.0 },
    Rotation = { x = 10.0, y = 160.0, z = 0.0 },
    Flags = 49,
}
```

### A3. Camera Upload Fails

**Symptoms**: Screenshot taken but not uploaded, no URL returned

**Check Order**:
1. `Config.ImageUpload.Provider` - Which provider?
2. Discord: Check `Config.ImageUpload.Discord.Webhook`
3. Fivemanage: Check `Config.ImageUpload.Fivemanage.ApiKey`
4. Fivemerr: Check `Config.ImageUpload.Fivemerr.ApiKey`

**Fix Template**:
```lua
Config.ImageUpload = {
    Provider = 'discord',  -- 'discord', 'fivemanage', 'fivemerr'
    Discord = {
        Webhook = 'https://discord.com/api/webhooks/YOUR_WEBHOOK_HERE',
    },
}
```

### A4. News Channel Missing or Wrong Job

**Symptoms**: Player can't write to channel, channel not visible

**Check**: `Config.News.Channels` array

**Fix Template**:
```lua
Config.News.Channels = {
    {
        id = 'custom_channel',
        name = 'Custom News',
        icon = '􀋚',           -- SF Symbol
        color = '#3B82F6',     -- Hex color
        description = 'Description here',
        job = 'reporter',      -- Job that can write
    },
}
```

### A5. Phone Message Sync Not Working

**Symptoms**: Messages app shows nothing, sync status false

**Check Order**:
1. `Config.Messages.Enabled` - Is sync enabled?
2. `Config.Messages.AutoDetect` - Is auto-detect on?
3. `Config.Messages.ForceAdapter` - Is specific adapter forced?
4. Check if phone resource is running

**Fix Template**:
```lua
Config.Messages = {
    Enabled = true,
    WriteEnabled = true,  -- or false for read-only
    AutoDetect = true,
    ForceAdapter = nil,   -- or 'lb', 'qb', 'qs', 'road', 'gks'
}
```

---

## Category B: Translation Issues

### B1. Missing Translation Key

**Symptoms**: UI shows key path like "settings.general.title" instead of text

**Location**: `shared/locales/en.lua` or `shared/locales/tr.lua`

**Fix Process**:
1. Find the missing key in the error/UI
2. Add to `en.lua` (English base)
3. Add to `tr.lua` (Turkish)

**Example**:
```lua
-- shared/locales/en.lua
return {
    settings = {
        general = {
            title = 'General Settings',  -- Add missing key
        },
    },
}
```

### B2. Wrong Translation

**Symptoms**: Text displays but is incorrect

**Fix**: Update value in appropriate locale file

### B3. New Language Support

**Process**:
1. Copy `shared/locales/en.lua` to `shared/locales/{lang}.lua`
2. Translate all values
3. Update `shared/locales/init.lua` to load new language
4. Add language to `Config.General.Locale` options

---

## Category C: Phone Adapter Issues

### C1. New Phone Resource Support

**Location**: `server/apps/messages/adapters/`

**Process**:
1. Create new file: `{phone}Adapter.lua`
2. Extend `PhoneAdapter` base class
3. Implement required methods

**Template**:
```lua
-- server/apps/messages/adapters/newPhoneAdapter.lua

local PhoneAdapter = PhoneAdapter  -- Global from baseAdapter.lua

NewPhoneAdapter = setmetatable({}, { __index = PhoneAdapter })
NewPhoneAdapter.__index = NewPhoneAdapter

function NewPhoneAdapter:new()
    local instance = PhoneAdapter:new('newphone')
    setmetatable(instance, self)
    return instance
end

function NewPhoneAdapter:GetPlayerPhoneNumber(identifier)
    local result = self:SafeSingle([[
        SELECT phone_number FROM newphone_users WHERE identifier = ?
    ]], { identifier })
    return result and result.phone_number or nil
end

function NewPhoneAdapter:GetContacts(phoneNumber)
    local contacts = self:SafeQuery([[
        SELECT * FROM newphone_contacts WHERE owner = ?
    ]], { phoneNumber })

    local transformed = {}
    for _, c in ipairs(contacts or {}) do
        table.insert(transformed, self:TransformContact(c))
    end
    return transformed
end

function NewPhoneAdapter:GetConversations(phoneNumber)
    -- Implement based on phone's database structure
end

function NewPhoneAdapter:GetMessages(phoneNumber, conversationId)
    -- Implement based on phone's database structure
end

function NewPhoneAdapter:TransformConversation(rawConv)
    return {
        id = rawConv.id,
        phone_number = rawConv.number,
        display_name = rawConv.name or rawConv.number,
        last_message = rawConv.last_message,
        last_message_time = rawConv.timestamp,
        unread_count = rawConv.unread or 0,
        is_muted = rawConv.muted == 1,
    }
end

function NewPhoneAdapter:TransformMessage(rawMsg, myPhoneNumber)
    return {
        id = rawMsg.id,
        content = rawMsg.message,
        timestamp = rawMsg.timestamp,
        is_sender = rawMsg.sender == myPhoneNumber,
        is_read = rawMsg.is_read == 1,
    }
end

return NewPhoneAdapter
```

### C2. Adapter Query Returns Wrong Data

**Debug Process**:
1. Check phone resource's database schema
2. Compare with adapter's SQL queries
3. Update queries to match actual schema

### C3. Transform Format Mismatch

**Required Format for Conversations**:
```lua
{
    id = string/number,      -- Unique ID
    phone_number = string,   -- Other party's number
    display_name = string,   -- Name to show
    last_message = string,   -- Preview text
    last_message_time = string/number,  -- Timestamp
    unread_count = number,   -- Unread badge
    is_muted = boolean,      -- Mute status
}
```

**Required Format for Messages**:
```lua
{
    id = string/number,      -- Unique ID
    content = string,        -- Message text
    timestamp = string/number,  -- When sent
    is_sender = boolean,     -- Did I send this?
    is_read = boolean,       -- Read status
}
```

---

## Category D: Telemetry Issues

### D1. Discord Alerts Not Working

**Check**: `telemetry/config.lua`

**Fix**:
```lua
DusaTraceConfig.Alerts = {
    Enabled = true,
    DiscordWebhook = 'https://discord.com/api/webhooks/VALID_WEBHOOK',
    SlowOperationAlerts = true,
    AnomalyDetection = true,
}
```

### D2. Too Many Alerts (Spam)

**Fix**: Adjust thresholds
```lua
DusaTraceConfig.Thresholds = {
    ErrorRateThreshold = 0.15,    -- Increase from 0.10
    SlowOperationMs = 2000,       -- Increase from 1000
    ErrorSpikeCount = 10,         -- Increase from 5
}
```

### D3. Dashboard Won't Open

**Check**:
```lua
DusaTraceConfig.Dashboard = {
    Enabled = true,               -- Must be true
    Command = 'telemetry',        -- /telemetry to open
}
```

---

## Category E: Core Logic Issues (LOCKED)

### Response Template for LOCKED Issues

When issue is in a locked file, respond with:

```
## Tespit Edilen Sorun

**Dosya**: `{file_path}`
**Durum**: LOCKED (Escrowed)
**Sorun**: {description}

### Bu Dosya Değiştirilemez

Bu dosya FiveM escrow koruması altında. Kod değişikliği yapılamaz.

### Önerilen Çözümler

1. **Geçici Çözüm**: {workaround if available}
2. **Kalıcı Çözüm**: Geliştirici ile iletişime geç
   - Discord: [link]
   - GitHub Issues: [link]

### İlgili Açık Dosyalar

Eğer sorun konfigürasyon ile çözülebilirse:
- `config/config.lua` - Ana ayarlar
- `telemetry/config.lua` - Telemetry ayarları
```

### Common LOCKED Issues and Workarounds

| Issue | Locked File | Workaround |
|-------|-------------|------------|
| Callback returns wrong data | server/apps/*.lua | Check if config can filter data |
| NUI not rendering | dist/**/* | Rebuild from web/ (dev branch) |
| Export not working | server/main.lua, client/main.lua | Check if calling correctly |
| Database query fails | server/database/*.lua | Check SQL schema matches |
| Animation broken | client/main.lua | Adjust Config.Tablet.Animation |

---

## Category F: NUI/React Issues (LOCKED)

### F1. UI Bug

**Response**:
```
NUI (React) kodları web/ klasöründe ve build output dist/ klasöründe.

Bu değişiklikler için:
1. dev branch'e geç
2. web/ klasöründe React kodunu düzenle
3. npm run build çalıştır
4. fivem branch'e merge et

Şu an sadece config değişiklikleri yapılabilir.
```

### F2. Missing Component/Feature

**Response**:
```
Yeni UI özelliği için:
1. GitHub'da feature request aç
2. veya dev branch'te React geliştirmesi yap

Mevcut config seçeneklerini kontrol et - belki toggle var.
```

---

## Quick Diagnostic Questions

AI should ask these to narrow down issues:

1. **Tablet açılmıyor mu?**
   - Hangi method kullanılıyor? (keybind/command/item)
   - Araçta mısın? Ölü müsün?
   - Console'da hata var mı?

2. **Uygulama çalışmıyor mu?**
   - Hangi uygulama?
   - Console'da hata mesajı?
   - Database tabloları oluşturuldu mu?

3. **Mesajlar senkronize olmuyor mu?**
   - Hangi telefon resource'u?
   - Config.Messages ayarları neler?
   - Telefon resource'u çalışıyor mu?

4. **Çeviri eksik mi?**
   - Hangi dil?
   - Eksik key nedir?
   - UI'da mı yoksa notification'da mı?

---

## File Modification Checklist

Before modifying any file, verify:

- [ ] File is in escrow_ignore list (OPEN)
- [ ] Backup of original config (if applicable)
- [ ] Lua syntax is valid
- [ ] No trailing commas in tables
- [ ] Strings properly quoted
- [ ] Tables properly closed

**escrow_ignore files**:
```
config/config.lua
shared/locales/*.lua
server/apps/messages/adapters/*.lua
telemetry/config.lua
```

All other .lua files are LOCKED.
