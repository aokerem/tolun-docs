# AKVARYUM OTOMATİK YEMLEME SİSTEMİ
## BLE GATT SPECIFICATION

---

## 1. Amaç
Bu doküman, mobil uygulama ile cihaz arasındaki Bluetooth Low Energy (BLE) haberleşmesini tanımlar.

---

## 2. Genel Yapı

- BLE rolü: Peripheral (ESP32)
- Mobil uygulama: Central
- Veri formatı: JSON (UTF-8)

---

## 3. Service Tanımları

### 3.1 Device Information Service
- UUID: `0x180A` (Bluetooth SIG Standart)
- Açıklama: Cihazın adı, üretici, model ve versiyon bilgilerini içerir. Yalnızca read-only karakteristiklerden oluşur. Mobil uygulama bağlantı kurulduktan sonra bu servisi okuyarak cihaz kimliğini doğrular.

### 3.2 Elapsed Time Service
- UUID: `0x183F` (Bluetooth SIG Standart)
- Açıklama: Cihazın son açılışından bu yana geçen süreyi (uptime) izlemek için kullanılır. Mobil uygulama bu servisi okuyarak cihazın ne kadar süredir çalıştığını görebilir.

### 3.3 Device Time Service
- UUID: `0x1847` (Bluetooth SIG Standart)
- Açıklama: Cihaz saatini senkronize etmek için kullanılır. Bağlantı kurulduktan hemen sonra mobil uygulama tarafından bir kez write edilir.

### 3.4 Feeding Service
- UUID: `6E400000-B5A3-F393-E0A9-E50E24DCCA9E`
- Açıklama: Besleme komutları, zamanlayıcı yönetimi ve cihaz durumu için kullanılır.

---

## 4. Device Information Service Characteristic Tanımları

### 4.1 Device Name
- UUID: `0x2A00`
- Properties: Read
- Açıklama: Cihazın BLE üzerinden yayınlanan adı.
- Örnek Değer: `"AquaFeeder"`

### 4.2 Manufacturer Name String
- UUID: `0x2A29`
- Properties: Read
- Açıklama: Üretici adı. Sabit string olarak firmware'de tanımlanır.
- Örnek Değer: `"AquaControl"`

### 4.3 Model Number String
- UUID: `0x2A24`
- Properties: Read
- Açıklama: Cihaz model numarası.
- Örnek Değer: `"FishFeeder-v1"`

### 4.4 Firmware Revision String
- UUID: `0x2A26`
- Properties: Read
- Açıklama: Çalışan firmware versiyonu. `system_config.h` içindeki `FIRMWARE_VERSION` değeriyle eşleşir.
- Örnek Değer: `"1.5.0"`

### 4.5 Hardware Revision String
- UUID: `0x2A27`
- Properties: Read
- Açıklama: Donanım revizyonu. `system_config.h` içindeki `HARDWARE_VERSION` değeriyle eşleşir.
- Örnek Değer: `"1.0"`

---

## 5. Elapsed Time Service Characteristic Tanımları

### 5.1 Elapsed Time Characteristic
- UUID: `0x2BF2`
- Properties: Read, Notify
- Açıklama: Cihazın son açılışından itibaren geçen süreyi saniye cinsinden döner. Bağlantı kurulduktan sonra okunabilir; cihaz 60 saniyede bir notify gönderir.

#### Örnek Payload:
```json
{
  "uptime_seconds": 3600
}
```

> **Not:** `uptime_seconds` değeri her yeniden başlatmada sıfırlanır. Değer `esp_timer_get_time()` fonksiyonundan türetilir.

---

## 6. Device Time Service Characteristic Tanımları

### 6.1 Time Command Characteristic
- UUID: `6E300001-B5A3-F393-E0A9-E50E24DCCA9E`
- Properties: Write
- Açıklama: Mobil uygulama bağlantı kurulduktan hemen sonra cihaza UNIX timestamp gönderir. Tekrar gönderim gerekmez.

#### Payload:
```json
{
  "cmd": "set_time",
  "data": {
    "timestamp": 1745481600,
    "tz_offset": 180
  }
}
```

> **Not:** `tz_offset` dakika cinsinden UTC farkını belirtir (örn. UTC+3 için `180`). Opsiyonel; gönderilmezse cihaz son bilinen değeri kullanır.

