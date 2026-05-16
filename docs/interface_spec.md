# TOLUN — AKILLI EV & AKVARYUM OTOMASYON PLATFORMU
## Interface Specification (BLE Protokolü)

> **Durum:** 🧪 Test Aşaması  
> **Baz Kaynak:** BLE GATT Specification  
> **Son Güncelleme:** 2026-05-14

Bu doküman, TolunControl mobil uygulaması ile Tolun platformu cihazları (AquaFeeder, AquaLighting, WallLighting, HomeLighting) arasındaki BLE haberleşme protokolünü tanımlar.

---

## 1. Genel Kurallar

- Tüm haberleşme JSON formatında yapılır
- Tüm mesajlar "type" veya "cmd" alanı içerir
- Tüm cevaplar "status" alanı içerir
- Timestamp: UNIX timestamp (saniye cinsinden) gönderilir, yanıtlarda okunabilir format (`"YYYY-MM-DD HH:MM:SS"`) döndürülür

---

## 2. Haberleşme Kanalları

### BLE (Bluetooth)
- Lokal kontrol
- Düşük gecikme

### MQTT (Wi-Fi)
- Uzaktan kontrol
- Gerçek zamanlı veri akışı

---

## 3. Komut Yapısı (Command Format)

Tüm komutlar aşağıdaki formatta olmalıdır:

```json
{
  "cmd": "<command_name>",
  "data": {
    "param1": "value"
  }
}
```

---

## 4. Desteklenen Komutlar

### 1) Manuel Yemleme

```json
{
  "cmd": "feed",
  "data": {
    "portion": 2,
    "portion_interval": 2000
  }
}
```

**Alan Açıklamaları:**
- `portion`: Porsiyon adedi (1–10)
- `portion_interval`: Porsiyon arası bekleme süresi (ms), opsiyonel (varsayılan: 0)

---

### 2) Otomatik Yemleme Ayarı

```json
{
  "cmd": "set_schedule",
  "data": {
    "kind": 0,
    "hour": 8,
    "minute": 30,
    "portion": 2,
    "portion_interval": 60000,
    "active": true,
    "label": "Sabah"
  }
}
```

**Alan Açıklamaları (Feeder Planı):**
- `kind`: `0` = feeder planı
- `hour`: Saat (0–23)
- `minute`: Dakika (0–59)
- `portion`: Porsiyon adedi (1–10)
- `portion_interval`: Porsiyon arası bekleme süresi (ms, 0–300000 / max 5 dk), opsiyonel
- `active`: Plan aktif mi (`true`/`false`), opsiyonel (varsayılan `true`)
- `label`: Plan adı, opsiyonel (max 16 karakter)

**Aydınlatma Planı (kind=1):**

```json
{
  "cmd": "set_schedule",
  "data": {
    "kind": 1,
    "hour": 8,
    "minute": 0,
    "end_hour": 22,
    "end_minute": 0,
    "r": 221,
    "g": 127,
    "b": 44,
    "brightness": 80,
    "active": true,
    "label": "Gündüz"
  }
}
```

**Ek Alan Açıklamaları (Lighting Planı):**
- `end_hour`, `end_minute`: Plan bitiş zamanı
- `r`, `g`, `b`: Plan tetiklendiğinde uygulanacak renk (0–255)
- `brightness`: Plan parlaklığı (0–100); `0` ise renk kaydı tutulmaz
- Gece geçişli aralık desteklenir (`end < start` → ertesi gün)

---

### 3) Otomatik Yemleme Planı Güncelleme

```json
{
  "cmd": "update_schedule",
  "data": {
    "old_hour": 8,
    "old_minute": 30,
    "hour": 9,
    "minute": 0,
    "portion": 3,
    "portion_interval": 120000
  }
}
```

---

### 4) Otomatik Yemleme Planı Silme

```json
{
  "cmd": "remove_schedule",
  "data": {
    "hour": 8,
    "minute": 30
  }
}
```

---

### 5) Tüm Otomatik Yemleme Planlarını Silme

```json
{
  "cmd": "clear_schedules"
}
```

---

### 6) Durum Sorgulama

```json
{
  "cmd": "get_status"
}
```

---

### 7) Sistem Saati Ayarlama

```json
{
  "cmd": "set_time",
  "data": {
    "timestamp": 1745481600,
    "tz_offset": 180
  }
}
```

> **Not:** `set_time` **Device Time Service** (`0x1847`) üzerinden gönderilir; Feeding Service'in komut karakteristiğinden değil. Yazma adresi `6E300001`, yanıt notify adresi `6E300002`. Bağlantı başına bir kez gönderilmesi yeterlidir.

---

### 8) Sistem Sıfırlama

```json
{
  "cmd": "reset"
}
```

---

### 9) Plan Sorgulama (Per-Index)

```json
{
  "cmd": "get_schedule",
  "data": { "index": 0 }
}
```

**Alan Açıklamaları:**
- `index`: 0'dan `schedule_count - 1`'e kadar; `get_status` yanıtındaki `schedule_count` kadar tekrarlanır

