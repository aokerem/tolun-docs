# MOBILE SPECIFICATION
## TolunControl — React Native / Expo

> **Durum:** 🧪 Test Aşaması  
> **Platform:** Android (iOS desteği gelecek)  
> **Framework:** React Native + Expo  
> **İletişim:** BLE (Bluetooth Low Energy)  
> **Desteklenen Ürün Ailesi:** AquaFeeder, AquaLighting, WallLighting, HomeLighting

---

## 1. Amaç
Bu doküman, mobil uygulamanın (TolunControl) işlevsel gereksinimlerini, ekran akışlarını ve cihaz (ESP32) ile olan BLE iletişim davranışlarını tanımlar.

---

## 2. İletişim Kanalları

- **BLE (Bluetooth):** Lokal bağlantı, düşük gecikme, ana kanal
- **Wi-Fi / MQTT:** Gelecek sürümlerde planlanmaktadır

---

## 3. Temel Özellikler

- Manuel yemleme
- Otomatik yemleme planlama
- Cihaz durumu görüntüleme
- Hata bildirimleri
- Bağlantı yönetimi (BLE + Wi-Fi)

---

## 4. Ekranlar

### 4.1 Ana Ekran (Dashboard)

Gösterilen bilgiler:
- Sistem durumu (IDLE, RUNNING, ERROR)
- Son yemleme zamanı
- Günlük toplam porsiyon

Aksiyonlar:
- “Yem Ver” butonu
- Hızlı porsiyon seçimi

---

### 4.2 Manuel Yemleme Ekranı

- Porsiyon seçimi (slider / stepper)
- “Yem Ver” butonu

Akış:
1. Kullanıcı porsiyon seçer
2. Komut gönderilir
3. Loading state gösterilir
4. Sonuç gösterilir

---

### 4.3 Otomatik Yemleme (Schedule)

- Saat seçimi
- Porsiyon seçimi
- Liste halinde schedule görüntüleme

Aksiyonlar:
- Ekle / sil / güncelle

---

### 4.4 Cihaz Durumu Ekranı

- State bilgisi
- Bağlantı durumu
- Hata mesajları

---

## 5. State Management

### Mobil Uygulama (BLE Bağlantı) State’leri

- **DISCONNECTED** — Cihaz bağlı değil
- **CONNECTING** — Bağlantı kurulmaya çalışılıyor
- **CONNECTED** — Cihaza başarıyla bağlanmış
- **ERROR** — Bağlantı hatası

### Cihaz (ESP32) State’leri

Cihazın öz durumu, mobil tarafından `get_status` komutu veya `state` notify’ı ile öğrenilir:

- **IDLE** — Sistem hazır, yemleme yapılabilir
- **RUNNING** — Yemleme işlemi devam ediyor
- **ERROR** — Hata durumu (motor/timeout vb.)

---

## 6. İletişim Katmanı

### 6.1 BLE (Mevcut Kanal)

**Kullanım:**
- Cihaz taraması ve keşfi
- Yemleme komutları
- Plan yönetimi
- Durum sorgulama
- Gerçek zamanlı event dinleme

**Akış:**
1. Tarama (`startScan`) — cihaz adı önekine göre filtreleme
2. Cihaza bağlanma (`connect`)
3. GATT service/characteristic keşfi (`discoverServices`)
4. Notify karakteristiklerine abone olma (`subscribe`)
5. Komut gönderme (`write`) → Response dinleme (`notify`)

**Tarama Filtresi (Desteklenen Ürün Aileleri):**
- `AquaFeeder-` — Otomatik balık yemleme cihazı
- `AquaLighting-` — Akvaryum aydınlatma kontrolü
- `WallLighting-` — Duvar aydınlatma kontrolü
- `HomeLighting-` — Ev aydınlatma kontrolü

**Servisler:**
- Device Information Service (DIS 0x180A) — cihaz bilgisi
- Elapsed Time Service (ETS 0x183F) — uptime
- Device Time Service (0x1847) — saat ayarlama
- Feeding Service (6E400000...) — komut/yanıt/event

### 6.2 Wi-Fi / MQTT (Gelecek)

Henüz uygulanmadı. Gelecek sürümlerde uzaktan erişim için planlanmaktadır.

---

## 7. Komut Gönderme Akışı

1. UI → Command oluştur
2. Transport seç (BLE / MQTT)
3. Mesaj gönder
4. Response bekle
5. UI güncelle

---

## 8. Response Handling

- Response id ile eşleşir
- Timeout: 3 saniye
- Retry: max 2

---

## 9. Event Handling

Dinlenen event’ler:
- feed_result
- error
- state

UI güncellemeleri:
- Toast / popup
- Dashboard refresh

---

## 10. Hata Yönetimi

### UI seviyesinde:
- Bağlantı hatası
- Timeout
- Cihaz hatası

### Kullanıcıya gösterim:
- Açıklayıcı mesaj
- Retry opsiyonu

---

## 11. Bağlantı Durumları

- **Connected:** BLE bağlantısı aktif, tüm işlevler kullanılabilir
- **Disconnected:** BLE bağlı değil, local veriler gösterilir, komut gönderilemez
- **Offline:** BLE bağlantısı yok ve Wi-Fi de yok — tam offline (gelecek: MQTT fallback)

---

## 12. Güvenlik

- BLE pairing opsiyonel
- MQTT authentication

---

## 13. Logging

- Debug log (dev mode)
- Event log (opsiyonel)

---

## 14. Performans

- UI bloklanmamalı
- Async işlemler kullanılmalı

---

## 15. UX Kuralları

- Kullanıcı her aksiyon sonrası geri bildirim almalı
- Loading state net olmalı
- Hatalar anlaşılır olmalı

---

## 16. Sonuç

Bu doküman, mobil uygulamanın cihaz ile uyumlu ve stabil çalışmasını sağlayacak temel davranışları tanımlar.

