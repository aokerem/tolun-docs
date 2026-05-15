# TOLUN — AKILLI EV & AKVARYUM OTOMASYON PLATFORMU
## Temel Sistem Tasarımı

> **Durum:** 🧪 Test Aşaması (BLE + Servo Motor — v1.0+)  
> **Son Güncelleme:** 2026-05-13

---

## 1. Amaç
Tolun platformunun amacı; akvaryum yemleme, akvaryum aydınlatma ve ev aydınlatma sistemlerinin tek bir mobil uygulama (TolunControl) ve ortak BLE protokolü üzerinden yönetilmesini sağlamaktır. Sistem hem lokal (Bluetooth) hem de uzaktan (Wi-Fi) erişilebilir olacaktır.

---

## 2. Sistem Kapsamı

### Desteklenen Ürün Aileleri:
- **AquaFeeder** — Otomatik akvaryum balık yemleyici
- **AquaLighting** — Akvaryum aydınlatma kontrolü
- **WallLighting** — Duvar aydınlatma kontrolü
- **HomeLighting** — Ev genel aydınlatma kontrolü

### Desteklenen Özellikler:
- Manuel yemleme (mobil uygulama üzerinden)
- Otomatik yemleme (zamanlanmış)
- Aydınlatma açma/kapama ve parlaklık/renk kontrolü
- Aydınlatma zamanlama planları
- Bluetooth ile lokal bağlantı
- Wi-Fi ile uzaktan kontrol (planlanmış)
- İşlem sonrası geri bildirim ve log kaydı

---

## 3. Sistem Mimarisi

### 3.1 Donanım
- Mikrodenetleyici: ESP32-S3
- Motor: Servo (LEDC PWM)
- Yem mekanizması: Döner hazne
- Güç kaynağı

---

### 3.2 Yazılım Katmanları

1. Haberleşme Katmanı
   - Bluetooth (BLE)
   - Wi-Fi (MQTT / HTTP)

2. Kontrol Katmanı
   - Komut işleme
   - Durum makinesi (state machine)

3. Zamanlama Katmanı
   - RTC veya sistem zamanı
   - Otomatik yemleme planı

4. Donanım Sürücü Katmanı
   - Motor kontrolü

---

## 4. Çalışma Modları

### 4.1 Manuel Yemleme

#### Akış:
1. Kullanıcı mobil uygulama üzerinden porsiyon belirler
2. Komut cihaza gönderilir
3. Cihaz komutu doğrular:
   - Sistem uygun mu?
   - Son yemleme zamanı uygun mu?
4. Yemleme gerçekleştirilir
5. Sonuç mobil uygulamaya gönderilir

---

### 4.2 Otomatik Yemleme

#### Akış:
1. Kullanıcı zaman ve porsiyon ayarlar
2. Ayarlar cihazda saklanır
3. Zaman geldiğinde sistem otomatik çalışır
4. Yemleme gerçekleştirilir
5. Log oluşturulur ve uygulamaya iletilir

---

## 5. Haberleşme Yapısı

### 5.1 Bluetooth (BLE) — Mevcut Uygulama
- İlk kurulum
- Lokal kontrol
- JSON tabanlı komut/yanıt

> **Not:** Wi-Fi / MQTT desteği gelecek sürümlerde planlanmaktadır.

---

## 7. Veri Formatı (JSON)

### Yemleme Komutu:
{
  "cmd": "feed",
  "data": {
    "portion": 2
  }
}

### Geri Bildirim:
{
  "type": "feed_result",
  "mode": "manual",
  "portion": 2,
  "status": "success",
  "time": "2026-04-25 08:30:00"
}

---

## 8. Durum Makinesi (State Machine)

### Device State'leri (Cihaz Durumu):
- **IDLE** — Sistem hazır, komut bekliyor
- **RUNNING** — Yemleme işlemi devam ediyor
- **ERROR** — Hata durumu (motor sıkışması, timeout vb.)

> **Not:** Mobil uygulamada ayrıca bağlantı state'leri vardır (CONNECTED, CONNECTING, DISCONNECTED vb.)

