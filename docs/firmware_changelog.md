# FIRMWARE CHANGELOG
## Akvaryum Otomatik Yemleme Sistemi — FishFeeder (ESP32-S3)

> **Durum:** 🧪 Test Aşaması  
> **Son Sürüm:** v1.15.0  
> **Framework:** ESP-IDF v6.0, FreeRTOS

---

## v1.15.0 — Lighting Planlarına Kalıcı Renk Desteği (10 Mayıs 2026)

### Özet
Aydınlatma planlarının renk, doygunluk ve parlaklık bilgileri artık cihazın NVS hafızasında kalıcı olarak saklanıyor. Telefon bağlı olmadan plan tetiklendiğinde de doğru renk kullanılıyor.

### Değişiklikler

#### 1. Scheduler — Renk Alanları NVS'e Eklendi
- `sched_lighting_params_t` struct'ına `r`, `g`, `b`, `brightness` alanları eklendi
- `sched_entry_packed_t` binary layout güncellendi: `_pad` alanı kaldırılıp yerine `color_r`, `color_g`, `color_b`, `color_brightness` alanları eklendi (24 → 28 byte)
- `SCHED_NVS_VERSION` 5 → 6 (layout değişikliğinden dolayı eski NVS verileri otomatik atılır)
- `pack_entry()` / `unpack_entry()`: Lighting entry'leri için renk alanları serialize/deserialize edildi
- `enqueue_cmd()`: Scheduler tetiklediğinde renk alanları `command_t`'ye kopyalanıyor

#### 2. Command Handler — Turn-On Öncesi Renk Uygulaması
- `build_entry_from_cmd()`: `SCHED_KIND_LIGHTING` case'inde `r`, `g`, `b`, `brightness` komut struct'tan entry'ye kopyalanıyor
- `handle_lighting()`: `lighting_on` öncesi `cmd->brightness > 0` ise `lighting_strip_set_color()` + `lighting_strip_set_brightness()` çağrılıyor — ışık ilk açılışta doğru rengi kullanıyor

#### 3. JSON Parser — State Response'a Renk Alanları Eklendi
- `json_build_state()`: Lighting schedule entry'lerinde `brightness > 0` ise `r`, `g`, `b`, `brightness` JSON'a ekleniyor
- Mobil uygulama bağlandığında plan renkleri cihazdan okunabiliyor

#### 4. BLE Service — Status Buffer Büyütüldü
- `status_buf`: 768 → 1536 byte (renk alanlarıyla birlikte 8 plan için yeterli alan)

---

## v1.0.0 — İlk Sürüm (18 Nisan 2026)

### Genel Bilgiler
- **MCU:** ESP32-S3
- **Framework:** ESP-IDF v6.0 (FreeRTOS)
- **İletişim:** BLE (NimBLE stack) — Peripheral
- **Motor:** Servo (LEDC PWM)
- **Sensör:** Yok

---

### Oluşturulan Modüller

