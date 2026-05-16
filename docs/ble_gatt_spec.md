# TOLUN — AKILLI EV & AKVARYUM OTOMASYON PLATFORMU
## BLE GATT SPECIFICATION

---

## 1. Amaç
Bu doküman, TolunControl mobil uygulaması ile Tolun platformu cihazları (AquaFeeder, AquaLighting, WallLighting, HomeLighting) arasındaki Bluetooth Low Energy (BLE) haberleşmesini tanımlar. Tüm cihaz tipleri aynı GATT yapısını paylaşır; komut seti cihaz tipine göre farklılaşır.

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
- Açıklama: Besleme ve aydınlatma komutları, zamanlayıcı yönetimi, cihaz durumu ve log geçmişi için kullanılır. Tüm cihaz tipleri (AquaFeeder, AquaLighting, WallLighting, HomeLighting) aynı servis üzerinden haberleşir; geçerli komut seti cihaz tipine göre farklılaşır.

---

## 4. Device Information Service Characteristic Tanımları

### 4.1 Device Name
- UUID: `0x2A00`
- Properties: Read
- Açıklama: Cihazın BLE üzerinden yayınlanan adı. Format: `<DEVICE_NAME_PREFIX><XXXX>` — prefix `system_config.h`'da build hedefine göre tanımlı, `XXXX` MAC adresinin son 4 hex karakteri.
- Örnek Değerler:
  - `"AquaFeeder-AB12"` (balık besleyici)
  - `"AquaLighting-AB12"` (akvaryum aydınlatma)
  - `"WallLighting-AB12"` (duvar aydınlatma)
  - `"HomeLighting-AB12"` (ev/ortam aydınlatma)

### 4.2 Manufacturer Name String
- UUID: `0x2A29`
- Properties: Read
- Açıklama: Üretici adı. Sabit string olarak firmware'de tanımlanır.
- Örnek Değer: `"TOLUNTECH"`