---

## 9. Güvenlik ve Kontroller

- Aşırı yemleme koruması
- Minimum yemleme aralığı: **60 saniye** (art arda yemleme engeli)
- Günlük maksimum porsiyon limiti: **30 porsiyon**
- Motor maksimum çalışma süresi: **20 saniye** (timeout + emergency stop)
- Hata durumunda sistem durdurma

---

## 10. Hata Senaryoları

- Yem haznesi boş
- Motor sıkışması
- Bağlantı kopması
- Güç kesintisi

---

## 11. Veri Saklama

- Otomatik yemleme planı (NVS / Flash)
- Son yemleme zamanı
- Log kayıtları

---

## 12. Gelecek Geliştirmeler

- Kamera entegrasyonu
- Yapay zeka ile yemleme optimizasyonu
- Bulut veri analizi
- Mobil bildirim sistemi

---

## 13. Sistem Mimarisi - Blok Diyagram

Aşağıda sistemin yüksek seviyeli blok diyagramı verilmiştir:

```
        +----------------------+
        |     Mobil App        |
        | (BLE / Wi-Fi MQTT)  |
        +----------+-----------+
                   |
        -------------------------
        |                       |
   (BLE)                    (Wi-Fi)
        |                       |
+-------v--------+     +--------v--------+
|   ESP32 (MCU)  |<--->|   MQTT Broker   |
|                |     | (Local / Cloud) |
|  - Control     |     +-----------------+
|  - Scheduler   |
|  - State Mach. |
+-------+--------+
        |
        |
+-------v--------+
| Motor Driver   |
| (Servo)        |
+-------+--------+
        |
+-------v--------+
| Yem Mekanizması|
+----------------+
```

---

## 14. Veri Akış Diyagramı

### 14.1 Manuel Yemleme Veri Akışı

```
[Mobil App — BLE Yazma]
     |
     |  {"cmd":"feed","data":{"portion":X}}
     v
[ESP32 - Command Handler]
     |
     |-- Doğrulama (1-10, state kontrolü)
     |
     v
[Motor Control]
     |
     |-- Yemleme işlemi
     |
     v
[BLE Notify — Cevap]
     |
     v
[Mobil App]
```

---

### 14.2 Otomatik Yemleme Veri Akışı

```
[Scheduler — Zaman Kontrolü]
     |
     | (Zaman geldi — her 60sn check)
     v
[ESP32 - Command Handler]
     |
     |-- Durum kontrolü (IDLE → RUNNING)
     |
     v
[Motor Control]
     |
     v
[BLE Notify — feed_result event'i]
     |
     v
[Mobil App]
```

---

## 15. İç Yazılım Veri Akışı (ESP32)

FreeRTOS tabanlı iç yapı:

```
          +----------------------+
          |   MQTT / BLE Task    |
          +----------+-----------+
                     |
                     v
          +----------+-----------+
          |   Command Queue      |
          +----------+-----------+
                     |
     +---------------+----------------
     |                                |
     v                                v
+----+--------+              +---------+------+
| Control Task|              | Scheduler Task|
+----+--------+              +---------+------+
     |                                |
     v                                v
+----+-------------------------------+----+
|           Motor Control Task            |
+----------------------------------------+

Ek:
- State Manager
```

---

## 16. Kritik Veri Akış Kuralları

- Komutlar doğrudan motora gitmez → önce doğrulama katmanı
- Tüm sonuçlar geri bildirilir (ack zorunlu)
- State machine her akışı kontrol eder
- MQTT publish işlemleri non-blocking olmalı


---

## 17. Sonuç

Bu sistem, temel bir otomatik yemleme çözümünden daha fazlasını hedefler. Modüler yapısı sayesinde genişletilebilir ve IoT tabanlı gelişmiş uygulamalara uygun bir altyapı sunar.	
Bu diyagramlar sistemin hem donanım hem yazılım seviyesinde nasıl organize edilmesi gerektiğini gösterir. Bu yapı sayesinde sistem modüler, genişletilebilir ve hatalara karşı dayanıklı hale gelir. 