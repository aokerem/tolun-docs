# Changelog

## [1.35.0] - 2026-05-16

### Added
- `src/core/ble/BleService.ts`: `DIS_FW_REVISION_UUID` ve `DIS_HW_REVISION_UUID` import edildi. `readDIS()` artık beş paralel DIS okuması yapıyor — eskiden sadece `deviceName`, `manufacturer`, `modelNumber` okunuyordu; bunlara `fwVersion` (`0x2A26`) ve `hwVersion` (`0x2A27`) eklendi. Standart DIS karakteristikleri tek karakter farkıyla (~5–10 B) çağrı başına ve MTU bütçesi için tamamen serbest.

### Changed
- `src/core/ble/BleService.ts`: `readStatus()` artık status JSON'undan `fw_version` ve `hw_version` alanlarını **parse etmiyor**. Bu alanlar firmware v1.20.0'dan itibaren payload'da yok; `lastDeviceInfo.fwVersion` ve `hwVersion` `readDIS()` üzerinden DIS karakteristiklerinden besleniyor. Status notify yolu ~30 B daha hafif.

### Removed
- `src/core/ble/BleConstants.ts`: `FEED_LOG_CHAR_UUID` (`6E400005`) ve `LIGHTING_LOG_CHAR_UUID` (`6E400006`) export'ları kaldırıldı. Bu iki sabit hiçbir yerden import edilmiyordu — uygulama v1.29.0'dan beri log girişlerini per-index komutlar (`get_feed_log_entry` / `get_lighting_log_entry`) üzerinden çekiyor; ATT Read karakteristikleri yalancı API kontratı sergiliyordu. Firmware v1.20.0 ile birlikte ilgili GATT karakteristikleri de cihaz tarafında kaldırıldı; bu iki sabite başvurmaya çalışan kod artık ölü uçtu.

### Background
**DIS'ten okuma temizliği:** `fwVersion` / `hwVersion` aynı anda iki yoldan geliyordu — `readDIS()` standart DIS karakteristiklerinden (`0x2A24`, `0x2A29`, `0x2A00` okuyordu ama `0x2A26` / `0x2A27` okumuyordu) ve `readStatus()` JSON payload'undan. Status payload'undaki versiyon string'leri ~30 B tutuyordu; DIS karakteristikleri zaten cihazda mevcut ve standart Bluetooth SIG arayüzüyle aynı bilgiyi sunuyordu. Tek kanonik kaynak DIS'e taşındı, status payload sadeleşti — MTU bütçesi büyüyor ve mimari daha temiz.

**Ölü sabitlerin hikayesi:** `FEED_LOG_CHAR_UUID` ve `LIGHTING_LOG_CHAR_UUID` v1.14.0'da eklenmişti — firmware'in tek-ATT-Read ile tüm log JSON'unu döndüren karakteristiklerine bağlanmak için. v1.29.0'da Android `BluetoothGatt.MAX_ATTR_LEN` 512 B sınırı yüzünden per-index komut akışına geçildi (`readFeedLog()` / `readLightingLog()` metotları o sürümde silinmişti). Sabitler `BleConstants.ts`'de kaldı, kimse import etmiyordu — ileride bir geliştirici "neden 20 ayrı komut atıyoruz?" diye refaktör başlatıp aynı MTU duvarına çarpma riski vardı. Şimdi tamamen temizlendi.

---

## [1.34.4] - 2026-05-15

### Fixed
- `app/_layout.tsx`: `DevicePlanSync`, `FeedLogSync`, `LightingLogSync` üçü de aynı bağlantı boyunca status notify ile gelen güncellemeleri UI'a yansıtıyor. Önceki `syncedRef === connectedDeviceId` guard'ı ilk sync'ten sonra her güncellemeyi bloke ediyor; manuel/planlı `lighting_off` sonrası firmware NVS'e log yazıp status notify push etse de "Son Çalışma" ve çalışma geçmişi ancak BLE disconnect+reconnect ile görünüyordu. Guard, son sync'lenen array referansını izleyecek şekilde yeniden yazıldı (`lastSyncedRef`); `BleService` her fetch'te yeni array oluşturduğundan referans değişimi yeni veri demektir, aynı veri için no-op kalır. `setPlans`/`syncDeviceLogs`/`syncLightingLogs` zaten dedup'lı olduğundan idempotent çağrı zarar vermez.

### Background
Kullanıcı raporu: aydınlatma manuel olarak açılıp kapatıldığında cihaz tarafında oturum doğru kaydediliyor (firmware `handle_lighting()` `lighting_log_off()` + `send_status_update()` çağrısı yapıyor) ve `BleService` status notify zincirinde `lighting_log_count` artışını yakalayıp `fetchLightingLog`'u tetikliyor — yani `deviceInfo.lightingLog` doğru güncelleniyordu. Tek eksik nokta `LightingLogSync` effect'inin yeni veriyi `setLightingLogs`'a yazmamasıydı. Aynı bug paralel olarak `FeedLogSync` ve `DevicePlanSync`'te de mevcuttu (manuel besleme / cihaz planı değişikliği aynı bağlantıda UI'a düşmüyordu); üç tarafı birden düzeltildi.

## [1.34.3] - 2026-05-14

### Fixed
- `src/core/ble/BleContext.tsx`: `sendCommand` artık `useCallback(..., [])` ile sarılı — stabil referans. Elapsed Time karakteristiği (handle 49) her 60 s'de `setDeviceInfo` ile `BleProvider`'ı re-render ettiriyordu; `sendCommand` her render'da yeni fonksiyon olduğundan tüketicilerdeki `useCallback`/`useEffect` zincirleri tetikleniyordu (örn. `HomeScreen.tsx`'te `send` referansı değişip `lighting_on`/`lighting_off` event handler'larının yeniden koşmasına neden oluyordu).
- `src/devices/Lighting/HomeScreen.tsx`: `lighting_on` ve `lighting_off` event handler'larına `consumedEventRef` koruması eklendi — aynı `lastEvent` objesi referans eşitliği ile iki kez işlenmez hale getirildi. `addLightingLog` veya `send('lighting_off')` gibi side-effect'lerin aynı event için defalarca tetiklenmesini önler.

### Removed
- `src/devices/Lighting/HomeScreen.tsx`: `lighting_on` event handler içindeki kalıntı `set_light` gönderimi kaldırıldı. Firmware `SCHED_NVS_VERSION 6` sürümünden itibaren schedule entry'sinde r/g/b/brightness tutuyor ve `handle_lighting()` `lighting_strip_on()` öncesi rengi kendisi uyguluyor; mobile'ın ek `set_light` paketi göndermesi gereksizdi. Plan başladığında her bağlı oturumda firmware'e fazladan iki paket gönderilmesini önler.

## [1.34.2] - 2026-05-14

### Added
- `src/devices/Lighting/PlansScreen.tsx`: Çalışan aktif planı pasifleştirmeden önce **onay diyaloğu** gösteriliyor. Kullanıcı yanlışlıkla çalışan oturumu kesmesin diye plan saat aralığı içindeyken toggle'ı OFF konumuna getirdiğinde "Vazgeç / Kapat" seçenekleri sunuluyor. Kapat seçilirse mevcut akış (lighting_off + log + update_schedule) çalışır. `performToggle()` yardımcısına refactor edildi.

## [1.34.1] - 2026-05-14

### Added
- `src/devices/Lighting/PlansScreen.tsx`: Plan ekleme ve düzenleme formuna **zaman aralığı çakışma kontrolü** eklendi. Yeni/güncellenen plan, aynı cihazın diğer planlarından biriyle aralık paylaşıyorsa uyarı veriliyor ve kayıt engelleniyor. Gece geçişli planlar (örn. 22:00–06:00) desteklenir; `rangesOverlap()` helper'ı modulo 1440 dakika üzerinden çalışır. Düzenleme modunda kendisiyle karşılaştırma yapılmaz (editingId ile filtre).

## [1.34.0] - 2026-05-14

### Added
- `src/types/ble.ts`: `DeviceInfo` interface'ine `firstConnectTs` ve `totalUptimeSec` alanları eklendi.
- `src/core/ble/BleService.ts`: `readStatus()` payload'undan `first_conn_ts` ve `total_uptime_sec` alanları parse edilip `lastDeviceInfo`'ya yazılıyor.
- `src/devices/Common/DevicesScreen.tsx`: Cihaz detaylarında **İlk Bağlantı** (yerel datetime) ve **Toplam Çalışma** (gün/saat/dk biçimli) satırları gösteriliyor — sahadaki cihazların yaşam istatistiklerini uzaktan görmek için. `formatTotalUptime()` ve `formatTs()` yardımcıları eklendi.

## [1.33.6] - 2026-05-14

### Fixed
- `src/devices/Lighting/HomeScreen.tsx`: Manuel oturum sırasında plan aralığına girip plan devraldığında manuel kısım çalışma geçmişine kaydediliyor. Önceden plan ON event'i geldiğinde `runningPlan` set ediliyor ve plan oturumu başlıyordu, ama önceki manuel kısım (manuel açılış saatinden plan başlangıç saatine kadar) log'a yazılmıyordu. Suppress effect plan devralma dalına `addLightingLog(deviceId, manualStartTs, duration, isScheduled=false)` çağrısı eklendi; ardından `manualTurnOnTsRef` sıfırlanıyor.

## [1.33.5] - 2026-05-14

### Fixed
- `src/data/AppStateContext.tsx`: Boot'ta `plans` AsyncStorage'dan yüklenirken `(deviceId, time)` ve `id` bazında duplicate kayıtlar temizleniyor — eski sürümlerden kalan kopya planlar siliniyor.
- `app/_layout.tsx`: `DevicePlanSync` final dedup eklendi — `(deviceId, time)` ve `id` bazında çakışan plan tek sayılıyor; `otherPlans` da diğer cihazların kopyalarından temizleniyor.
- `src/devices/Lighting/PlansScreen.tsx` & `src/devices/Feeder/PlansScreen.tsx`: Form save (add) — aynı cihaz için aynı saatte plan varsa ikincisi eklenmiyor; ayrıca aynı milisaniyede id çakışmasını önlemek için id artımlı atanıyor.

### Background
Kullanıcı raporu: "Plan1 plan2 plan3 aynı; birini silince diğer kopyalar da siliniyor". `deletePlan` `ps.filter(p => p.id !== id)` ile çalıştığından, aynı id'yi paylaşan tüm kopyalar birlikte siliniyordu. Kök neden eski persistance'tan kalan duplicate kayıtlar olduğu için load-time dedup + sync-time dedup + form-time dedup üçü birden uygulandı.

## [1.33.4] - 2026-05-14

### Fixed
- `src/data/AppStateContext.tsx`: `addFeedLog` artık ±2 sn pencere ile dedup yapıyor; aynı milisaniyede çağrı olursa id'yi inkremente ederek çakışmayı kesin önlüyor. Önceden çift tıklama veya senkron yarışı durumunda iki entry aynı `Date.now()` id'sini paylaşabiliyordu — "two children with the same key" hatasının kaynağı.
- `src/data/AppStateContext.tsx`: Boot'ta `feedLogs` AsyncStorage'dan yüklenirken duplicate `id` veya ±10 sn ts pencere çakışan kayıtlar temizleniyor — eski sürümlerden kalan duplicate'ler atılıyor.
- `src/devices/Common/DevicesScreen.tsx`: Feed log render key'i `${log.id}-${idx}` formatında.
- `src/devices/Feeder/HomeScreen.tsx`: Aynı şekilde feed log render key'i `${log.id}-${idx}` formatında.

## [1.33.3] - 2026-05-14

### Fixed
- `src/devices/Lighting/HomeScreen.tsx`: `lighting_off` event'i için yeni effect — firmware tarafından gelen plan tetikli kapatmalar (doğal plan bitişi, scheduler kesintisi vb.) çalışma geçmişine anında ekleniyor. Event'in `start_hour/start_minute` alanı plan saatiyle eşleşirse plan-triggered olarak değerlendirilir; manuel kapatmalar (eşleşen plan yok) bu effect'te atlanır ve `doTurnOff` içindeki mevcut mantıkla işlenir. Dedup ±10 s pencere ile firmware sync'inden gelen aynı kaydı engeller.

## [1.33.2] - 2026-05-14

### Fixed
- `src/devices/Lighting/HomeScreen.tsx`: `doTurnOff()` plan branch'i — plan çalışırken anasayfadan kapatıldığında çalışma geçmişine kayıt ekleniyor (önceden yalnızca manuel mod log alıyordu, plan-triggered kapatma sadece firmware sync'ine bağlıydı).
- `src/devices/Lighting/PlansScreen.tsx`: `logPlanSession()` yardımcısı eklendi; `toggleActive`, `deletePlan` ve form save (edit ile durdurma) akışlarında plan çalışırken kapatıldığında lighting log entry oluşturuluyor.
- Plan başlangıç ts'i bugünün ilgili saatinden hesaplanıyor; gece geçişli planda (örn. 22:00–06:00) sabah saatinde dünden başlamış olarak düzeltiliyor.