#### 1. Proje Altyapısı
- **`main/config/system_config.h`** — Sistem geneli sabitler (pin tanımları, porsiyon limitleri, queue boyutları, NVS anahtarları, güvenlik limitleri)
- **`sdkconfig.defaults`** — ESP32-S3 + NimBLE peripheral-only + NVS + Watchdog yapılandırması
- **`main/CMakeLists.txt`** — Tüm kaynak dosyalar, include path'ler ve bileşen bağımlılıkları (`nvs_flash`, `bt`, `esp_driver_ledc`, `esp_timer`, `cjson`)
- **`main/idf_component.yml`** — IDF Component Manager ile `espressif/cjson` bağımlılığı (ESP-IDF v6.0'da cJSON artık harici bileşen)

#### 2. Utility Katmanı
- **`main/utils/logger.h/c`** — ESP_LOG wrapper makroları (`LOG_INFO`, `LOG_WARN`, `LOG_ERROR`, `LOG_DEBUG`), global log seviyesi ayarı
- **`main/utils/json_parser.h/c`** — cJSON tabanlı komut parser ve response builder
  - `json_parse_command()` — Gelen JSON'u `command_t` struct'a çevirir
  - `json_build_response()` — Komut cevabı oluşturur
  - `json_build_feed_event()` — Yemleme sonucu event'i oluşturur
  - `json_build_state()` — State JSON'u oluşturur
  - `json_build_error()` — Hata bildirimi oluşturur

#### 3. Core Katmanı — State Manager
- **`main/core/state_manager.h/c`** — Thread-safe state machine (Mutex korumalı)
  - State'ler: `IDLE`, `RUNNING`, `ERROR`
  - Geçiş kuralları: RUNNING sırasında yeni komut engellenır, ERROR'da sadece reset kabul edilir
  - API: `state_get()`, `state_set()`, `state_can_feed()`, `state_to_string()`

#### 4. Core Katmanı — Command Handler
- **`main/core/command_handler.h/c`** — Merkezi komut işleme task'ı (FreeRTOS Task)
  - Feeding Service (`6E400001`) üzerinden gelen komutlar:
    - `feed` — Manuel yemleme (porsiyon 1-10 doğrulama)
    - `set_schedule` — Zamanlama ekleme (saat/dakika/porsiyon doğrulama)
    - `update_schedule` — Mevcut planı güncelleme (eski saat/dakika → yeni)
    - `remove_schedule` — Tek bir planı silme (saat/dakika ile)
    - `clear_schedules` — Tüm planları silme
    - `get_status` — Anlık durum, planlar ve versiyon sorgulama
    - `reset` — State'i IDLE'a sıfırlama
  - `set_time` ayrı kanaldan: Device Time Service (`6E300001`) üzerinden alınır, yanıtı `6E300002` notify ile döner
  - Command queue'dan okur, doğrular, motor queue'ya yönlendirir

#### 5. Driver Katmanı — Servo Motor
- **`main/drivers/motor_driver.h/c`** — LEDC PWM ile servo kontrol (FreeRTOS Task)
  - 50Hz PWM, 14-bit çözünürlük
  - Varsayılan GPIO: 4
  - Yemleme: servo aç → porsiyon × 1 saniye bekle → servo kapat
  - Güvenlik: Maksimum çalışma süresi (15 sn), emergency stop
  - State geçişlerinde Status notify gönderimi (RUNNING/IDLE)

#### 6. Zamanlayıcı
- **`main/scheduler/scheduler.h/c`** — Zaman bazlı otomatik yemleme
  - NVS'de kalıcı saklama (maksimum 8 zamanlama)
  - `esp_timer` ile her 60 saniyede periyodik kontrol
  - Zamana geldiğinde otomatik `feed` komutu oluşturma
  - API: `scheduler_add_entry()`, `scheduler_remove_entry()`, `scheduler_get_count()`

#### 7. BLE Servisi (NimBLE GATT)
- **`main/comm/ble_service.h/c`** — Tek modülde dört GATT servisi
  - **Device Information Service** (`0x180A`): Manufacturer, Model Number, FW/HW Revision, Device Name (read-only)
  - **Elapsed Time Service** (`0x183F`): Uptime saniye cinsinden — read + 60 saniyede bir notify
  - **Device Time Service** (`0x1847`): `set_time` komutu (`6E300001` write) ve yanıtı (`6E300002` notify)
  - **Feeding Service** (`6E400000-B5A3-F393-E0A9-E50E24DCCA9E`):

    | UUID (son) | Tip | Açıklama |
    |---|---|---|
    | `...0001` | Write | Mobil → Cihaz komut |
    | `...0002` | Notify | Cihaz → Mobil cevap (Response) |
    | `...0003` | Notify | Asenkron olaylar (Event) |
    | `...0004` | Read + Notify | Anlık durum (Status) |

  - Advertising: `"AquaFeeder"` adıyla yayın
  - MTU: 256 byte tercih (Central tarafından negotiate)
  - Response dispatch task: `resp_queue`'dan okuyup doğru characteristic'e notify gönderir

#### 8. Ana Uygulama
- **`main/main.c`** — Tüm modüllerin init sırası ve queue bağlantıları
  - Init sırası: Logger → NVS → State Manager → Queues → Motor → Command Handler → Scheduler → BLE

---

### Mimari (Queue Tabanlı İletişim)

```
BLE Write → cmd_queue → Command Handler → motor_queue → Motor Task (Servo)
                              ↓                              ↓
BLE Notify ← resp_queue ←────┴──────────────────────────────┘
Scheduler → cmd_queue (auto-feed)
```


---


### v1.0.1 — Yeni Eklenen Özellikler ve İyileştirmeler (18 Nisan 2026)

#### 1. Zaman Kalıcılığı (Time Persistence)
Cihaz, mobil uygulamadan saat bilgisini aldığında bunu NVS hafızasına kaydeder. Cihaz kapansa bile açıldığında en son bildiği zamandan devam eder.

#### 2. Gelişmiş Durum Bildirimi
`get_status` komutu artık şunları içerir:
-   **`time`**: Cihazın şu anki okunabilir saati.
-   **`schedules`**: Hafızadaki tüm kayıtlı zamanlamaların listesi.

#### 3. Okunabilir Zaman Formatı
Tüm bildirimlerde (`Response`, `Event`, `Status`) kullanılan Unix timestamp (saniye) formatı yerine, `YYYY-MM-DD HH:MM:SS` formatında `"time"` alanı kullanılmaya başlanmıştır.

#### 4. Toplu Silme Komutu
Tüm zamanlamaları tek seferde temizlemek için `clear_schedules` komutu eklenmiştir.

---

### v1.0.2 — Periyodik Zaman Kaydı (18 Nisan 2026)

#### 1. Tam Otomatik Zaman Kalıcılığı
Cihaz, artık sadece manuel saat ayarı yapıldığında değil, **her dakika başında** o anki saati kalıcı hafızaya (NVS) yazar. Bu sayede:
- Elektrik kesintilerinde cihazın saati en fazla 1 dakika geri kalır.
- Sistem ayağa kalktığında otomatik olarak en güncel "bookmark" değerinden devam eder.
- Manuel müdahale ihtiyacı minimuma indirilmiştir.


---


### v1.1.1 — Güvenlik ve Kararlılık Düzeltmeleri (22 Nisan 2026)

#### 1. Timer Callback'te Bloklamalı NVS Çağrısı Giderildi
`scheduler_timer_cb()` içinde doğrudan yapılan `nvs_open/nvs_set/nvs_commit` çağrıları FreeRTOS timer daemon task'ını bloke ediyordu. NVS yazma işlemi ayrı bir `time_persist_task`'a taşındı.

#### 2. Race Condition Düzeltmeleri
- `motor_driver.c`: `s_busy` ve `s_stop_requested` değişkenleri `volatile` ile korunuyordu ancak çift çekirdekli ESP32'de atomicity garantisi yoktu. `portMUX_TYPE` spinlock ile tüm erişimler korundu.
- `led_indicator.c`: `s_current_mode` değişkeni `led_task` ve `led_indicator_set_mode()` arasında spinlock ile korundu.

#### 3. BLE Status Buffer Taşması Giderildi
`ble_status_access_cb()` içindeki 512 baytlık stack buffer, maksimum 8 zamanlama girişiyle dolabiliyordu. Buffer 1024 bayta çıkarıldı.

#### 4. JSON Brace Sayımı Düzeltildi
`rx_buf_has_complete_json()` string değerleri içindeki `{` `}` karakterlerini yanlış sayıyordu. `in_string` / `escaped` durum takibi eklenerek düzeltildi.

#### 5. NVS Dönüş Değerleri Kontrol Altına Alındı
`command_handler.c` ve `scheduler.c` içindeki `nvs_set_u32`, `nvs_set_blob`, `nvs_commit` çağrılarının dönüş değerleri artık kontrol ediliyor; hata durumunda sessizce geçmek yerine log basılıyor.

#### 6. Queue Dolu Durumu Loglanıyor
`motor_driver.c`, `command_handler.c` içindeki `xQueueSend()` çağrılarının dönüş değerleri kontrol ediliyor; queue dolu olduğunda mesaj kaybı `LOG_WARN` ile raporlanıyor.

#### 7. JSON Parser Güvenlik İyileştirmeleri
- `strncpy` sonrası açık null sonlandırıcı eklendi.
- Timestamp `double` → `uint32_t` cast öncesi aralık kontrolü yapılıyor.
- `json_build_state()` içinde `sched_count` değeri `MAX_SCHEDULES` ile sınırlandırıldı.

---

### v1.5.0 — Aydınlatma Kontrolü ve Zaman Dilimi Desteği (Tarih bilinmiyor)

> **Not:** Bu sürüm ve arasındaki sürümler (v1.2.0–v1.4.x) retrospektif olarak belgelenmiştir. Kesin tarihler ve ara sürüm detayları git geçmişinden alınmalıdır.

#### 1. Aydınlatma Kontrolü
- `lighting_on` ve `lighting_off` komutları eklendi (Feeding Service `6E400001` üzerinden)
- Cihaza bağlı aydınlatma modülünü açma/kapama desteği

#### 2. Zaman Dilimi (Timezone) Desteği
- `set_time` komutu `tz_offset` alanını kabul ediyor
- `tz_offset`: Dakika cinsinden UTC farkı (örn. UTC+3 için `180`)
- NVS anahtarı: `tz_offset`
- JSON parser'a `tz_offset` alanı eklendi (`json_parser.h`)

#### 3. Besleme Log Modülü
- Cihaz taraflı besleme logu eklendi
- `FEED_LOG_MAX 20` girişe kadar log tutulur (`system_config.h`)
- `clear_logs` komutu ile tüm log geçmişi temizlenebilir

---

### v1.1.0 — LED Durum Göstergesi (18 Nisan 2026)

#### 1. Görsel Durum Geri Bildirimi
Cihazın Bluetooth durumunu izlemek için GPIO 13 üzerine bir LED göstergesi eklendi.
- **Breathing (Nefes Alma) Efekti:** Cihaz bağlı değilken (reklam/advertising modu) LED parlaklığı yavaşça artıp azalır.
- **Solid (Sabit Yanma):** Bir mobil cihaz bağlandığında LED sabit yanmaya başlar.
- **Otomatik Geçiş:** Bağlantı kesildiğinde LED otomatik olarak reklam moduna döner.

#### 2. Donanım ve PWM Altyapısı
- LED kontrolü için servo motorla çakışmayacak şekilde bağımsız bir LEDC kanalı (Channel 1) ve bağımsız bir zamanlayıcı (Timer 1) kullanıldı.
- Sinüzoidal parlaklık geçişi için FreeRTOS task tabanlı kontrolcü eklendi.


---


### JSON Komut Referansı (Genişletilmiş)

#### Manuel Yemleme
`{"cmd":"feed","data":{"portion":2}}`

#### Zamanlama Ekleme
`{"cmd":"set_schedule","data":{"hour":8,"minute":0,"portion":3}}`

#### Zamanlama Güncelleme
`{"cmd":"update_schedule","data":{"old_hour":8,"old_minute":0,"hour":9,"minute":0,"portion":3,"portion_interval":60000}}`

#### Zamanlama Silme
`{"cmd":"remove_schedule","data":{"hour":8,"minute":0}}`

#### Tüm Zamanlamaları Silme
`{"cmd":"clear_schedules"}`

#### Durum Sorgulama (Gelişmiş)
`{"cmd":"get_status"}`

#### Saat Ayarlama (Kalıcı hafızaya yazar)
`{"cmd":"set_time","data":{"timestamp":1744930095,"tz_offset":180}}`
> Bu komut **Device Time Service** üzerinden gönderilir: Write `6E300001`, Notify yanıt `6E300002`. Diğer komutlar Feeding Service'in `6E400001` karakteristiğinden gider.

#### Sistem Sıfırlama
`{"cmd":"reset"}`

#### Besleme Loglarını Temizle
`{"cmd":"clear_logs"}`

#### Aydınlatma Kontrolü
`{"cmd":"lighting_on"}`
`{"cmd":"lighting_off"}`

---

### Test Aracı
- **nRF Connect** (Android/iOS) ile BLE üzerinden test
- Notify'ları aktifleştir → Command characteristic'e JSON yaz → Response/Event/Status notify'larını gözlemle

---


### Derleme Sırasında Çözülen Sorunlar

| # | Sorun | Çözüm |
|---|---|---|
| 1 | `json` bileşeni bulunamadı | ESP-IDF v6.0'da cJSON harici bileşen → `idf_component.yml` ile `espressif/cjson` eklendi |
| 2 | `driver/ledc.h` bulunamadı | ESP-IDF v6.0'da driver parçalandı → `esp_driver_ledc` bileşeni kullanıldı |
| 3 | `snprintf` truncation uyarısı (-Werror) | `scheduler.c`'de NVS key buffer'ı `char key[16]` → `char key[24]` |
| 4 | `ble_gattc_exchange_mtu` undefined | Peripheral-only NimBLE'da GATT Client yok → çağrı kaldırıldı, Central MTU exchange yapıyor |
| 5 | BLE write fragmentation (20 byte limit) | Reassembly buffer eklendi: chunk'lar biriktirilip `{}` eşleşince JSON parse ediliyor |
| 6 | Status characteristic notify etmiyor | `motor_driver.c`ye `send_status_notify()` eklendi: RUNNING/IDLE geçişlerinde otomatik notify |
| 7 | Zaman Sıfırlanma Sorunu (Power Cycle) | Cihaz resetlendiğinde zamanın 1970'e dönmesi, NVS tabanlı "Time Persistence" (Zaman Kalıcılığı) ile çözüldü. |
| 8 | Yeni Sürücüler Derlenmiyor (Linker Error) | `led_indicator.c` gibi yeni eklenen sürücü dosyaları `main/CMakeLists.txt`'ye manuel eklenerek çözüldü. |