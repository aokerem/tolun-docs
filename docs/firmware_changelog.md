# FIRMWARE CHANGELOG
## Tolun — Akıllı Ev & Akvaryum Otomasyon Platformu (ESP32-S3)

> **Durum:** 🧪 Test Aşaması  
> **Son Sürüm:** v1.19.2  
> **Framework:** ESP-IDF v6.0, FreeRTOS

---

## v1.19.2 — NVS Yazma Sıklığı Azaltıldı (14 Mayıs 2026)

### Changed
- `main.c` `daily_reset_timer_callback`: `device_stats_persist()` her dakika yerine **her 5 dakikada bir** çağrılıyor. Daily reset kontrolleri ise (feed/lighting log gün dönümü) her dakika koşmaya devam ediyor. NVS yıpranmasını ~%80 azaltır.
- `scheduler/scheduler.c` `time_persist_task`: `vTaskDelay` 60000 → **300000 ms** (5 dakika). `NVS_KEY_LAST_TIME` artık 5 dakikada bir yazılıyor.

### Background
60 saniyelik yazma frekansı 3 yıllık serviste sektör başına ~6.500 erase cycle anlamına geliyordu (ESP32 100K rated). 5 dakikaya çekildiğinde ~1.300 cycle'a düşüyor — %1 yıpranma, 15+ yıllık marja taşır. Worst-case güç kesintisinde 5 dk uptime/saat kaybı (servis ömrü metriği için ihmal edilebilir).

---

## v1.19.1 — first_conn_ts Set_Time Sonrası Yeniden Dene (14 Mayıs 2026)