## [1.33.1] - 2026-05-14

### Fixed
- `src/data/AppStateContext.tsx`: `syncLightingLogs` aynı batch içindeki tekrarlı entry'leri tespit ediyor — önceden sadece `current` ile karşılaştırıyordu; firmware'in aynı sync'te birden fazla benzer entry yollaması veya retry'ler duplicate kayıt oluşturabiliyordu. `toAdd` listesi de ±10s pencere kontrolüne dahil edildi.
- `src/data/AppStateContext.tsx`: AsyncStorage'dan yüklenen `lightingLogs` boot'ta deduplicate ediliyor — eski sürümlerden kalan tekrarlı kayıtlar temizleniyor (deviceId + startTs ±10s eşleşmesi).
- `src/devices/Common/DevicesScreen.tsx`: Lighting log render key'i `${log.id}-${idx}` formatında — duplicate kayıt durumunda React "two children with the same key" hatası fırlatmıyor.

### Changed
- `src/devices/Common/DevicesScreen.tsx`: Aydınlatma çalışma geçmişi alt etiketi "Plansız" → **"Manuel"** olarak güncellendi (planlı/plansız yerine planlı/manuel ayrımı).

## [1.33.0] - 2026-05-14

### Changed
- `src/devices/Lighting/HomeScreen.tsx`: "Son Çalışma" kartında aktif oturum için başlangıç saati + canlı süre gösteriliyor. Önceden plan tetiklediğinde başlangıç ve planlı bitiş saatleri (örn. `12:30` ve `18:00`) görünüyordu; artık başlangıç + ne kadar süredir çalıştığı (örn. `12:30` + `18 dakika`) gösteriliyor. Süre 30 saniyede bir yenileniyor.
- `src/devices/Lighting/HomeScreen.tsx`: `sessionStartTs` türev'i — manuel modda `manualTurnOnTsRef`'ten, plan modunda bugünün plan saatinden hesaplanıyor.
- `src/devices/Lighting/HomeScreen.tsx`: `fmtLightingDuration()` formatı: `<1 dakika`, `X dakika`, `X saat`, `X saat Y dakika`. Tamamlanan log girişleri de aynı format.
- `src/devices/Common/DevicesScreen.tsx`: Aydınlatma çalışma geçmişi artık `HH:MM-HH:MM` (saat aralığı) yerine `HH:MM - X dakika/saat` (başlangıç + süre) gösteriyor. Planlı/plansız ayrımı korundu (alt etiket olarak).

## [1.32.0] - 2026-05-14

### Added
- `src/devices/Lighting/HomeScreen.tsx`: "Son Çalışma" kartı manuel açıkken de yeşil görünüyor. Önceden sadece plan tetiklediğinde yeşil + "Devam ediyor" rozeti gösteriyordu; artık manuel olarak aktif olduğunda da kart yeşil + "Manuel · Devam ediyor" indikatörü gösteriyor. Zaman alanı `manualTurnOnTsRef`'tan okunur; ref boşsa "Manuel" yazısı görüntülenir.
- `lightingActive` türetilmiş değişken (`lastPlanIsRunning || manualIsRunning`) — kart stilini yeşillendirme koşulu.

## [1.31.5] - 2026-05-14

### Fixed
- `src/devices/Lighting/HomeScreen.tsx`: BT bağlantısı koptuğunda `isOn` artık otomatik `false`'a çekilmiyor. Cihazın fiziksel ışığı açık kalırken UI'da animasyonun kapanması engellendi; geçici disconnect sırasında son bilinen durum korunup auto-reconnect sonrası `deviceInfo.ledOn` ile doğal şekilde senkronlanıyor. Sadece local ref'ler (`planOverrideRef`, `suppressLedOnRef`, `manualTurnOnTsRef`) temizleniyor.

## [1.31.4] - 2026-05-14

### Fixed
- `src/core/ble/BleService.ts`: Auto-reconnect başarısında `reconnectListeners` çağrıldıktan sonra `stateListeners` yeniden tetikleniyor. Önceden `connect()` içindeki `readDIS/readUptime/readStatus` çağrıları `stateListeners`'i `connectedDeviceId` hâlâ `null` iken fire ediyordu; HomeScreen'in sync effect'i `isConnected=false` görüp `setIsOn(false)` çağırıyor, sonra connectedDeviceId güncellenirken ledOn değişikliği başka bir render'a düştüğü için kaçırılabiliyordu. Reconnect sonrası deviceInfo yeniden yayımlanarak UI tek render'da hem `isConnected=true` hem de güncel `ledOn/state/schedules`'i görüyor — aydınlatma animasyonu doğru senkronlanıyor.

