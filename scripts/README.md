# DUSA Tablet - Mintlify Export

Bu klasör, Mintlify dokümantasyon reposuna copy-paste için hazır MDX dosyalarını içerir.

## Klasör Yapısı

```
dusa-tablet/
├── index.mdx                    # Ana sayfa
├── quickstart.mdx               # Hızlı başlangıç
├── navigation.json              # Navigasyon bilgisi
├── api-reference/
│   └── introduction.mdx         # API referansı
└── modules/
    ├── overview.mdx             # Modül genel bakış
    ├── shared/
    │   ├── config.mdx           # Konfigürasyon
    │   ├── constants.mdx        # Sabitler
    │   └── debug-bridge.mdx     # Debug bridge
    ├── server/
    │   ├── main.mdx             # Server main
    │   ├── database.mdx         # Database sistemi
    │   ├── debug.mdx            # Server debug
    │   └── apps.mdx             # Built-in uygulamalar
    ├── client/
    │   ├── main.mdx             # Client main
    │   └── debug.mdx            # Client debug
    └── telemetry/
        ├── overview.mdx         # Telemetry genel bakış
        ├── trace.mdx            # Trace engine
        ├── alerts.mdx           # Alert sistemi
        └── client.mdx           # Client tracing
```

## Entegrasyon Adımları

### 1. Klasörü Kopyala

`dusa-tablet/` klasörünü Mintlify docs root dizinine kopyala:

```
your-mintlify-docs/
├── docs.json
├── dusa-tablet/        <-- Buraya kopyala
│   ├── index.mdx
│   └── ...
└── other-docs/
```

### 2. Navigation Ekle

`navigation.json` dosyasındaki `navigation_snippet.groups` içeriğini `docs.json` dosyasına ekle:

```json
{
  "navigation": {
    "tabs": [
      {
        "tab": "DUSA Tablet",
        "groups": [
          // navigation.json'dan groups içeriğini buraya yapıştır
        ]
      }
    ]
  }
}
```

### 3. Yol Ayarlaması

Eğer farklı bir dizine koyarsan, sayfa yollarını güncelle:

```json
// Örnek: products/ altına koyarsan
"pages": [
  "products/dusa-tablet/index",
  "products/dusa-tablet/quickstart"
]
```

### 4. Görseller (Opsiyonel)

Hero image ve diğer görseller için:
- `images/` klasörüne görselleri ekle
- MDX dosyalarındaki image referanslarını güncelle

## Dosya Listesi (17 sayfa)

| Dosya | Başlık | Açıklama |
|-------|--------|----------|
| index.mdx | Introduction | Ana sayfa |
| quickstart.mdx | Quickstart | Kurulum rehberi |
| modules/overview.mdx | Module Overview | Modül yapısı |
| modules/shared/config.mdx | Configuration | Konfigürasyon |
| modules/shared/constants.mdx | Constants | Sabitler |
| modules/shared/debug-bridge.mdx | Debug Bridge | Cross-resource logging |
| modules/server/main.mdx | Server Main | Server entry point |
| modules/server/database.mdx | Database System | Migration sistemi |
| modules/server/debug.mdx | Server Debug | Server logging |
| modules/server/apps.mdx | Server Apps | Mail, News, Camera, Messages |
| modules/client/main.mdx | Client Main | Client entry point |
| modules/client/debug.mdx | Client Debug | Client logging |
| modules/telemetry/overview.mdx | Telemetry Overview | Telemetry sistemi |
| modules/telemetry/trace.mdx | Trace Engine | Tracing API |
| modules/telemetry/alerts.mdx | Alerts System | Discord webhooks |
| modules/telemetry/client.mdx | Client Tracing | Client-side tracing |
| api-reference/introduction.mdx | API Reference | Exports ve callbacks |

## Frontmatter Standartları

Tüm MDX dosyaları Mintlify standartlarına uygun frontmatter içerir:

```yaml
---
title: "Sayfa Başlığı"
description: "SEO ve navigasyon için kısa açıklama"
---
```

## Notlar

- Tüm internal linkler relative path kullanır
- Tüm code block'lar language tag içerir
- Image'lar alt text içerir
- Second-person voice ("you") kullanılır