---

### 6.2 Time Response Characteristic
- UUID: `6E300002-B5A3-F393-E0A9-E50E24DCCA9E`
- Properties: Notify
- Açıklama: `set_time` komutunun sonucu bu karakteristik üzerinden bildirilir.

#### Payload:
```json
{
  "type": "response",
  "cmd": "set_time",
  "status": "success"
}
```

---

## 7. Feeding Service Characteristic Tanımları

### 7.1 Command Characteristic
- UUID: `6E400001-B5A3-F393-E0A9-E50E24DCCA9E`
- Properties: Write
- Açıklama: Mobil uygulama besleme komutlarını bu karakteristik üzerinden gönderir.

#### Komutlar:

**feed — Hızlı Besleme:**
```json
{
  "cmd": "feed",
  "data": {
    "portion": 3,
    "portion_interval": 2000,
    "feed_type": "quick"
  }
}
```

**set_schedule — Plan Ekle:**
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

**update_schedule — Plan Düzenle:**
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

**remove_schedule — Plan Sil:**
```json
{
  "cmd": "remove_schedule",
  "data": {
    "hour": 8,
    "minute": 30
  }
}
```

**clear_schedules — Tüm Planları Sil:**
```json
{
  "cmd": "clear_schedules"
}
```

**get_status — Durum Sorgula:**
```json
{
  "cmd": "get_status"
}
```

**reset — Cihazı Sıfırla:**
```json
{
  "cmd": "reset"
}
```

**clear_logs — Besleme Loglarını Temizle:**
```json
{
  "cmd": "clear_logs"
}
```

**lighting_on — Aydınlatmayı Aç:**
```json
{
  "cmd": "lighting_on"
}
```

**lighting_off — Aydınlatmayı Kapat:**
```json
{
  "cmd": "lighting_off"
}
```

---

### 7.2 Response Characteristic
- UUID: `6E400002-B5A3-F393-E0A9-E50E24DCCA9E`
- Properties: Notify
- Açıklama: Her besleme komutuna karşılık gelen başarı/hata cevabı.

#### Örnek Payload:
```json
{
  "type": "response",
  "cmd": "feed",
  "status": "success",
  "time": "2025-04-24 08:30:00"
}
```

---

### 7.3 Event Characteristic
- UUID: `6E400003-B5A3-F393-E0A9-E50E24DCCA9E`
- Properties: Notify
- Açıklama: Besleme tamamlandığında veya hata oluştuğunda asenkron olay gönderilir.

#### Örnek Payload:
```json
{
  "type": "feed_result",
  "mode": "manual",
  "portion": 3,
  "status": "success",
  "time": "2025-04-24 08:30:00"
}
```

---

### 7.4 Status Characteristic
- UUID: `6E400004-B5A3-F393-E0A9-E50E24DCCA9E`
- Properties: Read, Notify
- Açıklama: Cihazın anlık durumu, planlar ve versiyon bilgisi.

#### Örnek Payload:
```json
{
  "type": "state",
  "state": "IDLE",
  "fw_version": "1.5.0",
  "hw_version": "1.0",
  "time": "2025-04-24 08:30:00",
  "schedules": [
    { "index": 0, "hour": 8, "minute": 30, "portion": 2, "portion_interval": 60000 },
    { "index": 1, "hour": 20, "minute": 0,  "portion": 1, "portion_interval": 0 }
  ]
}
```

---

## 8. UUID Özet Tablosu

| Kısa Ad                  | UUID                                   | Properties     |
|--------------------------|----------------------------------------|----------------|
| Device Information Svc   | `0x180A`                               | —              |
| Device Name              | `0x2A00`                               | Read           |
| Manufacturer Name String | `0x2A29`                               | Read           |
| Model Number String      | `0x2A24`                               | Read           |
| Firmware Revision String | `0x2A26`                               | Read           |
| Hardware Revision String | `0x2A27`                               | Read           |
| Elapsed Time Service     | `0x183F`                               | —              |
| Elapsed Time             | `0x2BF2`                               | Read + Notify  |
| Device Time Service      | `0x1847`                               | —              |
| Time Cmd                 | `6E300001-B5A3-F393-E0A9-E50E24DCCA9E`| Write          |
| Time Resp                | `6E300002-B5A3-F393-E0A9-E50E24DCCA9E`| Notify         |
| Feeding Service          | `6E400000-B5A3-F393-E0A9-E50E24DCCA9E`| —              |
| Feeding Cmd              | `6E400001-B5A3-F393-E0A9-E50E24DCCA9E`| Write          |
| Feeding Resp             | `6E400002-B5A3-F393-E0A9-E50E24DCCA9E`| Notify         |
| Feeding Event            | `6E400003-B5A3-F393-E0A9-E50E24DCCA9E`| Notify         |
| Feeding Status           | `6E400004-B5A3-F393-E0A9-E50E24DCCA9E`| Read + Notify  |