## [1.31.3] - 2026-05-14

### Fixed
- `src/core/ble/BleService.ts`: Cihaz gücü kesilince auto-reconnect döngüsünün sonsuza kadar dönmesi durduruldu. İki düzeltme:
  1. `attemptReconnect`: Max deneme sayısına ulaşıldığında `reconnectAttempts` artık sıfırlanmıyor. Önceden sıfırlama, sonraki herhangi bir disconnect event'inde yeni döngüyü tetikliyordu. Artık kullanıcı manuel connect ile sayacı sıfırlamadıkça yeni döngü başlamaz.
  2. `device.onDisconnected` subscription'ı artık `disconnectSubscription`'da tutulup `cleanup()`'ta `remove()` ediliyor. Önceden stale device object'lerin orphan callback'leri başarısız `connectToDevice` denemelerinde tekrar tetikleniyor, paralel reconnect zincirleri kuruyordu.

## [1.31.2] - 2026-05-14

### Fixed
- `src/core/ble/BleService.ts`: `isConnecting` bayrağı eklendi — `connect()` çağrısı zaten devam ediyorsa ikinci paralel çağrı erken dönüyor. Önceden art arda gelen disconnect event'leri paralel connect zincirleri kuruyor, "Operation was cancelled" ve "Device already connected" kaskadına yol açıyordu.
- `src/core/ble/BleService.ts`: `attemptReconnect()` başına `reconnectTimer || isConnecting` guard'ı eklendi — bekleyen veya devam eden bir reconnect varken yeni zincir başlatılmıyor.
- `src/core/ble/BleService.ts`: `connect()` body'si `try/finally` ile sarıldı; başarısızlık durumunda `isConnecting` her zaman sıfırlanıyor.

## [1.31.1] - 2026-05-14

### Fixed
- `src/core/ble/BleService.ts`: `connect()` başına "zaten aynı cihaza bağlı" guard'ı eklendi — auto-reconnect başarılı olduktan sonra kullanıcı manuel bağlanmaya çalıştığında `manager.connectToDevice` "Device is already connected" fırlatıyor ve bunu izleyen cascade hatalı reconnect denemelerine yol açıyordu. Artık no-op döner.
- `src/core/ble/BleService.ts`: `onReconnect(callback)` listener pattern eklendi; `attemptReconnect()` başarısında tetikleniyor.
- `src/core/ble/BleContext.tsx`: `onReconnect` dinleyicisi eklendi — auto-reconnect başarısında `connectedDeviceId` ve `connectionState='connected'` UI state'i yeniden senkronize ediliyor. Önceden auto-reconnect BLE seviyesinde başarılı olsa da UI hâlâ "bağlı değil" gösteriyordu.

## [1.31.0] - 2026-05-14

### Added
- `src/core/ble/BleService.ts`: `attemptReconnect()` — beklenmedik bağlantı kopuşlarında otomatik yeniden bağlanma (exponential backoff: 1 s / 2 s / 4 s, en fazla 3 deneme). `disconnect()` çağrılırsa veya yeni bir cihaza bağlanılırsa pending reconnect timer iptal edilir.

### Fixed
- `src/core/ble/BleService.ts`: `syncTime()` artık Device Time Service notify response'unu bekliyor — `pendingTimeSync` promise pattern eklendi; firmware `success`/`error` döndürmezse veya timeout olursa hata fırlatılıyor. Önceden write işlemi tamamlandıktan sonra hiçbir doğrulama yapılmadığı için silent fail riski vardı.

## [1.30.2] - 2026-05-14

### Removed
- `src/types/ble.ts`: `CommandName` union'ından `'set_preset'` kaldırıldı — hiçbir yerde gönderilmiyordu, firmware tarafında da implementasyonu yoktu.

## [1.30.1] - 2026-05-14

### Fixed
- `src/devices/Common/DevicesScreen.tsx`: "Cihaz Saati" alanı telefon saatini (`new Date()`) gösteriyordu; `deviceInfo.deviceTime` ile değiştirildi.
- `src/devices/Lighting/HomeScreen.tsx`: `lighting_on` suppress effect'inde `plans` ve `selectedDeviceId` dependency'den eksikti — pasif plan aktif edildiğinde stale closure nedeniyle suppress yanlış çalışmaya devam ediyordu.
- `app/_layout.tsx`: `LightingLogSync` bileşenine `syncedRef` pattern eklendi — `FeedLogSync` ve `DevicePlanSync` ile tutarlı hale getirildi; aynı cihaz için tekrar bağlantıda `syncLightingLogs` birden fazla kez çağrılmıyordu.
- `src/devices/Feeder/HomeScreen.tsx`: `feed` komutundan gereksiz `feed_type: 'quick'` alanı kaldırıldı — firmware tarafından kullanılmayan dead payload.

## [1.30.0] - 2026-05-11

### Added
- `src/data/AppStateContext.tsx`: `addLightingLog(deviceId, startTs, durationS, isScheduled?)` fonksiyonu eklendi — manuel aydınlatma kapama sonrası BLE zincirini beklemeden anında yerel geçmiş kaydı oluşturur; ±10 s deduplication ile BLE sync'ten gelen girişlerle çakışmaz.
- `src/devices/Lighting/HomeScreen.tsx`: `manualTurnOnTsRef` eklendi — `lighting_on` başarılı olunca UNIX timestamp kaydedilir; kapama sırasında süre hesaplanmasına olanak tanır.

### Changed
- `src/devices/Lighting/HomeScreen.tsx`: "Son Çalışma" kartı artık `lightingLogs` üzerinden gerçek geçmişi gösteriyor — plan tabanlı saatten bağımsız olarak manuel ve planlı oturumları (saat + süre + "Manuel" etiketi) yansıtıyor.
- `src/devices/Lighting/HomeScreen.tsx`: `doTurnOff` (plan olmayan yol) — `lighting_off` başarıyla tamamlanınca `addLightingLog` çağrılıyor; "Son Çalışma" anında güncelleniyor.
- `src/devices/Lighting/HomeScreen.tsx`: `handlePreset` preset ile açmada ve `handleToggle` manuel açmada da turn-on timestamp'i kaydediliyor.
- `src/devices/Lighting/HomeScreen.tsx`: Toggle kilidi 1 saniyeden **3 saniyeye** çıkarıldı — hızlı on/off döngüsünü önler.
- `src/core/ble/BleService.ts`: `readStatus()` schedule / feed log / lighting log fetch'lerini artık count değişmediğinde atlıyor — `lighting_off` sonrası status notify her tetiklendiğinde gereksiz `get_schedule` trafiği oluşmuyordu.
- `src/core/ble/BleService.ts`: `lastScheduleCount`, `lastFeedLogCount`, `lastLightingLogCount` başlangıç değerleri `−1` olarak ayarlandı; disconnect'te sıfırlanıyor — yeniden bağlantıda tam ilk çekim garantileniyor.
- `src/core/ble/BleContext.tsx`: `refreshLightingLog` metodu interface'e eklendi (tanısal kullanım için).

### Fixed
- Manuel aydınlatma açma/kapama sonrası "Son Çalışma" kartı güncellenmiyordu.
- Manuel aydınlatma oturumları çalışma geçmişine kaydedilmiyordu.

## [1.29.0] - 2026-05-11

### Added
- `src/core/ble/BleService.ts`: `fetchFeedLog(count)` / `doFetchFeedLog(count)` metotları eklendi — `get_feed_log_entry` komutunu indeks bazlı döngüyle çağırır, her response ~160 B olduğundan Android 512 B ATT sınırına takılmaz.
- `src/core/ble/BleService.ts`: `fetchLightingLog(count)` / `doFetchLightingLog(count)` metotları eklendi — `get_lighting_log_entry` komutunu indeks bazlı döngüyle çağırır; her response ~145 B.
- `src/core/ble/BleService.ts`: `fetchFeedLogInFlight` ve `fetchLightingLogInFlight` reentrancy guard'ları eklendi — paralel status notify'larında çift döngü aynı cmd key'ini paylaşıp "superseded" hatasına yol açmasın diye.
- `src/types/ble.ts`: `CommandName` listesine `'get_feed_log_entry'` ve `'get_lighting_log_entry'` eklendi.

### Changed
- `src/core/ble/BleService.ts`: `readStatus` artık status payload'undaki `feed_log_count` ve `lighting_log_count` alanlarını parse ediyor; `fetchSchedules` sonrasında `fetchFeedLog` ve `fetchLightingLog` zincirleme çağrılıyor.
- `src/core/ble/BleService.ts`: Connect akışından `readFeedLog()` ve `readLightingLog()` kaldırıldı — her ikisi artık `readStatus` zincirinin içinde per-index komutlarla çekiliyor.
- `src/core/ble/BleService.ts`: Status notify handler'ı sadeleştirildi — `readLightingLog()` ayrı çağrısı kaldırıldı, `readStatus()` tek çağrısı schedules + feed log + lighting log'u da kapsıyor.
- `src/core/ble/BleService.ts`: `readFeedLog()` ve `readLightingLog()` ATT read metotları kaldırıldı (FEED_LOG_CHAR_UUID / LIGHTING_LOG_CHAR_UUID artık kullanılmıyor).