### Fixed
- `comm/ble_service.c`: `ble_time_cmd_access_cb` set_time başarıyla işlendikten sonra `device_stats_register_connection()` çağrısı eklendi. İlk-ever pairing senaryosunda GAP_EVENT_CONNECT firingi anında `time(NULL)==0` olabildiğinden (NVS'te kaydedilmiş saat yok) ilk register denemesi atlanıyor; set_time geldiğinde sistem saati artık geçerli olduğundan ts başarıyla yazılıyor.

---

## v1.19.0 — Cihaz Yaşam İstatistikleri (14 Mayıs 2026)

### Özet
Cihazın ilk BT bağlantı zamanı ve toplam çalışma süresi (tüm güç çevrimleri boyunca biriken) NVS'te kalıcı tutuluyor. Bu bilgiler her BT bağlantısında terminal log'una basılıyor ve BLE status payload'una eklenerek mobil uygulamaya iletiliyor — sahadaki cihazların ne zaman kullanıma alındığı ve ne kadar çalıştığı uzaktan görüntülenebiliyor.

### Added
- `config/system_config.h`: `NVS_KEY_FIRST_CONN_TS` ve `NVS_KEY_TOTAL_UPTIME` anahtarları.
- `core/device_stats.h/.c`: Yaşam istatistikleri modülü.
  - `device_stats_init()` — NVS'ten son durumu yükler, boot referans zamanını alır.
  - `device_stats_register_connection()` — ilk BT bağlantısında unix timestamp NVS'e yazılır; sonraki bağlantılarda no-op.
  - `device_stats_persist()` — anlık toplam uptime'i NVS'e flush eder.
  - `device_stats_get_first_connect_ts()` / `device_stats_get_total_uptime_sec()` — getter'lar.
- `main.c`: `device_stats_init()` NVS init sonrası; `daily_reset_timer_callback` her 60 saniyede `device_stats_persist()` çağrısı.
- `comm/ble_service.c`: `BLE_GAP_EVENT_CONNECT` içinde `device_stats_register_connection()` çağrılıyor; ardından ilk bağlantı tarihi ve toplam uptime terminal log'una basılıyor (örn. `first connect: 2026-05-14 12:30:00, total uptime: 12345 s (~3 h 25 m)`).
- `utils/json_parser.h/.c`: `json_build_state()` imzasına `first_conn_ts` ve `total_uptime_sec` parametreleri eklendi; status payload'una bu iki alan dahil ediliyor.

### Changed
- `comm/ble_service.c`: `json_build_state` çağrısı yeni parametrelerle güncellendi.

---

## v1.18.0 — Aydınlatma Durum Kalıcılığı (14 Mayıs 2026)

### Özet
Aydınlatma durumunun (açık/kapalı + renk + parlaklık) elektrik kesintisi sonrası korunması eklendi. Cihaz manuel olarak açıkken güç kesilse bile, elektrik geldiğinde firmware NVS'den durumu okuyup şeridi otomatik olarak aynı renk ve parlaklıkta yeniden açıyor.

### Added
- `config/system_config.h`: `NVS_KEY_LIGHT_STATE` anahtarı eklendi — packed blob (is_on, r, g, b, brightness).
- `drivers/lighting_strip.h/.c`: `lighting_strip_restore_state()` fonksiyonu eklendi — NVS'den son durumu okuyup `is_on`/`base_r/g/b`/`brightness` alanlarını set eder; was-on ise transmit ederek şeridi açar.
- `drivers/lighting_strip.c`: `persist_state()` static helper'ı eklendi — `is_on/r/g/b/brightness`'ı 8 baytlık packed blob olarak NVS'e yazıyor.

### Changed
- `drivers/lighting_strip.c`: `lighting_strip_on()`, `lighting_strip_off()`, `lighting_strip_set_color()`, `lighting_strip_set_brightness()` çağrılarının her biri durum değişikliğinden sonra `persist_state()` çağırıyor.
- `main.c`: NVS init sonrasında (state manager init'inden önce) `lighting_strip_restore_state()` çağrılıyor.

### Davranış
- **Manuel ON → güç kesintisi → güç geldi:** Şerit aynı renk ve parlaklıkta otomatik açılır.
- **Manuel OFF → güç kesintisi:** Şerit kapalı kalır.
- **Plan aktifken güç kesintisi:** Son persist edilen durum geri yüklenir; scheduler tick'i sonraki dakikada plan zamanı içindeyse plan rengini override edebilir.

---

## v1.17.2 — Response Payload Sadeleştirmesi (14 Mayıs 2026)

### Removed
- `utils/json_parser.h` / `utils/json_parser.c`: `json_build_schedule()`, `json_build_feed_log_entry()` ve `json_build_lighting_log_entry()` imzalarından `int index` parametresi kaldırıldı; response payload'undan `"index"` alanı çıkarıldı — mobil tarafta indeks zaten sıralı `await` döngüsünde tutuluyordu, ek alan dead data idi.

### Changed
- `core/command_handler.c`: `handle_get_schedule()`, `handle_get_feed_log_entry()`, `handle_get_lighting_log_entry()` çağrıları yeni imzayla güncellendi.

> Her log/schedule response'undan ~10-12 bayt tasarruf; per-index transfer döngüsünde 20 kayıt için toplam ~200-240 bayt daha az trafik.

---

## v1.17.1 — Hata Düzeltmeleri (13 Mayıs 2026)

### Fixed
- `config/system_config.h`: `MOTOR_TIMEOUT_MS` 15000 → **20000** — per-porsiyon süre `RAMP+HOLD+RAMP+COOLDOWN = 2000 ms`; 10 porsiyon için gerçek maksimum 20 sn iken 15 sn timeout hatalı STATE_ERROR geçişine yol açıyordu.
- `comm/ble_service.c`: Uptime notify aralığı 300 s → **60 s** — v1.11.0'da 5 dakikaya çıkarılmıştı, spec ile uyumsuzluk giderildi.
- `utils/json_parser.c`: `portion_interval` parse'ında `valueint` → **`valuedouble`** + `0–300 000 ms` aralık kontrolü — büyük değerlerde signed int overflow riski giderildi; `timestamp` ile tutarlı hale getirildi.
- `core/command_handler.c`: `handle_feed()` içindeki `strcmp(feed_type, "scheduled")` kontrolü kaldırıldı — mobil tarafından gönderilen `feed_type` string'i `is_scheduled` flag'ini yanlışlıkla `true` yapabiliyordu; artık yalnızca scheduler'ın set ettiği `cmd->is_scheduled` kullanılıyor.
- `drivers/oled_display.c`: `prev_state` başlangıç değeri `(system_state_t)0xFF` → **`bool first_run`** bayrağı — geçersiz enum cast kaldırıldı, ilk çizim `first_run` ile güvenli şekilde zorlanıyor.

### Removed
- `utils/json_parser.h` / `utils/json_parser.c`: `feed_type` alanı ve parse kodu kaldırıldı — `is_scheduled` mantığından ayrıştırılmasının ardından firmware'de kullanılmayan dead code haline gelmişti.

---

## v1.17.0 — Per-Index Log Transferi (11 Mayıs 2026)

### Özet
Manuel aydınlatma açma/kapama sonrasında `readLightingLog` ATT okuma isteği Android'in 512 B `BluetoothGatt.MAX_ATTR_LEN` sınırında kesiliyordu (`JSON Parse error: Unexpected end of input`). Besleme ve aydınlatma logları artık v1.16.0'daki per-index `get_schedule` modeliyle aynı şekilde **indeks bazlı `get_feed_log_entry` / `get_lighting_log_entry` komutları** ile tek tek getiriliyor — her transaction MTU notify limitinin çok altında.

### Added
- `core/command_handler.c`: `handle_get_feed_log_entry()` eklendi — mobil `index` parametresiyle tek bir besleme log kaydını sorgular; her response ~160 B.
- `core/command_handler.c`: `handle_get_lighting_log_entry()` eklendi — aynı mimari, tek aydınlatma session kaydı döner; her response ~145 B.
- `utils/json_parser.h/.c`: `json_build_feed_log_entry()` ve `json_build_lighting_log_entry()` fonksiyonları eklendi — tek kayıt için `get_feed_log_entry` / `get_lighting_log_entry` response formatında JSON üretir.
- `utils/json_parser.h`: `command_t.cmd` alanı 20 → **24 bayt** — `"get_lighting_log_entry"` (22 karakter) için alan açıldı.

### Changed
- `utils/json_parser.c`: `json_build_state()` imzasına `feed_log_count` ve `lighting_log_count` parametreleri eklendi; status payload'u artık bu sayıları da içeriyor. Mobil bu sayılar kadar per-index komut döngüsü çalıştırır.
- `comm/ble_service.c`: `ble_status_access_cb` içinde `feed_log_get_all()` ve `lighting_log_get_all()` çağrılarak sayımlar alınıyor, `json_build_state()`'e geçiliyor. `status_buf` 1536 → **512 bayt** (log verisi artık status payload'unda yok).

### Fixed
- Manuel aydınlatma açma/kapama sonrasında `readLightingLog` ATT okuma isteği Android'in 512 B `BluetoothGatt.MAX_ATTR_LEN` sınırında kesiliyordu. Per-index `get_lighting_log_entry` chunklı transfer ile çözüldü.

---

## v1.16.0 — Schedule Chunking ve MTU İyileştirmesi (11 Mayıs 2026)

### Özet
3+ planlı cihazlarda mobil status okuması Android Bluetooth stack'inin **512 B `BluetoothGatt.MAX_ATTR_LEN`** sınırına takılıp JSON'u kesiyordu (`Unexpected end of input`). Status payload'undan schedules array'i çıkarıldı; yerine plan sayısı eklendi. Planlar artık **indeks bazlı `get_schedule` komutu** ile tek tek getiriliyor — her transaction MTU notify limitinin çok altında. ATT preferred MTU 256 → 512'ye yükseltildi.

### Added
- `core/command_handler.c`: `handle_get_schedule()` eklendi — `data.index` aralık kontrolü + `scheduler_get_entry()` + `json_build_schedule()` + resp_queue
- `core/command_handler.c`: Dispatch tablosuna `get_schedule` eklendi
- `utils/json_parser.h/.c`: `json_build_schedule()` — tek bir schedule entry'sini `{type:"response", cmd:"get_schedule", status, data:{...}, time}` formatında döner; kind'a göre feeder/lighting alanlarını ayırır (~240 B max)
- `utils/json_parser.h`: `command_t` struct'ına `int8_t index` alanı eklendi
- `utils/json_parser.c`: Parser `data.index` field'ını okuyup `out->index`'e yazıyor
- `core/command_handler.c`: `handle_set_schedule()` başarılı ekleme sonrası `send_status_update()` çağırıyor (önceden eksikti — diğer mutasyon handler'ları zaten çağırıyordu)

### Changed
- `utils/json_parser.c`: `json_build_state()` imzasından `sched_entries` parametresi çıkarıldı; payload artık `schedules` array yerine yalnızca `schedule_count` (number) içeriyor. Status read'i ~545 B → ~150 B'a düştü.
- `comm/ble_service.c`: `ble_status_access_cb` ve `json_build_state` çağrısı güncellendi; `scheduler_get_all_entries` yerine `scheduler_get_count` kullanılıyor.
- `utils/json_parser.h`: `outgoing_msg_t.payload` 128 → **256 bayt** — bir lighting `get_schedule` response'unun (~240 B) resp_queue'ya sığması için zorunlu. resp_queue başına ~128 B ek RAM.
- `sdkconfig.defaults` & `sdkconfig`: `CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU` 256 → **512** — peripheral artık 512 öneriyor; gerçek pazarlık sonucu central'a (mobil) bağlı.

### Fixed
- Forget + yeniden ekleme akışında 3+ planlı cihazlarda mobil status JSON'u Android'in 512 B sınırında kesiliyordu. Per-index `get_schedule` chunking ile çözüldü.

---

## v1.15.0 — Lighting Planlarına Kalıcı Renk Desteği (10 Mayıs 2026)

### Özet
Aydınlatma planlarının renk, doygunluk ve parlaklık bilgileri artık cihazın NVS hafızasında kalıcı olarak saklanıyor. Telefon bağlı olmadan plan tetiklendiğinde de doğru renk kullanılıyor.

### Changed
- `sched_lighting_params_t` struct'ına `r`, `g`, `b`, `brightness` alanları eklendi.
- `sched_entry_packed_t` binary layout güncellendi: `_pad` alanı kaldırılıp yerine `color_r`, `color_g`, `color_b`, `color_brightness` alanları eklendi (24 → 28 byte).
- `SCHED_NVS_VERSION` 5 → **6** (layout değişikliğinden dolayı eski NVS verileri otomatik atılır).
- `pack_entry()` / `unpack_entry()`: Lighting entry'leri için renk alanları serialize/deserialize edildi.
- `enqueue_cmd()`: Scheduler tetiklediğinde renk alanları `command_t`'ye kopyalanıyor.
- `build_entry_from_cmd()`: `SCHED_KIND_LIGHTING` case'inde `r`, `g`, `b`, `brightness` komut struct'tan entry'ye kopyalanıyor.
- `handle_lighting()`: `lighting_on` öncesi `cmd->brightness > 0` ise `lighting_strip_set_color()` + `lighting_strip_set_brightness()` çağrılıyor — ışık ilk açılışta doğru rengi kullanıyor.
- `json_build_state()`: Lighting schedule entry'lerinde `brightness > 0` ise `r`, `g`, `b`, `brightness` JSON'a ekleniyor.
- `status_buf`: 768 → **1536 byte** (renk alanlarıyla birlikte 8 plan için yeterli alan).

---

## v1.14.0 — Feed / Lighting Log Karakteristikleri (10 Mayıs 2026)

### Added
- `comm/ble_service.c`: Feed log (`6E400005`) ve lighting log (`6E400006`) için iki yeni salt-okunur GATT karakteristiği eklendi; her biri 1536 bayt statik buffer ile ATT Read fragmantasyonu üzerinden tam log verisini sunuyor.
- `utils/json_parser.h/.c`: `json_build_feed_log()` ve `json_build_lighting_log()` fonksiyonları eklendi — logları `{"type":"feed_log","data":[...]}` formatında döner.
- `scheduler/scheduler.h`: `sched_entry_packed_t` (`__attribute__((packed))`, 24 bayt) eklendi — NVS yazma/okumada enum/bool padding kaynaklı alan israfı giderildi.
- `Mobile App/src/core/ble/BleConstants.ts`: `FEED_LOG_CHAR_UUID` ve `LIGHTING_LOG_CHAR_UUID` sabitleri eklendi.
- `Mobile App/src/core/ble/BleService.ts`: `readFeedLog()` ve `readLightingLog()` metotları eklendi; bağlantı kurulumunda `readStatus()` ile paralel çağrılıyor.

### Changed
- `utils/json_parser.h`: `command_t` alanları sıkıştırıldı — `kind`, `portion`, `hour`, `minute`, `end_hour`, `end_minute`, `old_hour`, `old_minute` → `uint8_t`; `portion_interval` → `uint32_t`. Struct boyutu ~120 → ~80 bayta indi; cmd_queue tasarrufu ~400 bayt.
- `utils/json_parser.h`: `outgoing_msg_t.payload` 256 → **128 bayt**; resp_queue tasarrufu ~1280 bayt.
- `utils/json_parser.c`: `json_build_state()` imzası 12 → **8 parametre**; log parametreleri kaldırıldı — status yanıtı artık yalnızca durum + plan listesi içeriyor.
- `comm/ble_service.c`: `status_buf` 2048 → **768 bayt** (log verisi artık ayrı karakteristiklerden okunuyor).
- `comm/ble_service.c`, `core/command_handler.c`, `drivers/motor_driver.c`: `send_status_notify/update` çağrıları sadece `{"type":"state_changed"}` tetikleyici gönderiyor; mobil her zaman ATT Read ile tam veriyi okuyor.
- `scheduler/scheduler.c`: `SCHED_NVS_VERSION` **4 → 5** yükseltildi (`sched_entry_packed_t` formatına geçiş); eski blob'lar boot'ta atılır. NVS read/write `pack_entry()` / `unpack_entry()` üzerinden yapılıyor.
- `Mobile App/src/core/ble/BleService.ts`: `statusSubscription` handler, `readStatus()` sonrasında `readLightingLog()` da çağırıyor; `lighting_off` response'unda log senkronizasyonu eklendi.

---

## v1.13.0 — Aydınlatma Log Modülü (10 Mayıs 2026)

### Added
- `core/lighting_log.h/.c`: Günlük aydınlatma geçmişi ring buffer'ı eklendi — NVS üzerinde kalıcı, her gece yerel gece yarısında otomatik sıfırlama, tamamlanan session başına 8 bayt (`start_ts`, `duration_s`, `is_scheduled`, pad) depolama.
- `utils/json_parser.c`: `json_build_state` fonksiyonuna `llog_count` / `llog_entries` parametreleri eklendi; status JSON'una `lighting_log` dizisi eklendi.
- `comm/ble_service.c`: Status karakteristiği ATT Read yanıtı `lighting_log_get_all()` ile tamamlanan session'ları içeriyor.
- `core/command_handler.c`: `handle_lighting` içinde `lighting_log_on()` / `lighting_log_off()` çağrıları eklendi; `lighting_off` sonrası `send_status_update()` ile güncellenen log mobile'a iletiliyor.
- `main.c`: `lighting_log_init()` ve `lighting_log_check_daily_reset()` 60 s'lik timer'a eklendi.
- `CMakeLists.txt`: `core/lighting_log.c` derleme listesine eklendi.

### Changed
- `config/system_config.h`: `LIGHTING_LOG_MAX`, `NVS_KEY_LLOG_HEAD`, `NVS_KEY_LLOG_CNT`, `NVS_KEY_LLOG_RESET_DATE` sabitleri eklendi.

---

## v1.12.0 — Lighting Handler Basitleştirmesi (10 Mayıs 2026)

### Changed
- `core/command_handler.c`: `handle_set_light` basitleştirildi — koşullu RGB/brightness kontrolleri kaldırıldı; renk ve parlaklık her zaman atomik olarak uygulanır. Gereksiz event payload çıkarıldı.

---

## v1.11.0 — OLED Ekran, Yazılım Ramp ve Feed Animasyonu (10 Mayıs 2026)

### Added
- `drivers/oled_display.h/.c`: SH1106 128×64 OLED sürücüsü eklendi (native ESP-IDF I2C, Arduino bağımlılığı yok). Açılışta "TOLUN" splash ekranı, ardından zaman/durum bilgisi döngüsü.
- `core/command_handler.c`: `set_light` komutu eklendi — renk (R/G/B) ve parlaklığı tek atomik işlemde ayarlar; BLE event olarak yayınlar.
- `drivers/motor_driver.c/.h`: Servo için yazılım ramp (20 ms adımlı, durdurulabilir) eklendi. Porsiyon bazlı takip: `motor_is_open()`, `motor_get_current_portion()`, `motor_get_last_dispensed_portion()`.
- `drivers/lighting_strip.c/.h`: `lighting_strip_feed_anim()` — yemleme sırasında LED'leri tek tek turuncu renkle dolduran animasyon; önceki şerit durumunu kaydeder/geri yükler.
- `core/feed_log.c/.h`: `feed_log_check_daily_reset()` — UTC 00:00'da günlük log sıfırlama; `main.c`'de 60 s'lik `esp_timer` ile periyodik çağrılır.
- `scheduler/scheduler.h`: `schedule_entry_t` yapısına `label[16]` alanı eklendi (plan adı).
- `scheduler/scheduler.c`: `scheduler_get_change_id()` — her plan değişikliğinde artan sayaç.
- `comm/ble_service.c`: `ble_is_connected()` fonksiyonu eklendi.

### Changed
- `drivers/motor_driver.c`: Her porsiyon sonrasına `SERVO_COOLDOWN_MS` bekleme eklendi; init sırasında rampsız kapalı konuma geçiş; servo açık/kapalı durumu spinlock ile korunuyor.
- `drivers/motor_driver.c`: Yemleme sırasında `feed_anim_task` ayrı FreeRTOS task olarak LED animasyonunu paralel çalıştırıyor.
- `core/feed_log.c`: NVS depolama blob yerine per-entry u64 anahtarlarına (`flog_0`…`flog_19`) geçildi — heap parçalanması azaltıldı.
- `comm/ble_service.c`: Status JSON'una artık feed log dahil edilmiyor (heap parçalanması önlendi); statik buffer ile malloc/free döngüsü kaldırıldı.
- `comm/ble_service.c`: CCCD abonelik bitmask'i eklendi — her karakteristik için ayrı sub flag; bildirim yalnızca istemci abone olduğunda gönderilir.
- `comm/ble_service.c`: Uptime bildirimi 60 s → **5 dakika** olarak güncellendi.
- `comm/ble_service.c`: Disconnect → re-advertise koruması eklendi; `ble_on_reset` bağlantı handle ve aboneliği temizliyor.
- `comm/ble_service.c`: `ble_store_util_status_rr` store callback'i eklendi.
- `scheduler/scheduler.c`: `SCHED_NVS_VERSION` 3 → **4** yükseltildi (`label` alanı eklendi).
- `main.c`: OLED init ve günlük sıfırlama timer'ı eklendi.
- `config/system_config.h`: OLED (SH1106) pin/zamanlama sabitleri, `SERVO_COOLDOWN_MS` ve `OLED_FED_STATUS_MS` eklendi.
- `CMakeLists.txt`: `oled_display.c` derleme listesine eklendi.

---

## v1.10.0 — Aktif/Pasif Plan Desteği ve MAX_SCHEDULES 5'e Düşürüldü (29 Nisan 2026)

### Added
- `scheduler/scheduler.h`: `schedule_entry_t` yapısına `active` (aktif/pasif) bayrağı eklendi.
- `utils/json_parser.h/.c`: JSON parser ve builder mekanizması `active` durumunu destekleyecek şekilde güncellendi; `command_t`'ye `active` alanı eklendi.
- `core/command_handler.c`: Komut işleyici, gelen aktiflik durumunu zamanlayıcı kayıtlarına eşlemek üzere güncellendi.

### Changed
- `scheduler/scheduler.c`: Scheduler artık pasif olarak işaretlenmiş planları `scheduler_tick` içinde tetiklemez.
- `scheduler/scheduler.c`: `SCHED_NVS_VERSION` **3**'e yükseltildi (aktiflik durumu hafızada saklanıyor).
- `config/system_config.h`: `MAX_SCHEDULES` bellek optimizasyonu ve UI uyumu için 8 → **5**'e düşürüldü.

---

## v1.9.0 — Gamma Düzeltme ve RMT İyileştirmesi (29 Nisan 2026)

### Added
- `drivers/lighting_strip.c`: **Gamma 2.2 düzeltme tablosu** eklendi — LED renklerinin daha canlı ve insan gözüyle uyumlu görünmesi sağlandı.

### Changed
- `drivers/lighting_strip.c`: RMT `mem_block_symbols` değeri 64 → **256**'ya çıkarıldı — 8 LED ve üzeri donanımlarda sinyal bozulması ve rastgele beyaz yanma sorunları giderildi.
- `drivers/lighting_strip.c`: `lighting_strip_on()` fonksiyonu artık her açılışta renkleri varsayılana sıfırlamıyor; en son set edilen rengi veya başlangıç varsayılanını koruyor.
- `drivers/lighting_strip.c`: `scale()` fonksiyonu hem Gamma düzeltmesini hem de parlaklık ölçeklemesini tek adımda uygulayacak şekilde optimize edildi.
- `config/system_config.h`: Varsayılan "Gün Doğumu" rengi baz HSL değerlerine (L=50) göre (221, 127, 44) olarak güncellendi.

---

## v1.8.0 — LED Durum ve Clock-Aligned Uptime (28 Nisan 2026)

### Added
- `drivers/lighting_strip.c/.h`: `lighting_strip_is_on()` fonksiyonu eklendi — son `on()`/`off()` çağrısına göre şerit durumunu döner; `s` struct'ına `is_on` alanı eklendi.
- `utils/json_parser.h`: `MSG_TYPE_ELAPSED` enum değeri eklendi; uptime bildirimleri response task üzerinden gönderiliyor.
- `utils/json_parser.c`: `json_build_state()`'e `led_on` parametresi eklendi; LED açık/kapalı durumu state JSON'una yansıtılıyor.

### Changed
- `scheduler/scheduler.c`: Periyodik `esp_timer` kaldırıldı; dakikanın `:00` saniyesinde kendiliğinden hizalanan FreeRTOS task ile değiştirildi.
- `comm/ble_service.c`: Uptime task artık sabit 60 s bekleme yerine bir sonraki tam dakikanın başında ateşleniyor; payload'a `time` alanı eklendi. Response task'a `MSG_TYPE_ELAPSED` case eklendi.
- `comm/ble_service.c`, `core/command_handler.c`, `drivers/motor_driver.c`: `json_build_state()` çağrıları `lighting_strip_is_on()` argümanıyla güncellendi.
- `utils/json_parser.c`: `time_str` buffer 32 → **20 bayt**'a küçültüldü.

---

## v1.7.0 — Renk ve Parlaklık Komutları (27 Nisan 2026)

### Added
- `core/command_handler.c`: `set_color` ve `set_brightness` komut handler'ları eklendi (`lighting_strip_set_color` / `lighting_strip_set_brightness` çağırır).
- `utils/json_parser.h`: `command_t`'ye `color_r`, `color_g`, `color_b`, `brightness` alanları eklendi.
- `utils/json_parser.c`: `r`, `g`, `b`, `brightness` JSON alanları parse ediliyor.

### Fixed
- `core/command_handler.c`: `handle_lighting()` başarı durumunda `send_response()` çağırmıyordu; mobil taraf response alamayıp timeout'a düşüyor ve komutu 3 kez tekrar gönderiyordu. `send_event_payload()` öncesine `send_response(event_name, "success")` eklendi.

---

## v1.6.0 — WS2812B LED Şerit Sürücüsü (27 Nisan 2026)

### Added
- `drivers/lighting_strip.c/.h`: WS2812B adreslenebilir şerit LED sürücüsü eklendi (GPIO14, 8 LED).
  - Generic modüler arayüz (`lighting_strip_config_t`, `led_strip_type_t`) — gelecekte SK6812/APA102 desteği için genişletilebilir.
  - ESP-IDF v5/v6 RMT API ile composite encoder: bytes_encoder (GRB piksel verisi, MSB önce) + copy_encoder (>500 µs reset pulse).
  - API: `lighting_strip_on()`, `lighting_strip_off()`, `set_color()`, `set_brightness()`, `set_pixel()`, `refresh()`.
  - Varsayılan "açık" rengi: sıcak beyaz (R=255, G=200, B=100).
- `config/system_config.h`: WS2812B sabitleri eklendi (GPIO, LED sayısı, RMT çözünürlüğü, bit zamanlamaları, varsayılan renk).

### Changed
- `core/command_handler.c`: `handle_lighting()` artık `lighting_strip_on()` / `lighting_strip_off()` çağırarak fiziksel LED şeridini kontrol ediyor.
- `main.c`: Boot sırasında `lighting_strip_init()` çağrısı eklendi.
- `CMakeLists.txt`: `lighting_strip.c` SRCS'e, `esp_driver_rmt` REQUIRES'a eklendi.

---

## v1.5.1 — Aydınlatma Stub Komutları, Zaman Dilimi ve Log Altyapısı (Tarih bilinmiyor)

> **Not:** Retrospektif olarak belgelenmiştir; kesin tarih git geçmişinden alınmalıdır.

### Added
- `lighting_on` ve `lighting_off` komutları Feeding Service (`6E400001`) üzerinden eklendi — bu aşamada yalnızca BLE event gönderiyordu (fiziksel LED sürücüsü v1.6.0'da eklendi).
- `FEED_LOG_MAX 20` girişe kadar log tutulur (`system_config.h`).
- `clear_logs` komutu ile tüm log geçmişi temizlenebilir.

### Changed
- `set_time` komutu `tz_offset` alanını kabul ediyor — dakika cinsinden UTC farkı (örn. UTC+3 için `180`); NVS anahtarı `tz_offset`.
- JSON parser'a `tz_offset` alanı eklendi (`json_parser.h`).

> Günlük sıfırlama ve NVS per-entry depolama v1.11.0'da eklendi.

---

## v1.5.0 — Tagged Union Scheduler ve Lighting Plan Desteği (25 Nisan 2026)

### Added
- Cihaz türüne göre takvim yapısı: `schedule_entry_t` tagged union'a dönüştürüldü (`sched_kind_t` = FEEDER | LIGHTING). Ortak alanlar (`start_hour/minute`, `end_hour/minute`) + kind'a özgü `params` union'ı.
- Aydınlatma planı için bitiş saati desteği: scheduler `start..end` aralığında tutuyor; aralığa giriş/çıkış kenarında `lighting_on` / `lighting_off` event üretir.
- Dakika aralığı için gece-geçişli (wrap-around) hesap: 22:00–06:00 gibi planlar destekleniyor.
- Per-kind tick handler dispatch tablosu (`s_kind_tick[]`): yeni cihaz türü eklemek için sadece enum + handler + tablo satırı gerekir.
- `json_build_kind_event()`: kind ve start/end alanlarını taşıyan generic event üretici.

### Changed
- `command_t`: `kind`, `end_hour`, `end_minute` alanları eklendi; `cmd[16]` → `cmd[20]`.
- `set_schedule` / `update_schedule` payload'u artık `kind` ile birlikte gelir; lighting'de `portion` opsiyonel, `end_hour/end_minute` zorunlu.
- `state` JSON'undaki `schedules[]` her giriş için `kind`, `end_hour`, `end_minute` döner; feeder kayıtları ayrıca `portion` + `portion_interval` taşır.
- NVS şeması v2'ye yükseltildi (`sched_ver` anahtarı). Eski sürüm bulunan kayıtlar yüklemede atılır.

---

## v1.4.0 — Cihaz Adı Güncellendi (25 Nisan 2026)

### Changed
- BLE cihaz adı ön eki `"AquaControl-"` → `"AquaFeeder-"` (yayınlanan isim artık `"AquaFeeder-XXXX"`).
- DIS Model Number `"AquaController-"` → `"AquaFeeder"`.

---

## v1.3.2 — DIS Güncelleme (24 Nisan 2026)

### Changed
- DIS üretici adı `"AquaControl"` → `"TOLUNTECH"`.
- DIS model numarası `"FishFeeder-v1"` → `"AquaController-"`.

---

## v1.3.1 — BLE GATT Spec Dokümantasyonu (24 Nisan 2026)

### Docs
- `docs/ble_gatt_spec.md` yeniden düzenlendi: servis sıralaması GATT sırasına göre hizalandı, DIS ve ETS bölümleri eklendi, UUID özet tablosu güncellendi.

---

## v1.1.1 — Güvenlik ve Kararlılık Düzeltmeleri (22 Nisan 2026)

### Fixed
- **Timer Callback'te Bloklamalı NVS Çağrısı:** `scheduler_timer_cb()` içinde doğrudan yapılan `nvs_open/nvs_set/nvs_commit` çağrıları FreeRTOS timer daemon task'ını bloke ediyordu. NVS yazma işlemi ayrı bir `time_persist_task`'a taşındı.
- **Race Condition — motor_driver.c:** `s_busy` ve `s_stop_requested` değişkenleri `volatile` ile korunuyordu ancak çift çekirdekli ESP32'de atomicity garantisi yoktu. `portMUX_TYPE` spinlock ile tüm erişimler korundu.
- **Race Condition — led_indicator.c:** `s_current_mode` değişkeni `led_task` ve `led_indicator_set_mode()` arasında spinlock ile korundu.
- **BLE Status Buffer Taşması:** `ble_status_access_cb()` içindeki 512 baytlık stack buffer, maksimum 8 zamanlama girişiyle dolabiliyordu. Buffer 1024 bayta çıkarıldı.
- **JSON Brace Sayımı:** `rx_buf_has_complete_json()` string değerleri içindeki `{` `}` karakterlerini yanlış sayıyordu. `in_string` / `escaped` durum takibi eklenerek düzeltildi.
- **NVS Dönüş Değerleri:** `command_handler.c` ve `scheduler.c` içindeki `nvs_set_u32`, `nvs_set_blob`, `nvs_commit` çağrılarının dönüş değerleri artık kontrol ediliyor; hata durumunda log basılıyor.
- **Queue Dolu Durumu:** `motor_driver.c`, `command_handler.c` içindeki `xQueueSend()` dönüş değerleri kontrol ediliyor; queue dolu olduğunda mesaj kaybı `LOG_WARN` ile raporlanıyor.

### Changed
- `json_parser.c`: `strncpy` sonrası açık null sonlandırıcı eklendi; timestamp `double` → `uint32_t` cast öncesi aralık kontrolü yapılıyor; `json_build_state()` içinde `sched_count` değeri `MAX_SCHEDULES` ile sınırlandırıldı.

---

## v1.1.0 — LED Durum Göstergesi (18 Nisan 2026)

### Added
- `drivers/led_indicator.c/.h`: GPIO 13 üzerine BLE bağlantı durumu LED göstergesi eklendi.
  - **Breathing efekti:** Advertising modunda LED parlaklığı yavaşça artıp azalır (sinüzoidal).
  - **Sabit yanma:** Bağlantı kurulduğunda LED sabit olarak yanar.
  - **Otomatik geçiş:** Bağlantı kesildiğinde advertising moduna döner.
- LED kontrolü için servo motorla çakışmayacak şekilde bağımsız LEDC kanalı (Channel 1) ve zamanlayıcı (Timer 1) kullanıldı.

---

## v1.0.2 — Periyodik Zaman Kaydı (18 Nisan 2026)

### Changed
- Cihaz artık sadece manuel `set_time` alındığında değil, **her dakika başında** o anki saati NVS'e yazar. Elektrik kesintilerinde saat en fazla 1 dakika geri kalır; sistem ayağa kalktığında otomatik olarak en güncel değerden devam eder.

---

## v1.0.1 — Zaman Kalıcılığı ve Gelişmiş Durum (18 Nisan 2026)

### Added
- **Zaman Kalıcılığı:** `set_time` alındığında timestamp NVS'e kaydediliyor; cihaz resetlendiğinde son bilinen zamandan devam ediyor.
- **`clear_schedules` komutu:** Tüm zamanlamaları tek seferde temizler.

### Changed
- `get_status` yanıtına `time` (okunabilir saat) ve `schedules` (kayıtlı planlar listesi) alanları eklendi.
- Tüm bildirimlerde (`Response`, `Event`, `Status`) UNIX timestamp yerine `YYYY-MM-DD HH:MM:SS` formatında `"time"` alanı kullanılıyor.

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
  - Geçiş kuralları: RUNNING sırasında yeni komut engellenir, ERROR'da sadece reset kabul edilir
  - API: `state_get()`, `state_set()`, `state_can_feed()`, `state_to_string()`

#### 4. Core Katmanı — Command Handler
- **`main/core/command_handler.h/c`** — Merkezi komut işleme task'ı (FreeRTOS Task)
  - Feeding Service (`6E400001`) üzerinden gelen komutlar: `feed`, `set_schedule`, `update_schedule`, `remove_schedule`, `clear_schedules`, `get_status`, `reset`
  - `set_time` ayrı kanaldan: Device Time Service (`6E300001`) üzerinden alınır, yanıtı `6E300002` notify ile döner
  - Command queue'dan okur, doğrular, motor queue'ya yönlendirir

#### 5. Driver Katmanı — Servo Motor
- **`main/drivers/motor_driver.h/c`** — LEDC PWM ile servo kontrol (FreeRTOS Task)
  - 50Hz PWM, 14-bit çözünürlük, varsayılan GPIO: 4
  - Yemleme: servo aç → porsiyon × 1 saniye bekle → servo kapat
  - Güvenlik: Maksimum çalışma süresi (15 sn), emergency stop
  - State geçişlerinde Status notify gönderimi (RUNNING/IDLE)

#### 6. Zamanlayıcı
- **`main/scheduler/scheduler.h/c`** — Zaman bazlı otomatik yemleme
  - NVS'de kalıcı saklama (maksimum 8 zamanlama — v1.10.0'da 5'e düşürüldü)
  - `esp_timer` ile her 60 saniyede periyodik kontrol
  - API: `scheduler_add_entry()`, `scheduler_remove_entry()`, `scheduler_get_count()`

#### 7. BLE Servisi (NimBLE GATT)
- **`main/comm/ble_service.h/c`** — Tek modülde dört GATT servisi
  - **Device Information Service** (`0x180A`): Manufacturer, Model Number, FW/HW Revision (read-only)
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

### Derleme Sırasında Çözülen Sorunlar

| # | Sorun | Çözüm |
|---|---|---|
| 1 | `json` bileşeni bulunamadı | ESP-IDF v6.0'da cJSON harici bileşen → `idf_component.yml` ile `espressif/cjson` eklendi |
| 2 | `driver/ledc.h` bulunamadı | ESP-IDF v6.0'da driver parçalandı → `esp_driver_ledc` bileşeni kullanıldı |
| 3 | `snprintf` truncation uyarısı (-Werror) | `scheduler.c`'de NVS key buffer'ı `char key[16]` → `char key[24]` |
| 4 | `ble_gattc_exchange_mtu` undefined | Peripheral-only NimBLE'da GATT Client yok → çağrı kaldırıldı, Central MTU exchange yapıyor |
| 5 | BLE write fragmentation (20 byte limit) | Reassembly buffer eklendi: chunk'lar biriktirilip `{}` eşleşince JSON parse ediliyor |
| 6 | Status characteristic notify etmiyor | `motor_driver.c`'ye `send_status_notify()` eklendi: RUNNING/IDLE geçişlerinde otomatik notify |
| 7 | Zaman Sıfırlanma Sorunu (Power Cycle) | Cihaz resetlendiğinde zamanın 1970'e dönmesi, NVS tabanlı "Time Persistence" ile çözüldü |
| 8 | Yeni Sürücüler Derlenmiyor (Linker Error) | `led_indicator.c` gibi yeni eklenen sürücü dosyaları `main/CMakeLists.txt`'ye manuel eklenerek çözüldü |
