# AI Internal Documentation

> **Bu klasör sadece AI asistanlar içindir. Kullanıcıya gösterilmez.**

## Dosyalar

| Dosya | İçerik |
|-------|--------|
| `CODEBASE_MAP.md` | Tüm dosyaların escrow durumu, global table'lar, callback'ler |
| `ISSUE_RESOLUTION.md` | Sorun çözüm protokolü, kategori bazlı çözümler |
| `DATABASE_SCHEMA.md` | Tüm tablo şemaları, adapter query örnekleri |
| `APP_DEVELOPMENT.md` | Yeni app oluşturma rehberi, iframe entegrasyonu, registration |

## Escrow Kuralları

### OPEN Dosyalar (Değiştirilebilir)

```
config/config.lua
shared/locales/*.lua
server/apps/messages/adapters/*.lua
telemetry/config.lua
```

### LOCKED Dosyalar (Değiştirilemez)

Yukarıdakiler dışındaki tüm `.lua` dosyaları.

## AI Protokolü

1. **Sorun tespit et** → CODEBASE_MAP.md'den dosya bul
2. **Escrow kontrolü** → OPEN ise fix öner, LOCKED ise raporla
3. **Çözüm uygula** → ISSUE_RESOLUTION.md'deki template'leri kullan
4. **LOCKED dosya** → Kullanıcıya bildir, workaround öner

## Yanıt Template'i (LOCKED Dosya)

```markdown
## Tespit Edilen Sorun

**Dosya**: `{path}`
**Durum**: LOCKED (Escrowed)

Bu dosya FiveM escrow koruması altında ve değiştirilemez.

### Geçici Çözüm
{workaround}

### Kalıcı Çözüm
Geliştirici ile iletişime geçilmeli.
```

## Yanıt Template'i (OPEN Dosya)

```markdown
## Çözüm

**Dosya**: `{path}`
**Değişiklik**:

\`\`\`lua
-- Eski
{old_code}

-- Yeni
{new_code}
\`\`\`
```