### Fixed
- Manuel aydınlatma açma/kapama sonrasında `readLightingLog` çağrısı Android 512 B sınırında kesiliyordu (`JSON Parse error: Unexpected end of input`). Per-index `get_lighting_log_entry` chunking ile çözüldü (Firmware v1.17.0 ile birlikte).

## [1.28.0] - 2026-05-11

### Added
- `src/core/ble/BleService.ts`: `fetchSchedules(count)` metodu eklendi — `readStatus`'tan dönen `schedule_count` kadar `get_schedule` komutu sıralı gönderir, response.data'ları biriktirip `lastDeviceInfo.schedules`'a yığar. Her transaction MTU notify limitinin altında olduğundan Android'in 512 B `BluetoothGatt.MAX_ATTR_LEN` sınırına takılmaz.
- `src/types/ble.ts`: `CommandName` listesine `'get_schedule'` eklendi.

### Changed
- `app/_layout.tsx`: `DevicePlanSync`, `FeedLogSync`, `LightingLogSync` `useEffect` deps array'ine `devices` eklendi — forget + readd akışında `connectedDeviceId` ve `devices` farklı render cycle'larında set olunca stale closure planlar/logları kaçırıyordu.
- `src/devices/Common/DevicesScreen.tsx`: "Hakkında" bölümünde `Donanım` ve `Yazılım` rowları artık `device.hwVersion`/`fwVersion` (async setDevices ile geç dolan) yerine canlı `deviceInfo.hwVersion`/`fwVersion`'dan okuyor (forget + readd sonrası boş görünmüyor).
- `src/core/ble/BleService.ts`: `readStatus` artık status payload'undaki `schedule_count` alanını parse ediyor; schedules artık status read'i içinde inline gelmiyor — onun yerine `fetchSchedules` zincirleme çağrılıyor.
- `src/core/ble/BleService.ts`: Connect akışı sadeleştirildi — `Promise.all` paralel okumaları kaldırıldı, hepsi sıralı await ediliyor (NimBLE `GATT_MAX_PROCS=4` sınırını zorlamamak için). Tek bir `readStatus` çağrısı yeterli (fetchSchedules zaten o zincirin içinde).
- `src/core/ble/BleService.ts`: Status notify trigger'ı ve `lighting_off` response handler'ı da paralel `readStatus + readLightingLog` yerine sıralı await kullanıyor.
- `src/core/ble/BleService.ts`: `readStatus`, `readFeedLog`, `readLightingLog` 200 ms aralıkla 2 retry yapıyor; tüm denemeler başarısız olursa `console.warn` ile log basıyor (eskiden sessiz yutuluyordu).
- `src/core/ble/BleService.ts`: Bağlantı sonrası gerçekten pazarlık edilen ATT MTU değeri `console.log` ile yazılıyor (debug için).

### Fixed
- Forget + yeniden ekleme akışında 3+ planlı cihazlarda planlar mobil ekranda görünmüyordu — status JSON'u Android stack'inin 512 B sınırında kesiliyor (`JSON Parse error: Unexpected end of input`), parse hatası schedules'ı `null` bırakıyordu. Per-index `get_schedule` chunking mimarisi ile çözüldü (Firmware v1.16.0 ile birlikte).

## [1.27.0] - 2026-05-10

### Changed
- `Lighting/HomeScreen.tsx`: Seçili preset tekrar tıklandığında cihaza komut gönderilmiyor — `handlePreset` başına erken çıkış eklendi.
- `Lighting/HomeScreen.tsx`: "Son Çalışma" kartı artık aktif olmayan planları da kapsıyor — plan düzenlendikten sonra karttan kaybolma sorunu giderildi.
- `Lighting/HomeScreen.tsx`: Plan çalışırken "Son Çalışma" kartı yeşil kenarlık + açık yeşil arka plan + "Devam ediyor" göstergesiyle güncelleniyor; bitiş saati "ne zaman kapanacak" anlamına geliyor.
- `Lighting/HomeScreen.tsx`: `lighting_on` event alındığında `setRunningPlan` ve `setIsOn` aynı React batch'inde set ediliyor — kart güncellemesi aydınlatma animasyonuyla senkron.
- `Lighting/HomeScreen.tsx`: `lastPlan`, `runningPlan` set edildiğinde donmuş `nowMin` beklemeden doğrudan ondan türetiliyor — plan başladığı anda kart güncelleniyor.
- `Lighting/HomeScreen.tsx`: `nextPlan`, aktif çalışan planı filtreden çıkarıyor — plan başladıktan sonra "Sonraki Çalışma" kartında görünmeye devam etme sorunu giderildi.
- `Lighting/PlansScreen.tsx`: Varsayılan plan etiketi "Çalışma Planı X" → "Plan X" olarak güncellendi.

## [1.26.0] - 2026-05-10

### Changed
- `src/devices/Common/DevicesScreen.tsx`: Aydınlatma cihazlarında "Kayıtları Temizle" butonu artık yalnızca yerel geçmişi değil, cihaza `clear_logs` komutu göndererek NVS'deki kayıtları da temizler; bağlı değilken buton yalnızca yerel kaydı siler.
- `src/devices/Common/DevicesScreen.tsx`: Fabrika sıfırlama akışına `clearLightingLogs` eklendi — sıfırlama sonrası aydınlatma çalışma geçmişi de temizlenir.

## [1.25.0] - 2026-05-10

### Added
- `src/types/ble.ts`: `DeviceLightingEntry` arayüzü eklendi (`ts`, `duration_s`, `is_scheduled`); `DeviceInfo`'ya `lightingLog` alanı eklendi; `DeviceSchedule`'e `active` ve `label` alanları eklendi.
- `src/types/index.ts`: `LightingLog` arayüzü eklendi (`startTs`, `durationS`, `isScheduled`).
- `src/data/AppStateContext.tsx`: `lightingLogs` state, `syncLightingLogs`, `clearLightingLogs` eklendi; `@tolun:lightinglogs` anahtarıyla AsyncStorage kalıcılığı.
- `app/_layout.tsx`: `LightingLogSync` bileşeni eklendi — bağlantıda ve `deviceInfo.lightingLog` değiştiğinde otomatik senkronizasyon.
- `src/devices/Common/DevicesScreen.tsx`: Aydınlatma cihazları için **"Çalışma Geçmişi"** bölümü eklendi — her session `HH:MM-HH:MM Planlı/Plansız` formatında gösterilir.

### Changed
- `src/core/ble/BleService.ts`: `readStatus` içinde `lighting_log` parse edilerek `lastDeviceInfo.lightingLog`'a yazılıyor; `lighting_off` response başarılı gelince 300 ms sonra `readStatus()` yeniden çağrılır (BLE stack çakışması önlenir).
- `src/devices/Common/DevicesScreen.tsx`: Fish Feeder dışı cihazlarda geçmiş başlığı `"Besleme Kayıtları"` → `"Çalışma Geçmişi"` olarak güncellendi; Kayıtları Temizle de ilgili log tipini temizler.

## [1.24.0] - 2026-05-10

### Added
- `src/types/ble.ts`: `set_light` komut tipi eklendi.