---

## 9. Bağlantı Akışı

1. Mobil uygulama cihazı tarar
2. Cihaza bağlanır
3. Service keşfi yapılır
4. Notify karakteristikleri aktif edilir (6E300002, 6E400002, 6E400003, 6E400004, 0x2BF2)
5. **Mobil uygulama Device Information Service üzerinden cihaz bilgilerini okur (0x180A)**
6. **Mobil uygulama Elapsed Time Service üzerinden uptime değerini okur (0x183F → 0x2BF2)**
7. **Mobil uygulama Device Time Service üzerinden (6E300001) `set_time` gönderir — bağlantı başına bir kez**
8. Besleme komutları gönderimi başlar (6E400001)

---

## 10. Alan Açıklamaları

| Alan               | Tip    | Açıklama                                      |
|--------------------|--------|-----------------------------------------------|
| `portion`          | int    | Porsiyon adedi (1–10)                         |
| `portion_interval` | int    | Porsiyon arası bekleme süresi (ms, 0–300000)  |
| `feed_type`        | string | `"quick"` (hızlı) veya `"scheduled"` (planlı)|
| `hour`             | int    | Saat (0–23)                                   |
| `minute`           | int    | Dakika (0–59)                                 |
| `old_hour`         | int    | Güncellenecek planın mevcut saati             |
| `old_minute`       | int    | Güncellenecek planın mevcut dakikası          |
| `timestamp`        | int    | UNIX timestamp (yalnızca Device Time Service) |
| `tz_offset`        | int    | UTC farkı dakika cinsinden (örn. 180 = UTC+3) |
| `uptime_seconds`   | int    | Son açılıştan itibaren geçen süre (saniye)    |

---

## 11. Veri Boyutu ve Fragmentation

- Mobil uygulama bağlantı kurulduğunda 512 byte MTU negotiate eder
- Küçük paketler için firmware tarafında reassembly buffer mevcuttur
- JSON veriler parçalanabilir; firmware `{` `}` eşleşmesi tamamlanınca parse eder

---

## 12. Timeout ve Retry

- Komut sonrası max 3 saniye içinde response beklenir
- Response gelmezse retry yapılabilir

---

## 13. Güvenlik

- Opsiyonel: BLE pairing
- Opsiyonel: Encryption

---

## 14. Hata Yönetimi

```json
{
  "type": "error",
  "code": "INVALID_PORTION",
  "message": "Portion out of range"
}
```

---

## 15. Notlar

- Tüm karakteristikler UTF-8 string olarak veri taşır (Device Information Service hariç — raw string)
- `set_time` yalnızca bağlantı kurulduğunda bir kez gönderilir; yanıtı 6E300002 üzerinden gelir
- Feeding Service komutlarının yanıtları 6E400002 üzerinden gelir
- Device Information Service, Elapsed Time Service ve Device Time Service standart Bluetooth SIG servis UUID'leri kullanır
- Feeding Service karakteristikleri (6E4xxxxx) ve Device Time Service karakteristikleri (6E3xxxxx) projeye özgü custom UUID'lerdir
- JSON parse hatalarına karşı firmware tarafında koruma mevcuttur
- Notify kullanımı tercih edilmelidir (polling yerine)

---

## 16. Sonuç

Bu GATT yapısı dört ayrı service ile zaman senkronizasyonu, besleme yönetimi, cihaz kimliği ve uptime izleme sorumluluklarını birbirinden ayırır; sistemin basit, genişletilebilir ve platform bağımsız olmasını sağlar.