### 4.3 Model Number String
- UUID: `0x2A24`
- Properties: Read
- Açıklama: Cihaz model numarası. `system_config.h` içindeki `DEVICE_MODEL` makrosuyla eşleşir; her build varyantı için ayrı tanımlanır (Device Name prefix'i ile aynı senkronizasyon kuralı).
- Olası Değerler: `"AquaFeeder"`, `"AquaLighting"`, `"WallLighting"`, `"HomeLighting"`

### 4.4 Firmware Revision String
- UUID: `0x2A26`
- Properties: Read
- Açıklama: Çalışan firmware versiyonu. `system_config.h` içindeki `FIRMWARE_VERSION` değeriyle eşleşir. **v1.20.0'dan itibaren mobil uygulama `fwVersion`'u status payload'undan değil bu karakteristikten okur.**
- Örnek Değer: `"1.20.0"`

### 4.5 Hardware Revision String
- UUID: `0x2A27`
- Properties: Read
- Açıklama: Donanım revizyonu. `system_config.h` içindeki `HARDWARE_VERSION` değeriyle eşleşir. **v1.20.0'dan itibaren mobil uygulama `hwVersion`'u status payload'undan değil bu karakteristikten okur.**
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
    "portion_interval": 2000
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

**set_light — Renk ve Parlaklığı Atomik Ayarla:**
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

**set_color — Yalnızca Renk Ayarla:**
```json
{
  "cmd": "set_color",
  "data": { "r": 0, "g": 120, "b": 255 }
}
```

**set_brightness — Yalnızca Parlaklık Ayarla:**
```json
{
  "cmd": "set_brightness",
  "data": { "brightness": 75 }
}
```

**get_schedule — Tek Planı İndeks ile Getir:**
```json
{
  "cmd": "get_schedule",
  "data": { "index": 0 }
}
```

> Yanıt `6E400002` notify üzerinden gelir. `index` 0'dan `schedule_count - 1`'e kadar tekrarlanarak tüm planlar çekilir.

**get_feed_log_entry — Tek Besleme Log Kaydını İndeks ile Getir:**
```json
{
  "cmd": "get_feed_log_entry",
  "data": { "index": 0 }
}
```

> Her response ~160 B. `index` 0'dan `feed_log_count - 1`'e kadar tekrarlanır.

**get_lighting_log_entry — Tek Aydınlatma Log Kaydını İndeks ile Getir:**
```json
{
  "cmd": "get_lighting_log_entry",
  "data": { "index": 0 }
}
```

> Her response ~145 B. `index` 0'dan `lighting_log_count - 1`'e kadar tekrarlanır.

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
- Açıklama: Cihazın anlık durumu ve sayım bilgileri. Schedules array **içermez** — planlar per-index `get_schedule` komutuyla ayrıca çekilir. Log sayıları per-index `get_feed_log_entry` / `get_lighting_log_entry` döngüsü için kullanılır.

#### Örnek Payload:
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

> `schedules` array'i v1.16.0'dan itibaren status payload'undan çıkarıldı. Planlar `get_schedule` komutu ile indeks bazlı çekilir.
>
> `fw_version` ve `hw_version` alanları v1.20.0'dan itibaren status payload'undan çıkarıldı. Mobil uygulama bu değerleri DIS karakteristiklerinden okur: FW Revision (`0x2A26`) ve HW Revision (`0x2A27`). MTU bütçesi için ~30 B tasarruf.
>
> `first_conn_ts` ve `total_uptime_sec` alanları v1.19.0'da eklendi — cihazın ilk BT pairing zamanı (unix ts, `0` = henüz bağlanmamış) ve tüm güç çevrimleri boyunca biriken toplam çalışma saniyesi.

---

> **Tarihsel not — kaldırılan log karakteristikleri:** v1.14.0'da Feeding Service altına `6E400005` (Feed Log) ve `6E400006` (Lighting Log) salt-okunur karakteristikleri eklenmişti — tek ATT Read ile tüm log geçmişini JSON array olarak döndürüyorlardı. v1.17.0'da Android'in 512 B `BluetoothGatt.MAX_ATTR_LEN` sınırı yüzünden 20 entry'lik bir log JSON'unun kesilmesi/parse hatası vermesi üzerine mobil per-index komut akışına geçti. v1.20.0'da karakteristikler firmware'den tamamen kaldırıldı; log transferi yalnızca `get_feed_log_entry` / `get_lighting_log_entry` komutları üzerinden yapılır (her response ~150–250 B, MTU bütçesinin altında).

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
| `hour`             | int    | Saat (0–23)                                   |
| `minute`           | int    | Dakika (0–59)                                 |
| `end_hour`         | int    | Plan bitiş saati — yalnızca lighting planları (0–23) |
| `end_minute`       | int    | Plan bitiş dakikası — yalnızca lighting planları (0–59) |
| `old_hour`         | int    | Güncellenecek planın mevcut saati             |
| `old_minute`       | int    | Güncellenecek planın mevcut dakikası          |
| `kind`             | int    | Plan tipi — `0` feeder, `1` lighting          |
| `active`           | bool   | Plan aktif/pasif bayrağı (true/false)         |
| `r`, `g`, `b`      | int    | Renk kanalları (0–255) — lighting komutları ve planları |
| `brightness`       | int    | Parlaklık (0–100) — lighting komutları ve planları |
| `index`            | int    | Per-index sorgu indeksi (0…count-1)           |
| `timestamp`        | int    | UNIX timestamp (yalnızca Device Time Service) |
| `tz_offset`        | int    | UTC farkı dakika cinsinden (örn. 180 = UTC+3) |
| `uptime_seconds`   | int    | Son açılıştan itibaren geçen süre (saniye)    |
| `first_conn_ts`    | int    | İlk BT bağlantı unix ts (status payload)      |
| `total_uptime_sec` | int    | Tüm güç çevrimleri toplam çalışma saniyesi (status payload) |

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