---

### 10) Besleme Loglarını Temizle

```json
{
  "cmd": "clear_logs"
}
```

---

### 11) Besleme Log Kaydı Sorgulama (Per-Index)

```json
{
  "cmd": "get_feed_log_entry",
  "data": { "index": 0 }
}
```

**Alan Açıklamaları:**
- `index`: 0'dan `feed_log_count - 1`'e kadar; `get_status` yanıtındaki `feed_log_count` kadar tekrarlanır
- Her yanıt ~160 B

---

### 12) Aydınlatma Log Kaydı Sorgulama (Per-Index)

```json
{
  "cmd": "get_lighting_log_entry",
  "data": { "index": 0 }
}
```

**Alan Açıklamaları:**
- `index`: 0'dan `lighting_log_count - 1`'e kadar; `get_status` yanıtındaki `lighting_log_count` kadar tekrarlanır
- Her yanıt ~145 B

---

### 13) Aydınlatma Kontrolü

```json
{ "cmd": "lighting_on" }
```

```json
{ "cmd": "lighting_off" }
```

---

### 14) Parlaklık Ayarı

```json
{
  "cmd": "set_brightness",
  "data": {
    "brightness": 75
  }
}
```

**Alan Açıklamaları:**
- `brightness`: Parlaklık seviyesi (0–100)

---

### 15) Renk Ayarı (RGB)

```json
{
  "cmd": "set_color",
  "data": {
    "r": 0,
    "g": 120,
    "b": 255
  }
}
```

**Alan Açıklamaları:**
- `r`, `g`, `b`: Kırmızı / Yeşil / Mavi kanalları (0–255)

> Donanım WS2812B (3 kanal); RGBW desteği yok.

---

### 16) Atomik Renk + Parlaklık Ayarı

```json
{
  "cmd": "set_light",
  "data": {
    "r": 221,
    "g": 127,
    "b": 44,
    "brightness": 80
  }
}
```

**Alan Açıklamaları:**
- `r`, `g`, `b`: Renk kanalları (0–255)
- `brightness`: Parlaklık (0–100)

> `set_color` + `set_brightness` ardışık çağrılarına alternatif; tek atomik işlemde uygular. Plan tetiklemelerinde mobil bu komutu kullanır.

---

## 5. Response (Cevap) Yapısı

```json
{
  "type": "response",
  "cmd": "feed",
  "status": "success",
  "data": {},
  "time": "2025-04-24 08:30:00"
}
```

**Status Değerleri:**
- `"success"` — Komut başarıyla gerçekleştirildi
- `"error"` — Komut işleme hatası

---

## 6. Event (Asenkron Mesajlar)

Cihaz, komut olmadan da veri gönderebilir.

### Yemleme Sonucu

```json
{
  "type": "feed_result",
  "mode": "manual",
  "portion": 2,
  "status": "success",
  "time": "2025-04-24 08:30:00"
}
```

**Alan Açıklamaları:**
- `mode`: `"manual"` (manuel) veya `"scheduled"` (otomatik)
- `portion`: Verilen porsiyon adedi
- `status`: `"success"` veya `"error"`
- `time`: Yemleme gerçekleştiği zaman (okunabilir format)

---

### Hata Bildirimi

```json
{
  "type": "error",
  "code": "MOTOR_ERROR",
  "message": "Motor stuck",
  "time": "2025-04-24 08:30:00"
}
```

**Hata Kodları:**
- `MOTOR_ERROR` — Motor hatası
- `INVALID_PORTION` — Geçersiz porsiyon değeri
- `TIMEOUT` — İşlem timeout
- `INVALID_STATE` — Sistem uygun durumda değil

---

## 7. State (Durum) Yapısı

`get_status` komutu veya cihaz tarafından tetiklenen state değişikliklerinde `6E400004` (Read + Notify) üzerinden döner.

```json
{
  "type": "state",
  "state": "IDLE",
  "time": "2026-05-16 10:00:00",
  "led_on": false,
  "schedule_count": 2,
  "feed_log_count": 5,
  "lighting_log_count": 3,
  "first_conn_ts": 1747200000,
  "total_uptime_sec": 86400
}
```

### Alan Açıklamaları:
- `state`: Cihaz durumu — `"IDLE"`, `"RUNNING"`, `"ERROR"`
- `time`: Cihazın yerel saati (`YYYY-MM-DD HH:MM:SS`)
- `led_on`: Aydınlatma şeridinin açık/kapalı durumu
- `schedule_count`: NVS'deki kayıtlı plan sayısı (0–5); planlar `get_schedule` ile ayrıca çekilir
- `feed_log_count`: Günlük besleme log kayıt sayısı (0–20); kayıtlar `get_feed_log_entry` ile ayrıca çekilir
- `lighting_log_count`: Günlük aydınlatma log kayıt sayısı; kayıtlar `get_lighting_log_entry` ile ayrıca çekilir
- `first_conn_ts`: Cihazın ilk BT pairing unix timestamp'i (`0` = henüz bağlanmamış). NVS'te kalıcı, bir daha değişmez.
- `total_uptime_sec`: Tüm güç çevrimleri boyunca biriken toplam çalışma süresi (saniye). NVS'te kalıcı, her 5 dakikada bir flush'lanır.

