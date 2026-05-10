# Tolun Firmware — Bellek Optimizasyon Kılavuzu

## Özet

Kod analizi sonucunda önemli bir bulgu ortaya çıktı: **NVS (flash) depolaması zaten binary formatta yapılıyor; JSON hiçbir zaman NVS'e yazılmıyor.** Asıl bellek baskısı RAM'dedir — özellikle `get_status` komutunun oluşturduğu büyük JSON buffer ve FreeRTOS kuyruklarındaki veri yapısı boyutları.

Bu belge mevcut durumu analiz eder, sorunları öncelik sırasıyla listeler ve uygulanabilir çözümler sunar.

---

## 1. Mevcut Bellek Mimarisi

### 1.1 NVS (Flash) — Durum: İyi

NVS'de JSON **tutulmaz**. Tüm veriler binary olarak saklanır:

| Veri | Anahtar(lar) | Format | Toplam Boyut |
|------|-------------|--------|--------------|
| Feed log (maks. 20 kayıt) | `flog_0` … `flog_19` | `uint64_t` per entry | 160 byte |
| Lighting log (maks. 20 kayıt) | `llog_0` … `llog_19` | `uint64_t` per entry | 160 byte |
| Schedule (maks. 5 kayıt) | `sched_0` … `sched_4` | binary blob | ~185 byte |
| Sayaçlar, tarihler, zaman | çeşitli | u8 / u32 / i32 | ~40 byte |
| **Toplam** | | | **~545 byte** |

`feed_log_entry_t` ve `lighting_log_entry_t` her biri 8 byte'lık bir `uint64_t` olarak kaydediliyor. Bu format değiştirilmemeli.

### 1.2 RAM — Darboğazlar

ESP32-S3'ün kullanılabilir SRAM'ı 320 KB'tır. Aşağıdaki statik buffer'lar ve kuyruklar büyük yer kaplar:

| Bileşen | Konum | Boyut | Açıklama |
|---------|-------|-------|----------|
| `status_buf[2048]` | `ble_service.c:296` | 2048 byte | `get_status` JSON çıktısı |
| `resp_queue` (10 × `outgoing_msg_t`) | `main.c` | 2600 byte | 10 item × 260 byte |
| `s_rx_buf[512]` | `ble_service.c:125` | 512 byte | BLE paket birleştirme |
| `cmd_queue` (10 × `command_t`) | `main.c` | ~1200 byte | 10 item × ~120 byte |
| NimBLE host task stack | ESP-IDF | 12288 byte | Değiştirilemez |
| Uygulama görev stack'leri | `system_config.h` | ~25600 byte | 9 görev |

---

## 2. Tespit Edilen Sorunlar

### Sorun 1 — `get_status` JSON Buffer Taşması (Kritik)

