# Collation Uyumluluk Sistemi

Bu belge, `dusa_mechanicv2` için geliştirilen collation uyumluluk sistemini açıklar ve diğer projelere (örn: `dusa_tablet`) nasıl uygulanacağını gösterir.

## Problem

FiveM framework'leri farklı MySQL collation'ları kullanıyor:

| Framework | Collation |
|-----------|-----------|
| QBCore (yeni) | `utf8mb4_unicode_ci` |
| ESX (yeni) | `utf8mb4_unicode_ci` |
| ESX (legacy) | `utf8mb4_general_ci` |
| Bazı sunucular | `utf8mb4_general_ci` |

**Sonuç:** JOIN sorgularında `Illegal mix of collations` hatası:
```
Illegal mix of collations (utf8mb4_unicode_ci,IMPLICIT) and (utf8mb4_general_ci,IMPLICIT) for operation '='
```

## Çözüm Mimarisi

**ÖNEMLİ:** `DELIMITER` ve stored procedure'ler oxmysql üzerinden çalıştırılamaz!
Collation fix tamamen **Lua katmanında** yapılmalıdır.

```
┌─────────────────────────────────────────────────────────┐
│  LUA KATMANI (migration.lua) - TEK ÇÖZÜM               │
│     - Her başlangıçta collation kontrolü               │
│     - Uyumsuzluk varsa otomatik düzeltme               │
│     - Manuel komutlar (debug için)                     │
│     - Fresh install, legacy migration, normal migration │
│       tüm senaryolarda otomatik çalışır                │
└─────────────────────────────────────────────────────────┘
```

## Uygulama Adımları (Yeni Proje İçin)

### 1. SQL Migration Dosyası (Placeholder)

`sql/migrations/XXX_universal_collation_fix.sql` oluştur:

```sql
-- ============================================================================
-- Migration: XXX_universal_collation_fix
-- Description: Collation fix - handled by Lua layer
-- Date: 2026-01-03
--
-- NOTE: Stored procedures with DELIMITER are not supported by oxmysql.
--       Collation fix is handled automatically by the Lua migration system.
-- ============================================================================

-- This migration is intentionally empty - collation is handled by Lua
SELECT 'Collation fix handled by Lua migration system' AS info;
```

### 2. SQL Schema Dosyasına Not Ekle

`sql/release/consolidated_schema.sql` sonuna:

```sql
-- ============================================================================
-- COLLATION FIX NOTE
-- ============================================================================
-- Collation compatibility is handled automatically by the Lua migration system.
-- After this schema is applied, Migration.fixCollation() runs and:
--   1. Detects dominant collation from framework tables (players/users)
--   2. Converts all your_prefix_* tables to match
--   3. Prevents "Illegal mix of collations" errors in JOIN queries
--
-- Manual command: /yourprefix_fix_collation (if needed)
-- ============================================================================
```

### 3. Lua Fonksiyonları

`server/database/migration.lua` dosyasına ekle:

```lua
-- ============================================================================
-- COLLATION COMPATIBILITY
-- ============================================================================

local TABLE_PREFIX = 'your_prefix_'  -- << DEĞİŞTİR

--- Detect and return the dominant collation from framework tables
---@return string collation The detected collation (defaults to utf8mb4_general_ci)
---@return string charset The charset part of the collation
function Migration.detectDominantCollation()
    local collation = nil

    -- Priority order: players > users > player_vehicles > owned_vehicles
    local frameworkTables = { 'players', 'users', 'player_vehicles', 'owned_vehicles' }

    for _, tableName in ipairs(frameworkTables) do
        local result = MySQL.scalar.await([[
            SELECT TABLE_COLLATION
            FROM information_schema.TABLES
            WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = ?
            LIMIT 1
        ]], { tableName })

        if result and result ~= '' then
            collation = result
            break
        end
    end

    -- Fallback to general_ci
    if not collation or collation == '' then
        collation = 'utf8mb4_general_ci'
    end

    -- Extract charset (e.g., utf8mb4_unicode_ci -> utf8mb4)
    local charset = collation:match('^([^_]+)')
    if not charset then
        charset = 'utf8mb4'
    end

    return collation, charset
end

--- Check if any table has mismatched collation
---@return boolean needsFix True if collation fix is needed
---@return string targetCollation The target collation to use
function Migration.checkCollationMismatch()
    local targetCollation, _ = Migration.detectDominantCollation()

    local mismatchCount = MySQL.scalar.await([[
        SELECT COUNT(*)
        FROM information_schema.TABLES
        WHERE TABLE_SCHEMA = DATABASE()
        AND TABLE_NAME LIKE ?
        AND TABLE_COLLATION != ?
    ]], { TABLE_PREFIX .. '%', targetCollation })

    return (mismatchCount or 0) > 0, targetCollation
end

--- Fix collation for all tables to match framework tables
---@return boolean success
---@return number fixedCount Number of tables fixed
function Migration.fixCollation()
    local targetCollation, targetCharset = Migration.detectDominantCollation()

    Logger.info(string.format('Checking collation compatibility (target: %s)', targetCollation))

    local mismatchedTables = MySQL.query.await([[
        SELECT TABLE_NAME, TABLE_COLLATION
        FROM information_schema.TABLES
        WHERE TABLE_SCHEMA = DATABASE()
        AND TABLE_NAME LIKE ?
        AND TABLE_COLLATION != ?
    ]], { TABLE_PREFIX .. '%', targetCollation })

    if not mismatchedTables or #mismatchedTables == 0 then
        Logger.debug('All tables have correct collation')
        return true, 0
    end

    Logger.warn(string.format('Found %d table(s) with mismatched collation - fixing...', #mismatchedTables))

    local fixedCount = 0

    for _, row in ipairs(mismatchedTables) do
        local tableName = row.TABLE_NAME
        local currentCollation = row.TABLE_COLLATION

        local query = string.format(
            'ALTER TABLE `%s` CONVERT TO CHARACTER SET %s COLLATE %s',
            tableName, targetCharset, targetCollation
        )

        local ok, err = pcall(function()
            MySQL.query.await(query)
        end)

        if ok then
            fixedCount = fixedCount + 1
            Logger.info(string.format('  Fixed: %s (%s -> %s)', tableName, currentCollation, targetCollation))
        else
            Logger.error(string.format('  Failed to fix %s: %s', tableName, tostring(err)))
        end
    end

    Logger.info(string.format('Collation fix complete: %d/%d tables fixed', fixedCount, #mismatchedTables))
    return fixedCount == #mismatchedTables, fixedCount
end
```

### 4. Initialize Fonksiyonuna Entegrasyon

Her migration/install senaryosundan sonra `Migration.fixCollation()` çağır:

```lua
function Migration.initialize()
    -- ... existing code ...

    if fresh then
        -- Fresh install
        local success, err = Migration.runConsolidatedSchema()
        if not success then return false end

        Migration.fixCollation()  -- << EKLE
        return true
    end

    -- ... existing migration code ...

    -- After all migrations
    Migration.fixCollation()  -- << EKLE

    return true
end
```

### 5. Admin Komutları (Opsiyonel)

Debug için komutlar ekle:

```lua
RegisterCommand('yourprefix_fix_collation', function(source)
    local success, fixedCount = Migration.fixCollation()
    -- ... response handling ...
end, false)

RegisterCommand('yourprefix_collation_status', function(source)
    local targetCollation, targetCharset = Migration.detectDominantCollation()
    local needsFix, _ = Migration.checkCollationMismatch()
    -- ... response handling ...
end, false)
```

## Tespit Sırası

Framework tablolarından collation şu sırayla tespit edilir:

1. `players` (QBCore ana tablosu)
2. `users` (ESX ana tablosu)
3. `player_vehicles` (QBCore araç tablosu)
4. `owned_vehicles` (ESX araç tablosu)
5. **Fallback:** `utf8mb4_general_ci`

## Komutlar

| Komut | Açıklama |
|-------|----------|
| `/mechanic_collation_status` | Mevcut collation durumunu gösterir |
| `/mechanic_fix_collation` | Manuel collation düzeltme |

## Notlar

- Collation değişikliği veri kaybına **neden olmaz**
- `ALTER TABLE ... CONVERT TO` komutu tüm string kolonları dönüştürür
- İşlem büyük tablolarda birkaç saniye sürebilir
- Her resource restart'ta otomatik kontrol yapılır

## dusa_tablet İçin Uygulama

`dusa_tablet` için uygulamak için:

1. Table prefix: `ds_tablet_` (veya mevcut prefix)
2. Yukarıdaki adımları takip et
3. `consolidated_schema.sql` sonuna stored procedure ekle
4. Migration dosyası oluştur
5. `migration.lua`'ya Lua fonksiyonları ekle
6. `initialize()` fonksiyonuna `fixCollation()` çağrıları ekle