> `schedules` array'i v1.16.0'dan itibaren status payload'undan çıkarıldı.
>
> `fw_version` ve `hw_version` v1.20.0'dan itibaren status payload'undan çıkarıldı. Mobil uygulama bu değerleri DIS karakteristiklerinden okur: FW Revision (`0x2A26` = `FIRMWARE_VERSION`) ve HW Revision (`0x2A27` = `HARDWARE_VERSION`). MTU bütçesi için ~30 B tasarruf.
>
> `first_conn_ts` ve `total_uptime_sec` v1.19.0'da eklendi — cihazın yaşam istatistikleri (servis/satış sonrası takibi için).

---

## 8. MQTT Topic Yapısı

> Wi-Fi / MQTT desteği planlanmaktadır; henüz firmware veya mobil uygulama kodu mevcut değildir.

| Yön | Topic | Açıklama |
|---|---|---|
| Mobil → Cihaz | `tolun/{device_id}/feed/command` | Besleme komutu |
| Cihaz → Mobil | `tolun/{device_id}/feed/result` | Besleme sonucu event |
| Mobil → Cihaz | `tolun/{device_id}/lighting/command` | Aydınlatma komutu |
| Cihaz → Mobil | `tolun/{device_id}/lighting/result` | Aydınlatma sonucu event |
| Cihaz → Mobil | `tolun/{device_id}/device/state` | Anlık durum değişikliği |
| Cihaz → Mobil | `tolun/{device_id}/device/error` | Hata bildirimi |
| Cihaz → Mobil | `tolun/{device_id}/log/feed` | Besleme log güncellemesi |
| Cihaz → Mobil | `tolun/{device_id}/log/lighting` | Aydınlatma log güncellemesi |

`{device_id}`: Cihazın BLE adının son 4 karakteri (örn. `AquaFeeder-1A2B` → `1A2B`).

---

## 9. Veri Doğrulama Kuralları

| Alan | Geçerli Aralık | Notlar |
|------|---|---|
| `portion` | 1 – 10 | Manuel ve otomatik yemleme için |
| `portion_interval` | 0 – 300000 | Milisaniye cinsinden (maksimum 5 dakika) |
| `hour` | 0 – 23 | 24 saat formatı |
| `minute` | 0 – 59 | |
| `timestamp` | Geçerli UNIX timestamp | Saniye cinsinden |
| `tz_offset` | -720 – 840 | Dakika cinsinden UTC farkı (örn. UTC+3 = 180) |
| Maksimum planlar | 5 | Cihazda saklanabilecek otomatik plan sayısı |
| Minimum yemleme aralığı | 60 saniye | Art arda iki feed komutu arasındaki zorunlu bekleme |
| Günlük porsiyon limiti | 30 | Cihazın bir günde kabul ettiği maksimum toplam porsiyon |

---

## 10. Timeout ve Retry

- Komut sonrası maksimum 3 saniye içinde response beklenir
- Response gelmezse mobil taraf 2 kez retry yapabilir
- Toplam deneme süresi: 9 saniye (3 × 3 sn)

---

## 11. Genel Kurallar

- Tüm komutlar idempotent olmalı (tekrar çalıştırılırsa aynı sonuç vermelidir)
- Cihaz her zaman response dönmeli (başarı veya hata)
- Kritik işlemler için ACK mekanizması kullanılmalı
- Aynı cihazda aynı saatte iki plan olamaz
- Yemleme işlemi sırasında yeni yemleme komutu engellenir

---

## 12. Örnek Akış

### Manuel Yemleme Akışı

```
1. Mobil → Cihaz: {"cmd":"feed","data":{"portion":2}}
2. Cihaz: Command doğrulama (portion 1-10, state kontrol)
3. Cihaz: Motor control ile yemleme başlatma
4. Cihaz → Mobil: {"type":"response","cmd":"feed","status":"success","time":"..."}
5. Cihaz → Mobil: {"type":"feed_result","mode":"manual","portion":2,"status":"success","time":"..."}
```

### Otomatik Yemleme Akışı

```
1. Mobil → Cihaz: {"cmd":"set_schedule","data":{"hour":8,"minute":0,"portion":2}}
2. Cihaz: Plan doğrulama, NVS'e kayıt
3. Cihaz → Mobil: {"type":"response","cmd":"set_schedule","status":"success","time":"..."}
4. [Zamanı gelince] Cihaz: Otomatik yemleme tetikleme
5. Cihaz → Mobil: {"type":"feed_result","mode":"scheduled","portion":2,"status":"success","time":"..."}
```

---

## 13. Sonuç

Bu interface specification dokümanı, tüm sistem bileşenlerinin ortak bir dil kullanmasını sağlar. Bu yapı sayesinde mobil uygulama ve embedded sistem bağımsız geliştirilebilir ve sorunsuz entegre olur.