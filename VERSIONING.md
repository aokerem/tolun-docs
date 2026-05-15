# VERSIONING REHBERİ — Tolun

## Mevcut Sürümler

| Bileşen | Versiyon | Kaynak Dosya |
|---|---|---|
| **Firmware** | `1.19.2` | `Firmware/main/config/system_config.h` → `FIRMWARE_VERSION` |
| **Mobile App (TolunControl)** | `1.34.3` | `Mobile App/package.json` → `version` |
| **Hardware** | `1.0.0` | Sabit — donanım revizyonu değişmedikçe güncellenmez |

---

## Repo Yapısı

Firmware ve Mobile App **ayrı git repolarında** tutulur. Her bileşen kendi versiyon geçmişini ve tag'ini bağımsız yönetir.

```
tolun/
  Firmware/     ← kendi git reposu, kendi tag'leri (v1.17.0, v1.18.0 ...)
  Mobile App/   ← kendi git reposu, kendi tag'leri (v1.30.0, v1.31.0 ...)
  docs/         ← paylaşılan dokümanlar (versiyonlanmaz)
```

---

## Versiyon Şeması (Semantic Versioning)

`MAJOR.MINOR.PATCH` formatı kullanılır.

| Değişiklik türü | Etki | Örnek |
|---|---|---|
| Geriye dönük uyumsuz değişiklik (breaking) | MAJOR artar | `1.x.x → 2.0.0` |
| Yeni özellik (geriye uyumlu) | MINOR artar | `1.16.x → 1.17.0` |
| Hata düzeltme, küçük iyileştirme | PATCH artar | `1.17.0 → 1.17.1` |

---

## Commit Mesajı Formatı

Her commit [Conventional Commits](https://www.conventionalcommits.org/) formatında yazılır.

```
feat: yeni özellik açıklaması
fix: düzeltilen hata açıklaması
feat!: geriye uyumsuz değişiklik
chore: bakım, bağımlılık güncellemesi
docs: sadece dokümantasyon değişikliği
refactor: davranış değiştirmeyen kod düzenlemesi
test: test ekleme/düzeltme
ci: CI/CD değişikliği
```

`feat!` veya `fix!` → MAJOR bump gerektirir.  
`feat` → MINOR bump gerektirir.  
`fix`, `chore`, `refactor` vb. → PATCH bump gerektirir.

## Otomatik Versiyon Bump (`prepare-commit-msg` hook)

Her iki repo'da `scripts/git-hooks/prepare-commit-msg` git hook'u tanımlı. Commit mesajı `vX.Y.Z` içeriyorsa hook ilgili versiyon dosyasını **otomatik günceller ve commit'e ekler**:

| Repo | Güncellediği dosya | Alan |
|---|---|---|
| Mobile App | `package.json` | `version` |
| Firmware | `main/config/system_config.h` | `#define FIRMWARE_VERSION` |

Versiyon içermeyen commit'lerde (ara çalışma, `chore:` vb.) hook **no-op** kalır.

### Aktivasyon (her clone sonrası bir kez)

```bash
# Mobile App
cd "Mobile App"
git config core.hooksPath scripts/git-hooks

# Firmware
cd Firmware
git config core.hooksPath scripts/git-hooks
```

`core.hooksPath` ayarlanmazsa hook çalışmaz; commit mesajındaki versiyon ile dosyadaki versiyon arasında drift oluşur (geçmişte v1.30.1 → v1.34.3 arası 20 commit boyunca `package.json` 1.30.0'da kalmıştı).

---

## Release Akışı

### Commit Aşaması (günlük iş)

Özellik geliştirirken normal conventional commit'ler atılır. Her commit release değildir; commitler birikir.

```
feat: yeni komut eklendi
fix: BLE timeout düzeltildi
fix: plan senkronizasyon hatası giderildi
```

### Release Aşaması

Release hazır olduğunda şu adımlar sırayla yapılır:

#### 1. Changelog güncelle

`docs/firmware_changelog.md` veya `docs/mobileapp_changelog.md` dosyasına yeni versiyon bölümü ekle:

```markdown
## [1.18.0] - 2026-05-15

### Added
- ...

### Fixed
- ...
```

#### 2. Release commit'i at

Versiyon numarasını **manuel güncelleme yok** — `prepare-commit-msg` hook commit mesajındaki `vX.Y.Z`'yi parse edip ilgili dosyayı (`package.json` veya `system_config.h`) günceller ve commit'e ekler.

```bash
git add docs/firmware_changelog.md  # veya mobileapp_changelog.md
git commit -m "feat: v1.18.0 — kısa açıklama"
# → hook FIRMWARE_VERSION/version'u 1.18.0 yapar, commit'e dahil eder
```

#### 3. Tag at

```bash
git tag v1.18.0
```

#### 4. Push et (commit + tag)

```bash
git push origin main
git push origin v1.18.0
```

#### 5. GitHub Release oluştur

GitHub'da ilgili tag üzerinden Release oluştur. Release notlarına changelog'dan ilgili versiyon bölümünü yapıştır.

---

## Uyumluluk Matrisi

BLE protokolü değiştiğinde veya yeni komut eklendiğinde bu tablo güncellenir.

| TolunControl (Mobile) | Firmware | Hardware | BLE Protokol Notu |
|---|---|---|---|
| `1.34.3` | `1.19.2` | `1.0.0` | `first_conn_ts` / `total_uptime_sec` status alanları; plan/log response'undan `index` alanı kaldırıldı |
| `1.34.0` | `1.19.0` | `1.0.0` | Cihaz yaşam istatistikleri (`first_conn_ts`, `total_uptime_sec`) status payload'una eklendi |
| `1.30.0` | `1.17.0` | `1.0.0` | `get_schedule` indeks bazlı, `get_feed_log_entry` / `get_lighting_log_entry` destekleniyor |
| `1.29.0` | `1.16.0` | `1.0.0` | Schedule chunking, status payload daraltıldı |

> Firmware ve Mobile App versiyonları birebir eşleşmek zorunda değildir. Ancak BLE protokol değişikliklerinde (yeni komut ekleme, payload formatı değişimi) her iki tarafın minimum uyumlu versiyonu bu tabloya işlenmeli.

---

## Stabil / Pre-release

| Versiyon aralığı | Anlam |
|---|---|
| `0.x.y` | Pre-release — API değişebilir |
| `1.0.0+` | Stabil — üretim ortamına uygun |

Her iki bileşen de `1.x.x` bandında olduğundan stabil kabul edilir.