### Changed
- `Lighting/PlansScreen.tsx`: Plan düzenleme ekranında slider (parlaklık, ton, doygunluk) ve preset geçişlerinde canlı BLE gönderimi kaldırıldı; değişiklikler yalnızca Kaydet butonuyla uygulanır.
- `Lighting/PlansScreen.tsx` & `Lighting/HomeScreen.tsx`: `set_color` + `set_brightness` çiftleri tek `set_light` paketiyle değiştirildi (atomik renk+parlaklık güncelleme).
- `Lighting/PlansScreen.tsx`: Plan aktifleştirildiğinde sıra düzeltildi — `set_light` önce, `lighting_on` sonra.
- `Lighting/PlansScreen.tsx`: Yeni plan aktif çalışma aralığına düşünce `lighting_on` da gönderilir; anasayfa animasyonu doğru tetiklenir.
- `Lighting/HomeScreen.tsx`: Renk önizleme kutusu ve RGB değerleri plan aktifken de tam görünürlükte gösterilir (opacity wrapper'ından çıkarıldı).

## [1.23.0] - 2026-05-10

### Added
- `Feeder/HomeScreen.tsx`: Yemleme durumu için `actionState` FSM eklendi (`idle → feeding → fed → interval`). Porsiyon ilerleme göstergesi (ör. `✓ Yemlendi! (2/3)`) feed butonunda görünüyor.
- `Feeder/HomeScreen.tsx`: Cihaz bağlı değilken yemle butonuna basınca 2,5 saniyelik kayan "bağlı değil" banner gösteriliyor.
- `Feeder/PlansScreen.tsx`: Plan etiketi boş bırakılırsa otomatik `Plan N` atanıyor.

### Changed
- `Feeder/HomeScreen.tsx`: Yemleme sonrası geri sayım (`feedCooldown`) kaldırıldı; yerini `actionState` FSM aldı. Hata banner'ı ekranın altına kayan overlay olarak taşındı.
- `Feeder/PlansScreen.tsx`: Ekle butonu maks. 5 plan sınırını da kontrol ediyor; düzenle/sil butonları `canAdd` yerine `isSelectedConnected` kontrolüne geçildi.
- `Lighting/HomeScreen.tsx`: Plan aktif bildirimi ve hata banner'ı satır içinden ekranın altına kayan overlay'e taşındı.
- `Lighting/PlansScreen.tsx`: `set_schedule` / `update_schedule` komutlarına `label` ve `active` alanları eklendi; ekle butonu maks. 5 plan sınırını kontrol ediyor; düzenle/sil butonları `isSelectedConnected` kontrolüne geçildi.
- `app/_layout.tsx`: `useBannerStyle` hook'u tüm banner'lar için sabit beyaz/kırmızı stile indirgendi; `DeviceNotConnectedBanner` geçici olarak devre dışı bırakıldı.
- Tüm ekranlarda banner stili beyaz arka plan + kırmızı kenarlık olarak standartlaştırıldı (`HomeScreen`, `PlansScreen`, `_layout.tsx`).
- `src/data/AppStateContext.tsx`: Yükleme sırasında eski mock cihazlar ('Çocuk Odası Işığı', 'Çocuk Odası', 'AquaFeeder') filtreleniyor; `INITIAL_DEVICES` ile birleştirme mantığı iyileştirildi.
- `src/data/mockData.ts`: Tek gerçek cihaza indirgendi (AL-001 'AquaLighting'); `INITIAL_PLANS` ve `INITIAL_FEED_LOGS` boşaltıldı.

## [1.22.0] - 2026-04-29

- `Lighting/HomeScreen.tsx`: "Çalışma Planı Aktif" uyarısı ekranın altından kaldırılıp "Sonraki Çalışma" kartları ile kontrol paneli arasına taşındı.
- `Lighting/HomeScreen.tsx`: Plan uyarısı metni "Çalışma [Ad] aktif: ss:dd-ss:dd" formatına güncellendi ve etiket girilmediği durumlar için otomatik numaralandırma (Plan 1, Plan 2...) desteği eklendi.
- `Lighting/HomeScreen.tsx` & `Lighting/PlansScreen.tsx`: Ekranlar arası geçişte kartların dikey hizasının bozulmaması için `marginTop` ve `marginBottom` değerleri senkronize edildi.
- `Lighting/HomeScreen.tsx`: Kartlar, uyarı banner'ı ve kontrol paneli arasındaki dikey boşluklar 5'er birim olacak şekilde (toplam 10 birim gap) standartlaştırıldı.
- `Lighting/PlansScreen.tsx` & `Feeder/PlansScreen.tsx`: Plan aktiflik durumu (`active`) değiştirildiğinde veya plan kaydedildiğinde cihazla anında senkronizasyon sağlanıyor (`update_schedule`).
- `app/_layout.tsx`: Cihazdan gelen plan verileri senkronize edilirken `active` (aktiflik) alanı artık dikkate alınıyor; "Cihazı Unut" sonrası tekrar eklemelerde durum korunuyor.
- `Lighting/PlansScreen.tsx` & `Feeder/PlansScreen.tsx`: Maksimum plan sayısı **5** ile sınırlandırıldı.
- `Lighting/PlansScreen.tsx` & `Feeder/PlansScreen.tsx`: Plan limitine ulaşıldığında "+" butonuna basılırsa "En fazla 5 plan oluşturabilirsiniz" uyarı banner'ı gösteriliyor.
- `DevicesScreen.tsx`: **"Fabrika Ayarlarına Dön"** fonksiyonu eklendi; cihazdaki tüm planları, geçmişi temizler ve sistemi sıfırlar.

### Fixed
- `app/_layout.tsx`: `PlanConnectSync` bileşeni kaldırıldı; planların o anki saate göre otomatik pasife alınması hatası (hatalı tasarım) giderildi.

## [1.21.0] - 2026-04-29

- `Lighting/HomeScreen.tsx`: Manuel müdahalelerde (güç butonu veya preset seçimi) cihazın plan takvimi tarafından otomatik ezilmesini önlemek için `planOverrideRef` koruması eklendi.
- `Lighting/HomeScreen.tsx`: Ampul simgesine `interactionScale` animasyonu eklendi; sürgüler hareket ettiğinde ampul "nabız" efektiyle tepki verir.
- `Lighting/HomeScreen.tsx` & `Lighting/PlansScreen.tsx`: Manuel parlaklık veya renk ayarı yapıldığında "Özel" mod otomatik seçilir ve liste bu moda kendiliğinden kayar (`scrollToCustom`).

### Changed
- `Lighting/HomeScreen.tsx`: Varsayılan başlangıç modu "Gündüz" yerine **"Gün Doğumu"** (Hue: 28, Sat: 72, Bri: 52) olarak güncellendi.
- `Lighting/HomeScreen.tsx`: Güç butonuna (`handleToggle`) ve hazır modlara (`handlePreset`) basıldığında arayüzün anında tepki vermesi için asenkron yapı optimize edildi; cihazdan yanıt beklemeden ampul yanar.
- `Lighting/HomeScreen.tsx` & `Lighting/PlansScreen.tsx`: Cihaza gönderilen renk komutlarında `L=50` (baz renk) kullanılacak şekilde güncellendi; parlaklık ölçeklendirmesi tamamen cihaz donanımına bırakıldı (çift ölçekleme hatası giderildi).
- `Lighting/PlansScreen.tsx`: Yeni plan oluşturma formu varsayılan olarak "Gün Doğumu" preset değerleriyle açılıyor.

## [1.20.0] - 2026-04-28

### Added
- `Lighting/PlansScreen.tsx`: Plan düzenleme formunda parlaklık, ton veya doygunluk slider'ı sürüklenirken — ya da preset seçilirken — şu anki saat planın başlangıç–bitiş aralığındaysa `set_color` + `set_brightness` komutları cihaza anlık gönderiliyor.
- `Lighting/PlansScreen.tsx`: Yerel `Slider` bileşenine `onDone` prop'u ve `onPanResponderRelease` handler'ı eklendi — parmak bırakıldığında son değer kesin olarak iletiliyor.

### Changed
- `Lighting/PlansScreen.tsx`: Slider'lardaki canlı renk gönderimi debounce → throttle (150 ms) + `onDone` (bırakışta) olarak güncellendi; HomeScreen ile aynı pattern.

## [1.19.0] - 2026-04-28

### Added
- `app/_layout.tsx`: `PlanConnectSync` bileşeni eklendi — bağlantıda aktif aydınlatma planlarının geçerlilik aralığı kontrol edilir; mevcut dakika plan aralığı dışındaysa plan pasife alınır.
- `app/_layout.tsx`: `useBannerStyle` hook'u çıkarıldı — tüm banner'ların dark/light tema renkleri tek noktadan yönetiliyor.
- `Lighting/HomeScreen.tsx`: Son/Sonraki Çalışma kartları eklendi — aktif planlardan önceki ve bir sonraki zaman dilimi gösteriliyor.
- `Lighting/HomeScreen.tsx`: Plan çalışırken plan banner'ı gösteriliyor; preset, parlaklık ve renk kontrolleri pasifleşiyor.
- `Lighting/HomeScreen.tsx`: Kapatma sırasında aktif plan tespit edilirse onay uyarısı gösteriliyor; kullanıcı onaylarsa plan pasife alınıyor.
- `Lighting/HomeScreen.tsx`: `lighting_on` BLE event'i alındığında tetiklenen planın hue/sat/brightness/preset ayarları UI'ya uygulanıyor.
- `Lighting/HomeScreen.tsx`: Cihaz bağlı değilken slider/buton etkileşimine mini bağlantı uyarı banner'ı eklendi.
- `Lighting/PlansScreen.tsx`: Plan formuna renk/parlaklık/preset ayarları eklendi — her plana ayrı aydınlatma rengi atanabiliyor.
- `Lighting/PlansScreen.tsx`: Plan silinmesi/pasifleştirilmesi sırasında plan çalışıyorsa cihaza `lighting_off` gönderiliyor.
- `Lighting/PlansScreen.tsx`: `isPlanRunning()` yardımcı fonksiyonu ve yerel `Slider` bileşeni eklendi.
- `Icon.tsx`: `pine`, `plumfruit`, `sunrise`, `crescent` ikonları eklendi.
- `src/types/ble.ts`: `SCHED_KIND_LIGHTING` sabiti eklendi.
- `src/types/index.ts`: `FeedPlan`'a `hue`, `sat`, `brightness`, `preset` alanları eklendi.

### Changed
- `Lighting/HomeScreen.tsx`: Preset listesi yeniden düzenlendi ve genişletildi — 13 preset: Gün Doğumu, Gündüz, Gün Batımı, Ay Işığı, Bitki, Orman, Mercan, Mürdüm, Gül Kurusu, Fuşya, Turkuaz, Alev, Özel.
- `Lighting/HomeScreen.tsx`: `isOn` durumu bağlantıda `deviceInfo.ledOn` ile senkronize ediliyor; `lighting_on`/`lighting_off` BLE event'leri anlık günceller.
- `app/_layout.tsx`: `DevicePlanSync` aydınlatma planları için `timeEnd` alanını da senkronize ediyor; cihaz stale NVS döndürebileceğinden mevcut yerel `timeEnd` korunur.
- `app/_layout.tsx`: `DeviceNotConnectedBanner` tab dışındaki ekranlarda (welcome vb.) gösterilmiyor.
- `app/_layout.tsx`: Banner'lar dark mode'da uygun arka plan rengi alıyor.
- `BleService.ts`: Status okumada `led_on` alanı parse edilerek `deviceInfo.ledOn`'a yazılıyor. `lighting_on`/`lighting_off` BLE event'leri `handleEvent()`'te yakalanarak `ledOn` anlık güncelleniyor. `syncTime` ardından `readStatus` tekrar çağrılıyor — zaman senkronizasyonu sonrası güncel durum alınıyor.
- `BleService.ts`: Uptime bildiriminde `time` alanı parse edilerek `deviceTime`'a yazılıyor. Timestamp hesabı `Math.floor` → `Math.round`.
- `DevicesScreen.tsx`: Uptime formatı sadeleştirildi — saniye gösterilmiyor; saat/dakika formatı netleştirildi. Cihaz saati gösterimi yerel sistem saatine dönüştürüldü. "Yeniden Başlat" butonu daha uygun konuma taşındı.
- `BottomSheet.tsx`: `maxHeight: screenHeight * 0.92` sınırı eklendi — uzun içeriklerde ekran taşması önleniyor.

## [1.18.0] - 2026-04-27

### Added
- `Icon.tsx`: `heart`, `star`, `droplet`, `flame`, `snowflake`, `bulb` ikonları eklendi.
- `Lighting/HomeScreen.tsx`: 5 yeni preset modu eklendi — Lavanta, Pembe, Altın, Nane, Alev (toplam 13 preset: 12 hazır + Özel).
- `Lighting/PlansScreen.tsx`: Çalışma saat aralığı için drum-roll tarzı `WheelPicker` bileşeni — saat/dakika her biri bağımsız dikey kaydırmayla ayarlanır (tepede `HH : MM – HH : MM` tek satır düzeni).

### Changed
- `Lighting/HomeScreen.tsx`: Güç butonu ikonu `lightning` → `bulb` olarak değiştirildi.
- `Lighting/HomeScreen.tsx`: Güç kartı yüksekliği küçültüldü (`POWER_SIZE=84`, `paddingTop:20`).
- `Lighting/HomeScreen.tsx`: GlowRing merkezi ikon ile hizalandı (`top:10`, çap `POWER_SIZE+20`).
- `Lighting/HomeScreen.tsx`: Parlaklık ve renk (ton/doygunluk) slider'ları sürüklenirken 150 ms throttle ile `set_brightness` / `set_color` gönderir — gerçek zamanlı LED geri bildirimi.
- `Lighting/HomeScreen.tsx`: Slider dokunuşta başa/sona atlama engellendi — sadece mevcut konumdan sürükleme (drag-only, `gestureState.dx + startValue`).
- `BleService.ts`: Bekleyen komut varken aynı komut tekrar gönderildiğinde önceki reddedilmek yerine `superseded` ile iptal edilip yenisi kuyruğa alınır.

### Removed
- `Lighting/PlansScreen.tsx`: `NumSlider` ve yatay slider düzeni kaldırıldı; `WheelPicker` ile değiştirildi.

## [1.17.0] - 2026-04-27

### Added
- `Icon.tsx`: `leaf`, `sunset`, `palette`, `waves`, `stars` ikonları eklendi.
- `Lighting/HomeScreen.tsx`: 3 yeni preset modu eklendi — Okyanus (mavi, %60), Mercan (mavi-mor, %72), Ay Işığı (loş mavi, %8).

### Changed
- `Lighting/HomeScreen.tsx`: Preset seçimi artık `set_preset` yerine `set_color` + `set_brightness` kombinasyonu gönderiyor; ışık kapalıyken preset seçilirse `lighting_on` da gönderilir.
- `Lighting/HomeScreen.tsx`: `presetChip` üzerindeki `elevation` / `shadow*` prop'ları kaldırıldı — Android'de beyaz dikdörtgen artefakt oluşturuyordu.
- `BleService.ts`: Aynı komut zaten pending'deyken tekrar gönderim engellendi.
- `Lighting/HomeScreen.tsx`: Toggle butonu için `toggleLockRef` kilidi eklendi — komut yanıtlanana kadar ikinci dokunuş yoksayılır.

## [1.16.0] - 2026-04-27

### Changed
- `Lighting/HomeScreen.tsx`: Tasarım mockup'a göre komple yeniden yazıldı.
  - **Header:** Feeder ekranıyla aynı hizaya getirildi — "TolunControl" başlığı, bağlantı pill'i, border yok.
  - **Güç kartı:** 100 px dairesel buton; `Animated.loop` ile pulsing ambient glow + ring efekti; alt status strip (LED | Parlaklık | Mod) üç sütunlu.
  - **Preset modlar:** Yatay scroll; her chip dikey ikon + etiket düzeni (Gündüz/Gece/Bitki/Gün Batımı/Özel); aktif chip gölge + renk efekti.
  - **Parlaklık kartı:** Slider + 20 parçalı dot bar göstergesi.
  - **Renk kontrolü kartı:** Renk önizleme karesi + R/G/B sayısal değer hücreleri + Ton (rainbow gradient) + Doygunluk slider.
- `app/_layout.tsx`: Tüm banner'lar birleşik koyu stile (yarı şeffaf arka plan, renkli nokta, renkli metin) dönüştürüldü.
  - `ConnectBanner`, `DisconnectBanner`, `ConnectErrorBanner`, `BluetoothOffBanner` aynı base style kullanıyor.
  - `DeviceNotConnectedBanner` eklendi — seçili cihaz bağlı değilken bottom nav üzerinde gösterilir.
  - Tüm banner'lar 3 saniye sonra otomatik kapanır.

## [1.15.0] - 2026-04-26

### Changed
- `Lighting/HomeScreen.tsx`: Kontrol paneli tamamen yeniden tasarlandı. Güç, parlaklık, renk ve hazır modlar artık tek bir birleşik kart içinde; planlar ve kayıtlar ikincil konuma taşındı.
  - **Status bar:** Panelin üstünde daima görünür; cihaz durumu (AÇIK/KAPALI), parlaklık yüzdesi ve aktif mod adını gösterir; açıkken renkli nokta glow efekti uygular.
  - **Güç butonu:** 88 px dairesel buton; `Animated` ile `glowAura` (yumuşak halo) + `glowRing` (renkli çerçeve) katmanlı glow animasyonu — açılışta 450 ms ease-in, kapanışta ease-out.
  - **Parlaklık slider:** `darkGradient` modu (koyu → accent rengi SVG gradyanı) ile tam aralık görünür; bırakınca `set_brightness` komutu gönderilir.
  - **Renk & Beyaz sliderları:** Yalnızca RGBW cihazlarda; `onDone` callback'i güncel ratio değerini doğrudan alır (stale closure sorunu giderildi).
  - **Hazır mod chips:** Yatay ScrollView içinde; aktif chip'te preset renginde glow nokta göstergesi.
- `Slider` bileşeni: `onDone(finalRatio)` imzasına geçildi; `onRatioRef` / `onDoneRef` ile her render'da callback güncelleniyor — PanResponder'ın stale closure hatası giderildi.

## [1.14.0] - 2026-04-26

### Added
- `Lighting/HomeScreen.tsx`: Aydınlatma cihazları için temel kontrol arayüzü eklendi.
  - Aç/Kapat butonu — `lighting_on` / `lighting_off` BLE komutu; açıkken yeşil glow efekti.
  - Parlaklık slider'ı (PanResponder) — bırakınca `set_brightness` komutu (0–100).
  - Ton (hue) slider'ı — `react-native-svg` LinearGradient ile gökkuşağı gradyanı; bırakınca `set_color` komutu.
  - Beyaz kanal slider'ı — yalnızca `AquaLighting` (RGBW) cihazlarda görünür; bırakınca `set_color` komutu.
  - Renk önizleme çemberi — mevcut ton + beyaz kanalı karışımını gösterir.
  - 6 hazır mod butonu (flex-wrap grid): `daylight`, `night`, `plant`, `moonlight`, `sunset`, `ocean` — `set_preset` komutu; seçili moda renkli nokta göstergesi.
- `types/ble.ts`: `CommandName`'e `lighting_on`, `lighting_off`, `set_brightness`, `set_color`, `set_preset` eklendi.
- `docs/interface_spec.md`: Bölüm 11 (set_brightness), 12 (set_color — r/g/b/w), 13 (set_preset — 6 preset değeri) eklendi.

## [1.13.0] - 2026-04-26

### Added
- `src/components/DeviceChipSelector.tsx`: Tüm ekranlar için ortak cihaz seçici bileşeni oluşturuldu. Seçim durumu, scroll-to-selected (sekme geçişinde) ve yeni cihaz eklenince liste sonuna otomatik scroll mantığını içeriyor. İsteğe bağlı `getSubtitle` prop'u ile ekrana özel alt başlık desteği (Planlar ekranında plan sayısı/aktif plan sayısı).
- `AppStateContext`: `selectedDeviceId` / `setSelectedDeviceId` paylaşımlı state olarak eklendi. Seçilen cihaz artık tüm sekmeler arasında senkron; cihaz silindiğinde otomatik olarak ilk cihaza düşüyor.
- `BleConstants.ts`: `CONNECT_TIMEOUT_MS = 3000` sabiti eklendi.

### Changed
- `DevicesScreen`, `Feeder/HomeScreen`, `Lighting/HomeScreen`, `Feeder/PlansScreen`: Yerel `selectedDeviceId` state'leri ve tekrarlanan chip JSX kodu kaldırıldı; `<DeviceChipSelector />` bileşeni kullanılıyor. Herhangi bir sekmede seçilen cihaz diğer sekmelerde de seçili görünüyor.
- `BleService.ts`: `connectToDevice` çağrısına `{ timeout: CONNECT_TIMEOUT_MS }` eklendi; bağlantı denemesi artık en fazla 3 saniye sürüyor.

## [1.12.0] - 2026-04-26

### Changed
- `Feeder/PlansScreen.tsx`, `Lighting/PlansScreen.tsx`: Saat seçici drum-roll (ScrollView snap) yerine sürüklemeli slider (PanResponder) + rakam göstergesiyle değiştirildi.
- `TimeRangePicker`: Tek satır HH:MM–HH:MM düzeni, başlangıç ve bitiş için iki ayrı satıra ayrıldı; ayırıcı `–` satırlar arasında ortalı metin olarak gösteriliyor.
- Slider vibrasyon geri bildirimi korundu (her adımda 10 ms).

## [1.11.0] - 2026-04-25

### Added
- `types/ble.ts`: `SCHED_KIND_FEEDER` / `SCHED_KIND_LIGHTING` sabitleri ve `SchedKind` tipi (firmware `sched_kind_t` ile bire bir).
- `LightingEvent` BLE event tipi (`lighting_on` / `lighting_off`) — kind, start/end saatlerini taşır.

### Changed
- `DeviceSchedule` arayüzü cihaz türünü taşıyacak şekilde genişletildi: `kind`, `end_hour`, `end_minute` zorunlu; `portion` ve `portion_interval` yalnızca feeder kayıtlarında dolu (opsiyonel).
- `Lighting/PlansScreen.tsx`: `set_schedule` / `update_schedule` payload'una `kind: SCHED_KIND_LIGHTING` ve `end_hour` / `end_minute` eklendi (önceden yalnızca başlangıç saati gönderiliyordu).
- `Feeder/PlansScreen.tsx`: `set_schedule` / `update_schedule` payload'una `kind: SCHED_KIND_FEEDER` eklendi.

## [1.10.1] - 2026-04-25

### Changed
- `devices/Ortak/` klasörü `devices/Common/` olarak yeniden adlandırıldı.
- `devices/fish_feeder/` klasörü `devices/Feeder/` olarak yeniden adlandırıldı.

## [1.10.0] - 2026-04-25

### Changed
- Cihaz ekranları klasör yapısı yeniden düzenlendi: `devices/Lighting/` (HomeScreen, PlansScreen) ve `devices/Ortak/` (DevicesScreen, SettingsScreen) klasörleri oluşturuldu.
- `DevicesScreen` ve `SettingsScreen` tüm cihaz tipleri için ortak kullanıma alındı; `fish_feeder` klasöründen kaldırıldı.
- Lighting cihazları için ayrı `HomeScreen` (Hızlı Çalıştır akışı, porsiyon göstergeleri olmadan) ve `PlansScreen` (başlangıç–bitiş saat aralığı, porsiyon formu olmadan) eklendi.
- Tab router importları `fish_feeder/` → `Ortak/` yoluna güncellendi.

## [1.9.0] - 2026-04-25

### Changed
- BLE tarama filtresi çoklu ürün ailesini destekliyor: AquaFeeder-, AquaLighting-, WallLighting-, HomeLighting-.
- Cihaz tipi BLE adından otomatik çıkarılıyor (AquaFeeder → Fish Feeder, AquaLighting → Akvaryum Aydınlatma, vb.).
- Cihazlar ekranı: tarama metni "TolunControl" → "TolunTech"; varsayılan cihaz adı "TolunTech Cihaz".
- Ana ekran: "Hızlı Yemleme" kartı yalnızca AquaFeeder cihazlarında gösteriliyor; diğer cihazlarda "Hızlı Çalıştır" kartı görünüyor.

## [1.8.0] - 2026-04-24

### Changed
- Uygulama markası AquaControl → TolunControl olarak güncellendi (tüm ekran başlıkları, scan metni, gizlilik politikası, kullanım koşulları).
- Cihazlar ekranı: boş durum metni "Lütfen sağ üst köşeden 'Cihaz Ekle' butonuna basın." olarak güncellendi; varsayılan cihaz adı TolunControl Cihaz.
- Cihazlar ekranı: "Besleme Geçmişi" → Fish Feeder için "Besleme Kayıtları", diğer cihazlar için "Cihaz Geçmişi".
- Planlar ekranı: "Günlük Porsiyon" özet kartı kaldırıldı; boş durum ipucu eklendi.
- Planlar ekranı (Fish Feeder dışı cihazlar): Saat yerine çalışma saat aralığı (başlangıç–bitiş), porsiyon ve porsiyon aralığı formu gizlendi; plan kartında saat aralığı gösterimi.
- Ana ekran (Fish Feeder dışı cihazlar): "Son Yemleme/Sonraki Yemleme/Besleme Planları/Besleme Kayıtları" → "Son Çalışma/Sonraki Çalışma/Çalışma Planları/Kayıtlar".
- Ayarlar: "Yemleme Bildirimleri" → "Cihaz Bildirimleri"; gizlilik politikasında "yemleme" ibaresi kaldırıldı; kullanım koşulları TolunTech olarak güncellendi.

## [1.7.2] - 2026-04-24

### Fixed
- BLE status notify artık ATT MTU sınırını (253 byte) aşan JSON'u keserek `SyntaxError: Unexpected end of input` hatasına yol açıyordu. Status notify yalnızca tetikleyici olarak kullanılıp tam veri ATT Read (fragmentation destekli) ile çekiliyor.

## [1.7.1] - 2026-04-24

### Changed
- Plan kaydında etiket girilmezse sıra numarasına göre `Plan 1`, `Plan 2`, … adı otomatik atanıyor; düzenleme sırasında etiket silinirse planın listedeki sırası korunarak etiket yeniden atanıyor (önceki davranış: boş etiket yerine saat dizisi `08:00` kaydediliyordu)

## [1.7.0] - 2026-04-24

### Added
- Cihaza bağlanıldığında Status karakteristiğinden gelen `schedules` alanı parse ediliyor; `DevicePlanSync` bileşeni (`_layout.tsx`) bağlantı kurulduğu anda çalışıyor
- Eşleşen saat dilimindeki planların `label` ve `active` durumu korunuyor; `portion` ve `interval` cihazdan güncelleniyor; cihazda bulunmayan yerel planlar temizleniyor; aynı bağlantı için tek seferlik çalışıyor
- `DeviceSchedule` arayüzü eklendi (`types/ble.ts`); `DeviceInfo`'ya `schedules: DeviceSchedule[] | null` alanı eklendi

## [1.6.0] - 2026-04-24

### Added
- Device Information Service (DIS 0x180A) üzerinden cihaz adı, üretici ve model numarası bağlantı kurulduğunda okunuyor
- Elapsed Time Service (ETS 0x183F) üzerinden çalışma süresi notify ile canlı güncelleniyor (60 saniyede bir)
- `BleConstants.ts`: DIS, ETS ve Device Time Service için tam 128-bit UUID'ler eklendi
- `types/ble.ts`: `DeviceInfo` arayüzüne `manufacturer`, `modelNumber`, `deviceName`, `uptimeSeconds` alanları eklendi
- `BleService`: bağlantıda `readStatus()`, `readDIS()`, `readUptime()` paralel çalışıyor (`Promise.all`); ETS notify aboneliği uptime güncellemelerini `stateListeners`'a iletir; `cleanup()` uptime aboneliğini de kaldırıyor

## [1.5.2] - 2026-04-23

### Fixed
- `PlansScreen` cihaz chip boyutları HomeScreen ve Cihazlar ekranıyla eşleştirildi: `padding: 10 → 12`, `minWidth: 120` eklendi; alt metin fazladan `marginTop` kaldırıldı

## [1.5.1] - 2026-04-23

### Fixed
- `ensureBluetoothOn()` içine runtime izin isteği eklendi: Android 12+ için `BLUETOOTH_SCAN` + `BLUETOOTH_CONNECT`, Android 11 ve altı için `ACCESS_FINE_LOCATION`; izin reddedilirse kullanıcıya Alert gösteriliyor
- PlansScreen cihaz chip'inde bağlantı durumu noktası (dot) cihaz adıyla aynı satırda gösteriliyor

## [1.5.0] - 2026-04-23

### Added
- `@react-native-async-storage/async-storage` entegrasyonu: cihazlar, besleme planları, besleme geçmişi, tema modu ve vurgu rengi kalıcı hale getirildi (`@aquacontrol:` önekli anahtarlar)
- `AppStateContext` artık `isLoading: boolean` expose ediyor
- Uygulama açılışında `btConnected` tüm cihazlarda `false` sıfırlanıyor; veriler yüklenene kadar arka plan rengi gösteriliyor

### Fixed
- `SettingsScreen` versiyon numarası artık `expo-constants` üzerinden gerçek `app.json` versiyonunu gösteriyor (sabit `1.0.0` yerine)

## [1.4.2] - 2026-04-23

### Added
- Welcome ekranı: uygulama açılışında 1.5 saniyelik splash, `app/index.tsx` ile `/(tabs)` yönlendirmesi (geri tuşu ile dönülemez)
- Bluetooth kapalı bildirimi: `+` veya `Bağlan` butonuna basıldığında BT kapalıysa tıklanabilir mavi banner; Android'de BT ayarları, iOS'ta uygulama ayarları açılır; 4 saniye sonra otomatik kapanır
- `BleService`'e `getBluetoothState()` ve `enableBluetooth()` metotları eklendi
- `BleContext`'e `bluetoothOffNotice` state ve `notifyBluetoothOff()` fonksiyonu eklendi

### Changed
- DevicesScreen kart sırası: Bağlantı Durumu → Bildirimler → Besleme Geçmişi → Hakkında
- `KeyboardAvoidingView` yerine `Keyboard` event listener ile dinamik `paddingBottom`; klavye açıkken Kaydet butonu görünür kalır
- `ConnectErrorBanner` `_layout.tsx`'e taşındı; DevicesScreen'deki satır içi `lastError` banner'ı kaldırıldı

### Fixed
- Cihazı Unut: `macId`, `id` ve lowercase üçlü eşleşmeyle BT bağlantısı kesilmiyor hatası giderildi
- `connectedDeviceId` null olduğunda tüm cihazlarda `btConnected` bayrağı sıfırlanıyor
- `ConnectErrorBanner` yeni tarama/bağlanma başlayınca hemen kapanıyor

## [1.4.1] - 2026-04-22

### Changed
- DevicesScreen: bağlanma işlemi sürerken (`connectionState === 'connecting'`) header'daki `+` butonu `disabled` + `opacity: 0.35` ile devre dışı bırakılıyor

## [1.4.0] - 2026-04-22

### Added
- Bluetooth Bağlantısını Kes / Cihazı Unut / Yeniden Başlat butonlarına `Alert.alert` onay dialogları eklendi
- Bağlantı cooldown: bağlantı kurulunca Kes butonu, kesildikten sonra Bağlan butonu 3 saniye devre dışı
- `ConnectErrorBanner`: `connecting → error` geçişinde `✕ Cihaza bağlanamıyor` kırmızı banner 3 saniye gösteriliyor
- PlansScreen: kayıtlı cihaz yokken boş durum mesajı; zaman çakışması koruması (`Alert.alert`)

### Fixed
- `HomeScreen`: `setInterval`/`setTimeout` `useEffect` + `useRef` yapısına taşındı; bellek sızıntısı giderildi
- `BleContext`: `startScan()` içindeki `setTimeout` `useRef` ile saklanıyor; `SCAN_TIMEOUT_MS` sabiti kullanılıyor
- `BleService`: `sendCommand()` başında `connectedDevice` yerel değişkene alındı; JSON parse hataları `console.warn` ile loglanıyor
- `PlansScreen — openEdit`: `interval: p.interval` düzeltmesi; `save()` modal tipi güvenli daraltması uygulandı
- `DevicesScreen — handleForgetDevice`: cihaz silinince ilgili tüm planlar temizleniyor; stale closure düzeltildi
- `HomeScreen`: `plansWithMin.filter().sort()` `useMemo` ile önbelleğe alındı

## [1.3.1] - 2026-04-22

### Added
- Global bağlantı bildirimleri: bağlanınca `✓ Cihaz bağlandı` yeşil banner, beklenmedik kopuşta `⚠ Cihaz bağlantısı kesildi` kırmızı banner (tab bar üstünde, 3 saniye)
- `BleService.disconnect()`: `intentionalDisconnect` bayrağı; `onDisconnected` yalnızca beklenmedik kopmalar için `disconnectListeners` tetikliyor

### Changed
- PlansScreen: Düzenle/Sil butonları cihaz bağlı değilken `opacity: 0.35` ve `disabled`; varsayılan plan açılış saati `12:30`; TimePicker açılışında seçili saate otomatik scroll; plan kartı düzeni (etiket üstte büyük/bold, detaylar altta)
- `FeedPlan` tipine `interval` alanı eklendi
- Başlangıç vurgu rengi turkuaz (hue 185) olarak değiştirildi

## [1.3.0] - 2026-04-22

### Added
- Yemleme cooldown: yemleme sonrası 5 saniyelik geri sayım; basılı tutulursa `İptal · 3s` geri sayımı; `✓ Yemlendi!` onay mesajı
- DevicesScreen boş durum ekranı: akvaryum görseli + yönlendirici metin
- Cihaz bağlantı durumu noktası: BT/WiFi bağlıysa yeşil, ikisi de yoksa kırmızı

### Fixed
- `isConnected` yerine `connectedDeviceId === device.macId` kontrolü (tarama sırasında yanlış `Bağlan` görünmesi düzeltildi)
- `Son Yemleme` kartı gerçek besleme logunu gösteriyor (plan zamanı yerine)
- Cihaz silinince `selectedDeviceId` senkronize ediliyor

## [1.2.0] - 2026-04-19

### Changed
- DevicesScreen: cihaz ekleme butonu `+` olarak header'a taşındı; BT bağlan/kes butonları cihaz kartı içine eklendi; besleme geçmişi toggle ile görüntüleniyor; cihaz bilgileri kartı eklendi
- HomeScreen: sayfa başlığı kaldırıldı; bağlantı durumu pill'i header'a taşındı
- Navigation bar: tab etiketleri güncellendi; aktif tab genişleyen arka plan + renk değişimi vurgusu

## [1.1.0] - 2026-04-18

### Changed
- BLE cihaz adı filtresi: yalnızca `AquaControl-` önekiyle başlayan cihazlar listelenir; servis UUID filtresi kaldırıldı
- Android izinleri eklendi: `BLUETOOTH_SCAN`, `ACCESS_FINE_LOCATION`
- Cihaz ekleme akışı: BLE tarama → seçim → otomatik kayıt; bağlantı kesilince cihaz listeden kaldırılmıyor
- `DeviceInfo` arayüzüne `fwVersion`, `hwVersion` alanları eklendi

## [1.0.0] - 2026-04-18

### Added
- İlk sürüm: React Native + Expo (EAS Build), Android platform, `react-native-ble-plx`, Expo Router (tab tabanlı)
- `BleService.ts`, `BleContext.tsx`, `BleConstants.ts` — BLE katmanı
- `AppStateContext.tsx` — cihaz, plan ve besleme logu global state; `mockData.ts` örnek veriler
- HomeScreen, DevicesScreen, PlansScreen, SettingsScreen
- `Icon`, `Toggle`, `BottomSheet` bileşenleri
- `ThemeContext.tsx` — açık/koyu tema ve accent renk desteği
