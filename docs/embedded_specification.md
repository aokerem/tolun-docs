# EMBEDDED SPECIFICATION
## Akvaryum Otomatik Yemleme Sistemi

> **Durum:** 🧪 Test Aşaması  
> **Donanım:** ESP32-S3  
> **Framework:** ESP-IDF v6.0 (FreeRTOS)  
> **İletişim:** BLE (NimBLE GATT Server)  
> **Motor:** Servo (PWM/LEDC)

---

## 1. Amaç
Bu doküman, ESP32 tabanlı gömülü sistemin yazılım mimarisi, task yapısı ve donanım entegrasyonunu tanımlar.

---

## 2. Donanım Bileşenleri

| Bileşen | Teknik Detay |
|---------|---|
| **MCU** | ESP32-S3 (dual-core, 240 MHz) |
| **Motor** | Servo (RC servo, PWM kontrol via LEDC) |
| **Motor Driver** | Entegre (PWM doğrudan GPIO'dan) |
| **Güç** | Dahili regülatör (5V giriş önerilir) |
| **LED** | Durum göstergesi (GPIO 13 üzerinden PWM) |
| **Flash/NVS** | Konfigürasyon ve plan depolama |

---

## 3. Yazılım Mimarisi

Sistem FreeRTOS tabanlıdır.

### 3.1 Task Yapısı

- Communication Task (BLE — Wi-Fi/MQTT gelecek sürümlerde planlanmaktadır)
- Command Handler Task
- Scheduler Task
- Motor Control Task
- State Manager

---

## 4. Task Detayları

### 4.1 Communication Task
- Mevcut: BLE haberleşmesini yönetir
- Gelecek: Wi-Fi/MQTT desteği planlanmaktadır
- Gelen veriyi parse eder
- Command Queue'ya iletir

---

### 4.2 Command Handler Task
- Komutları işler
- Doğrulama yapar
- State kontrolü yapar
- Motor task’a yönlendirir

---

### 4.3 Scheduler Task
- Zaman bazlı yemleme
- RTC veya sistem timer kullanır

---

### 4.4 Motor Control Task
- Motor sürme
- Porsiyon kontrolü
- Kalibrasyon

---

### 4.5 State Manager
- Sistem durumunu yönetir
- IDLE, RUNNING, ERROR

---

## 5. Veri Akışı

1. Komut alınır
2. Queue’ya yazılır
3. Command Handler işler
4. Motor Task tetiklenir
5. Sonuç oluşturulur
6. Communication Task üzerinden gönderilir

---

## 6. Queue ve Senkronizasyon

- FreeRTOS Queue kullanılır
- Mutex ile shared resource korunur
- Event Group ile task senkronizasyonu yapılır

---

## 7. Motor Kontrol

### Servo Motor (Mevcut Uygulama)
- **GPIO:** 4 (konfigüre edilebilir)
- **PWM:** 50 Hz, 14-bit çözünürlük (LEDC)
- **Mantık:** Aç (açma) → bekle (porsiyon × 1 sn) → kapat
- **Güvenlik:** 15 sn maksimum çalışma süresi, emergency stop
- **Feedback:** RUNNING/IDLE durum değişimine otomatik notify gönderimi

---

## 8. Porsiyon Kalibrasyonu

- 1 porsiyon = servo açık kalma süresi (saniye cinsinden)
- Varsayılan: 1 porsiyon = 1 saniye açık bekleme
- Kalibrasyon değeri ileriki sürümlerde NVS’de saklanabilir

---

## 9. Hafıza Yönetimi (NVS)

| Veri | Boyut | Notlar |
|------|---|---|
| Zamanlanmış planlar | ~256 byte | Maksimum 8 plan (hour/min/portion/interval) |
| Sistem saati | 8 byte | UNIX timestamp (dakika başında persist edilir) |
| Zaman dilimi | 4 byte | UTC farkı dakika cinsinden (NVS key: `tz_offset`) |
| Son durum | 4 byte | IDLE/RUNNING/ERROR |
| Besleme logu | değişken | Maksimum 20 giriş (FEED_LOG_MAX) |

---

## 10. Hata Yönetimi

- Motor sıkışması
- Timeout

Hata durumunda:
- State = ERROR
- Event gönderilir

---

## 11. Güç Yönetimi

- Brownout detection aktif
- Restart sonrası state recovery

---

## 12. OTA Güncelleme (Opsiyonel)

- Wi-Fi üzerinden firmware güncelleme

---

## 13. Logging

- UART debug log
- MQTT üzerinden log (opsiyonel)

---

## 14. Güvenlik

- Watchdog timer aktif
- Task lock-up koruması

---

## 15. State Machine (Detaylı Tanım)

### 15.1 Device State Listesi

- **IDLE** — Sistem hazır, komut bekliyor
- **RUNNING** — Yemleme işlemi devam ediyor
- **ERROR** — Hata durumu (motor sıkışması, timeout, veri hatası vb.)

---

### 15.2 State Transition Kuralları

```
IDLE -> RUNNING     (feed command)
RUNNING -> IDLE     (işlem tamamlandı)
ANY -> ERROR        (hata oluştu)
ERROR -> IDLE       (reset / recovery)
```

---

### 15.3 Kurallar

- RUNNING state sırasında yeni komut kabul edilmez
- ERROR state'te sadece reset komutu işlenir

---

## 16. Command → Action Mapping

| Command | Açıklama | Yanıt |
|---------|---|---|
| `feed` | Anlık yemleme (1-10 porsiyon) | response + feed_result event |
| `set_schedule` | Plan ekle (saat/dakika/porsiyon) | response |
| `update_schedule` | Plan güncelle (eski→yeni saat) | response |
| `remove_schedule` | Plan sil | response |
| `clear_schedules` | Tüm planları sil | response |
| `get_status` | Durum, planlar, versiyon sorgula | state JSON |
| `set_time` | Sistem saati ayarla (UNIX timestamp + tz_offset) — Device Time Service `6E300001` üzerinden | response (`6E300002`) |
| `reset` | State'i IDLE'a döndür | response |
| `clear_logs` | Besleme log geçmişini temizle | response |
| `lighting_on` | Aydınlatmayı aç | response |
| `lighting_off` | Aydınlatmayı kapat | response |

---

## 17. Zamanlama (Scheduler Detayı)

- RTC veya sistem timer kullanılır
- Schedule listesi NVS'de saklanır

### Yapı:
```
struct schedule {
  hour
  minute
  portion
  portion_interval
}
```

---

## 18. Concurrency Kuralları

- Motor Control Task tek erişimli olmalı
- Queue üzerinden komut işlenmeli
- ISR içinde ağır işlem yapılmaz

---

## 19. Error Handling (Detay)

| Hata          | Aksiyon |
|--------------|--------|
| MOTOR_ERROR  | Motor durdur |
| TIMEOUT      | İşlem iptal |

---

## 20. Watchdog ve Recovery

- Task watchdog aktif olmalı
- Sistem reset sonrası:
  - State IDLE
  - Schedule yüklenir

---

## 21. Logging (Genişletilmiş)

- Event log tutulur
- Örnek:
```
FEED_SUCCESS
FEED_FAIL
ERROR_MOTOR
```

---

## 22. Sonuç

Bu yapı ile sistem deterministic, güvenilir ve üretim seviyesinde bir embedded yazılım mimarisine ulaşır.

---

## 23. Gerçek Firmware Geliştirme Mimarisi (C / ESP32)

Bu bölüm, sistemin gerçek kod tabanında nasıl modüllere ayrılacağını tanımlar.

---

## 23.1 Proje Dizin Yapısı

```
/firmware
  /main
    main.c

  /core
    state_manager.c
    state_manager.h

    command_handler.c
    command_handler.h

  /drivers
    motor_driver.c
    motor_driver.h

  /comm
    ble_service.c
    ble_service.h

    mqtt_client.c
    mqtt_client.h

  /scheduler
    scheduler.c
    scheduler.h

  /utils
    json_parser.c
    json_parser.h

    logger.c
    logger.h

  /config
    system_config.h
```

---

## 23.2 Katmanlı Mimari

### 1. Application Layer
- main.c
- system init
- task creation

### 2. Core Layer
- state machine
- command handling

### 3. Driver Layer
- motor control

### 4. Communication Layer
- BLE
- MQTT

### 5. Utility Layer
- JSON parse
- logging

---

## 23.3 FreeRTOS Task Mapping

```
Comm Task        -> BLE RX/TX (Wi-Fi/MQTT planlanıyor)
Command Task     -> command queue processing
Motor Task       -> servo control
Scheduler Task   -> timed feeding
State Task       -> global state management
```

---

## 23.4 Queue Yapısı

### Command Queue
```c
typedef struct {
    char cmd[16];
    int portion;
    int portion_interval;
    int id;
} command_t;
```

---

## 23.5 State Management (C Implementation Model)

```c
typedef enum {
    STATE_IDLE,
    STATE_RUNNING,
    STATE_ERROR
} system_state_t;
```

---

## 23.6 Command Handler Flow

```
receive_command()
   -> validate
   -> check state
   -> push to queue
   -> execute
```

---

## 23.7 JSON Handling

- Lightweight parser kullanılır
- Heap fragmentation önlenir
- Static buffer tercih edilir

---

## 23.8 Memory Strategy

- Stack: task local işlemler
- Heap: minimal kullanım
- NVS: schedule + config

---

## 23.9 Interrupt Kullanımı

- UART RX ISR (opsiyonel)
- ISR içinde minimal işlem

---

## 23.10 Watchdog Strategy

- Task watchdog aktif
- Motor task için ayrı timeout
- Reset sonrası state recovery

---

## 23.11 Build Configuration

- Debug / Release ayrımı
- Logging seviyeleri
- Feature flags (BLE / MQTT enable)

---

## 24. Final Mimari Hedef

Bu yapı ile firmware:

- Modüler
- Test edilebilir
- Genişletilebilir
- Gerçek ürün seviyesinde

hale gelir.