`get_status` komutu yanıtı içinde şu anda şunlar yer alıyor:
- 5 schedule (her biri JSON'da ~80 byte) → ~400 byte
- 20 feed log kaydı (her biri ~60 byte) → ~1200 byte
- 20 lighting log kaydı (her biri ~55 byte) → ~1100 byte
- Genel JSON overhead → ~100 byte

**Toplam: ~2800 byte** — ancak `status_buf` yalnızca 2048 byte.

Gün içinde log sayısı arttıkça `cJSON_PrintPreallocated()` başarısız olur ve cevap boş gider veya kesik kalır. Bu en kritik sorundur.

### Sorun 2 — `outgoing_msg_t` Queue Şişkinliği (Orta)

```c
typedef struct {
    msg_type_t type;     // 4 byte
    char       payload[256]; // 256 byte — çoğu mesaj için fazla
} outgoing_msg_t;        // 260 byte
```

`resp_queue` 10 item taşır: **10 × 260 = 2600 byte**. Yanıtların büyük çoğunluğu 128 byte'ın altında kalır.

### Sorun 3 — `command_t` Struct Şişkinliği (Düşük)

0–23 aralığında değer taşıyan alanlar için `int` (4 byte) kullanılıyor:

```c
// utils/json_parser.h — şu an
int kind;        // 0 veya 1 → 4 byte harcıyor
int portion;     // 1–10   → 4 byte harcıyor
int hour;        // 0–23   → 4 byte harcıyor
int minute;      // 0–59   → 4 byte harcıyor
int end_hour;    // 0–23   → 4 byte harcıyor
int end_minute;  // 0–59   → 4 byte harcıyor
int old_hour;    // 0–23   → 4 byte harcıyor
int old_minute;  // 0–59   → 4 byte harcıyor
```

Bu 8 alan birlikte **32 byte** yer tutuyor; `uint8_t` kullansak 8 byte olur.

### Sorun 4 — `schedule_entry_t` Alignment Kaybı (Düşük)

```c
// scheduler.h — şu an
sched_kind_t kind;  // enum = int = 4 byte (0 veya 1 için)
bool         active; // 1 byte + derleyici 3 byte padding ekler
```

Tek entry'de ~6 byte boşa gidiyor. 5 entry için 30 byte kayıp.

---

## 3. Optimizasyon Önerileri

### Öncelik 1 — `get_status` Yanıtını Parçala

**Bu değişiklik en kritik olanı ve en büyük kazancı sağlar.**

`get_status` komutu artık yalnızca anlık durum ve schedule listesini döndürmeli. Feed log ve lighting log verileri ayrı komutlarla istenecek.

#### Yeni Komut Protokolü

```json
// get_status — sadece durum + schedule
// İstek:
{"cmd": "get_status"}

// Cevap (artık log yok):
{
  "type": "state",
  "state": "IDLE",
  "led_on": false,
  "fw_version": "1.13.0",
  "hw_version": "1.0",
  "time": "2026-05-10 10:00:00",
  "schedules": [
    {"index": 0, "kind": 0, "hour": 8, "minute": 30, "active": true,
     "portion": 2, "portion_interval": 60000, "label": "Sabah"}
  ]
}
```

```json
// get_feed_log — feed geçmişi
// İstek:
{"cmd": "get_feed_log"}

// Cevap:
{
  "type": "feed_log",
  "data": [
    {"ts": 1746871200, "portion": 2, "mode": "manual", "status": "success"},
    {"ts": 1746857200, "portion": 1, "mode": "scheduled", "status": "success"}
  ]
}
```

```json
// get_lighting_log — aydınlatma geçmişi
// İstek:
{"cmd": "get_lighting_log"}

// Cevap:
{
  "type": "lighting_log",
  "data": [
    {"ts": 1746860000, "duration_s": 3600, "is_scheduled": 1}
  ]
}
```

#### Firmware Değişiklikleri

**`Firmware/main/utils/json_parser.c`** — `json_build_state()` fonksiyonundan `feed_log` ve `lighting_log` dizilerini kaldır. Ayrı `json_build_feed_log()` ve `json_build_lighting_log()` fonksiyonları ekle.

**`Firmware/main/core/command_handler.c`** — `handle_get_status()` fonksiyonu sadece state + schedule dönsün. Yeni `handle_get_feed_log()` ve `handle_get_lighting_log()` handler'ları ekle.

**`Firmware/main/comm/ble_service.c`** — `status_buf[2048]` boyutunu `status_buf[512]`'ye düşür.

**`Firmware/main/config/system_config.h`** — Yeni NVS anahtar sabitleri gerekmez; sadece yeni komut adları eklenir.

#### Kazanç

| Kaynak | Önceki | Sonraki | Tasarruf |
|--------|--------|---------|----------|
| `status_buf` | 2048 byte | 512 byte | **1536 byte** |
| `outgoing_msg_t.payload` (izleyen adımla) | 256 byte | 128 byte | **1280 byte** |

#### Mobil Uygulama Değişiklikleri

**`Mobile App/src/core/ble/BleService.ts`** — Bağlantı sonrası `get_status` yerine sırasıyla `get_status`, `get_feed_log`, `get_lighting_log` çağır.

**`Mobile App/src/data/AppStateContext.tsx`** — `syncDeviceLogs()` fonksiyonu mevcut; sadece yeni komut adını kullan.

---

### Öncelik 2 — `command_t` Struct'ını Sıkıştır

**`Firmware/main/utils/json_parser.h`** dosyasında:

```c
// ÖNCE (~120 byte)
typedef struct {
    char     cmd[20];
    int      kind;
    int      portion;
    int      hour;
    int      minute;
    int      end_hour;
    int      end_minute;
    int      old_hour;
    int      old_minute;
    bool     active;
    bool     is_scheduled;
    char     feed_type[12];
    int      portion_interval;
    uint32_t timestamp;
    int32_t  tz_offset;
    uint8_t  color_r, color_g, color_b;
    uint8_t  brightness;
    char     label[16];
} command_t;

// SONRA (~80 byte)
typedef struct {
    char     cmd[20];
    uint8_t  kind;            // 0 veya 1
    uint8_t  portion;         // 1–10
    uint8_t  hour;            // 0–23
    uint8_t  minute;          // 0–59
    uint8_t  end_hour;        // 0–23
    uint8_t  end_minute;      // 0–59
    uint8_t  old_hour;        // 0–23
    uint8_t  old_minute;      // 0–59
    bool     active;
    bool     is_scheduled;
    char     feed_type[12];
    uint32_t portion_interval; // 0–300000, uint32_t gerekli
    uint32_t timestamp;
    int32_t  tz_offset;
    uint8_t  color_r, color_g, color_b;
    uint8_t  brightness;
    char     label[16];
} command_t;
```

**`Firmware/main/utils/json_parser.c`** içindeki tüm `cmd.kind`, `cmd.portion` vb. atamaları `(int)` → `(uint8_t)` cast'i ile güncelle. Doğrulama mantığı değişmez.

**Kazanç:** `cmd_queue`: 10 × 40 byte = **400 byte** tasarruf.

---

### Öncelik 3 — `schedule_entry_t` NVS Formatını Sıkıştır

Bu değişiklik NVS versiyonunu artırır — cihazdaki mevcut schedule verisi silinir, kullanıcı schedule'larını yeniden girmek zorunda kalır. Dikkatli uygulanmalı.

**`Firmware/main/scheduler/scheduler.h`** dosyasına packed struct ekle:

```c
// NVS'e yazılan kompakt format (sadece depolama için)
typedef struct __attribute__((packed)) {
    char     label[12];       // 16 → 12 byte (12 karakter yeterli)
    uint8_t  kind;            // sched_kind_t enum → 1 byte
    uint8_t  flags;           // bit 0 = active, diğerleri rezerve
    uint8_t  start_hour;
    uint8_t  start_minute;
    uint8_t  end_hour;
    uint8_t  end_minute;
    uint8_t  portion;
    uint8_t  _pad;            // gelecek kullanım
    uint32_t portion_interval;
} sched_entry_packed_t;       // 24 byte (37'den 24'e)
```

**`Firmware/main/scheduler/scheduler.c`** içinde NVS okuma/yazma fonksiyonları `sched_entry_packed_t` kullanacak şekilde güncellenir. `schedule_entry_t` (RAM'deki büyük struct) aynen kalır; yalnızca NVS serialize/deserialize adımları dönüşüm yapar.

**`Firmware/main/config/system_config.h`** içinde `SCHED_NVS_VERSION` değerini artır:

```c
#define SCHED_NVS_VERSION  5   // 4'ten 5'e
```

**Kazanç:** NVS: 5 × 13 byte = **65 byte** tasarruf.

---

### Öncelik 4 — `outgoing_msg_t` Payload'ı Küçült (Öncelik 1'e Bağlı)

Öncelik 1 uygulandıktan sonra hiçbir yanıt 128 byte'ı geçmeyecek.

**`Firmware/main/utils/json_parser.h`** dosyasında:

```c
typedef struct {
    msg_type_t type;
    char       payload[128];  // 256 → 128
} outgoing_msg_t;
```

**`Firmware/main/config/system_config.h`** içindeki `RESP_PAYLOAD_SIZE` (varsa) sabitini de güncelle.

**Kazanç:** `resp_queue`: 10 × 128 byte = **1280 byte** tasarruf.

---

## 4. BLE Protokolü — JSON Kullanmaya Devam Edin

BLE üzerinden JSON kullanmak doğru bir karardır. Alternatifler (MessagePack, Protobuf, özel binary) küçük veri boyutları için anlamsız karmaşıklık ekler ve mobil tarafta uyumsuzluk riski yaratır.

**Gerekli tek protokol değişikliği:** `get_status` artık log döndürmez. Mobil uygulama bağlantı sonrası üç ayrı komut gönderir.

JSON alan isimlerini kısaltmak (örn. `"portion"` → `"p"`) teorik olarak %10–15 boyut kazancı sağlar ama okunabilirliği yok eder ve debug'ı zorlaştırır. Önerilmez.

---

## 5. Özet — Kazanç Tablosu

| Değişiklik | RAM Tasarrufu | NVS Tasarrufu | Zorluk | Öncelik |
|-----------|---------------|---------------|--------|---------|
| `get_status` yanıtını parçala | ~1536 byte | — | Orta | 1 |
| `command_t` struct sıkıştır | ~400 byte | — | Kolay | 2 |
| `schedule_entry_t` NVS formatı | — | ~65 byte | Orta | 3 |
| `outgoing_msg_t` payload küçült | ~1280 byte | — | Kolay | 4 (1 sonrası) |
| **Toplam** | **~3216 byte** | **~65 byte** | | |

---

## 6. Etkilenen Dosyalar

### Firmware

| Dosya | Değişiklik |
|-------|-----------|
| [main/utils/json_parser.h](../Firmware/main/utils/json_parser.h) | `command_t` struct tipler, `outgoing_msg_t` payload boyutu |
| [main/utils/json_parser.c](../Firmware/main/utils/json_parser.c) | `json_build_state()` log kaldır, yeni log builder fonksiyonlar |
| [main/comm/ble_service.c](../Firmware/main/comm/ble_service.c) | `status_buf` boyutu, yeni komut dispatch |
| [main/core/command_handler.c](../Firmware/main/core/command_handler.c) | `get_feed_log`, `get_lighting_log` handler'ları |
| [main/scheduler/scheduler.h](../Firmware/main/scheduler/scheduler.h) | `sched_entry_packed_t` struct |
| [main/scheduler/scheduler.c](../Firmware/main/scheduler/scheduler.c) | NVS serialize/deserialize dönüşüm |
| [main/config/system_config.h](../Firmware/main/config/system_config.h) | `SCHED_NVS_VERSION` artır |

### Mobile App

| Dosya | Değişiklik |
|-------|-----------|
| [src/core/ble/BleService.ts](../Mobile%20App/src/core/ble/BleService.ts) | Bağlantı sonrası üç ayrı komut gönder |
| [src/data/AppStateContext.tsx](../Mobile%20App/src/data/AppStateContext.tsx) | Log sync komutu güncelle |

---

## 7. Doğrulama

### Heap Ölçümü

Firmware'e şu logu ekle, bağlantı öncesi ve sonrası karşılaştır:

```c
ESP_LOGI("MEM", "Free heap: %lu bytes",
         (unsigned long)heap_caps_get_free_size(MALLOC_CAP_DEFAULT));
```

### Test Akışı

1. `idf.py -p COM9 flash monitor` ile cihazı yükle ve izle
2. Mobil uygulamayı bağla — heap değerleri logda görünmeli
3. `get_status` komutunu gönder → yanıtta `feed_log` ve `lighting_log` **olmayacak**
4. `get_feed_log` komutunu gönder → feed log gelecek
5. `get_lighting_log` komutunu gönder → lighting log gelecek
6. Schedule ekle, sil, güncelle → NVS versiyonu 5 ile doğru çalışmalı
7. Cihazı yeniden başlat → schedule'lar ve zaman NVS'den doğru yüklenmeli
8. Mobil uygulamada loglar ve planlar eskisi gibi görünmeli
