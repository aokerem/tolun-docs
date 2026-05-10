# AKVARYUM OTOMATİK YEMLEME SİSTEMİ
## Interface Specification (BLE Protokolü)

> **Durum:** 🧪 Test Aşaması  
> **Baz Kaynak:** BLE GATT Specification  
> **Son Güncelleme:** 2026-04-25

Bu doküman, mobil uygulama (React Native) ve ESP32 firmware arasındaki BLE haberleşme protokolünü tanımlar.

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
    "portion_interval": 2000,
    "feed_type": "quick"
  }
}
```

**Alan Açıklamaları:**
- `portion`: Porsiyon adedi (1–10)
- `portion_interval`: Porsiyon arası bekleme süresi (ms), opsiyonel (varsayılan: 0)
- `feed_type`: `"quick"` (hızlı) veya `"scheduled"` (planlı), opsiyonel

---

### 2) Otomatik Yemleme Ayarı

```json
{
  "cmd": "set_schedule",
  "data": {
    "hour": 8,
    "minute": 30,
    "portion": 2,
    "portion_interval": 60000
  }
}
```

**Alan Açıklamaları:**
- `hour`: Saat (0–23)
- `minute`: Dakika (0–59)
- `portion`: Porsiyon adedi (1–10)
- `portion_interval`: Porsiyon arası bekleme süresi (ms, 0–300000 / max 5 dk), opsiyonel

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

### 9) Besleme Loglarını Temizle

```json
{
  "cmd": "clear_logs"
}
```

---

### 10) Aydınlatma Kontrolü

```json
{ "cmd": "lighting_on" }
```

```json
{ "cmd": "lighting_off" }
```

---

### 11) Parlaklık Ayarı

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

### 12) Renk Ayarı (RGB / RGBW)

```json
{
  "cmd": "set_color",
  "data": {
    "r": 0,
    "g": 120,
    "b": 255,
    "w": 80
  }
}
```

**Alan Açıklamaları:**
- `r`, `g`, `b`: Kırmızı / Yeşil / Mavi kanalları (0–255)
- `w`: Beyaz kanal (0–255), opsiyonel — yalnızca RGBW cihazlarda dolu

---

### 13) Hazır Mod

```json
{
  "cmd": "set_preset",
  "data": {
    "preset": "daylight"
  }
}
```

**Geçerli Preset Değerleri:**
- `daylight` — Gün ışığı (sıcak beyaz, tam parlaklık)
- `night` — Gece (soluk mavi, düşük parlaklık)
- `plant` — Bitki modu (mavi + kırmızı spektrum)
- `moonlight` — Ay ışığı (çok düşük parlaklık, serin mavi)
- `sunset` — Gün batımı (sıcak turuncu)
- `ocean` — Okyanus (derin mavi)

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

```json
{
  "type": "state",
  "state": "IDLE"
}
```

### State Değerleri:
- IDLE
- RUNNING
- ERROR

---

## 8. MQTT Topic Yapısı

- aquarium/feed/set
- aquarium/feed/result
- aquarium/device/state
- aquarium/device/error

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
| Maksimum planlar | 8 | Cihazda saklanabilecek otomatik yemleme planı sayısı |
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