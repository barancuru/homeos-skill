# HomeOS API ve MCP başvurusu

Sürüm: HomeOS 0.10.119, REST API 0.5.11, MCP 0.7.11 (2026-11-16). Canlı REST şeması: `GET https://home.curu.tr/api/openapi.json` (anahtar istemez). MCP'de kesin liste `tools/list`'tir; bu dosyayla çelişirse canlı yanıt doğrudur.

## Kimlik doğrulama
İki yol vardır, ikisi de aynı delegasyon, hane ve kota kurallarına bağlıdır.
- **OAuth (MCP için önerilen):** `https://home.curu.tr/mcp` adresine bağlanan istemci OAuth 2.1 akışını kendisi yürütür (dinamik istemci kaydı, PKCE S256). Kapsamlar: `mcp:read`, `mcp:write`, `mcp:delete` (yalnız `mcp:write` ile), `mcp:health`. Erişim belirteci (`hoa_`) 1 saat, yenileme belirteci (`hor_`) 30 gün geçerlidir ve her kullanımda değişir. Bağlantı kullanıcının "API erişimim" sayfasında bir anahtar gibi listelenir ve oradan iptal edilir.
- **API anahtarı:** `Authorization: Bearer hos_<43 karakter>` (yedek başlık: `X-HomeOS-Api-Key`). Anahtar URL'de, çerezde ya da gövdede kabul edilmez. Ömrü paket ayarına bağlıdır (en çok 90 gün). Oluştururken seçilenler: **yazma izni** (ekleme araçları), **arşivleme ve silme izni** (yalnız yazma izniyle), **taslak zorunlu** (kişisel kayıtlar onay bekler), **sağlık okuma** (ayrıca rıza gerekir).
- Geçerlilik için aktif API delegasyonu, aktif hesap ve aktif hane üyeliği gerekir; biri biterse `401` döner.
- **Delegasyon:** e-postası doğrulanan kullanıcıya ilk hanesinde ücretsiz paket kendiliğinden açılır. Hane sahibi, Entegrasyonlar → API erişimim'den kendine ya da e-postası doğrulanmış bir üyeye ücretsiz delegasyon açabilir (24 saatte en çok 3). Yöneticinin kapattığı delegasyon buradan yeniden açılamaz.
- `hhc_` Health Connect cihaz anahtarları yalnız `POST /api/v1/health-connect/daily` içindir; bu skill'in işi değildir.

## Kapsam
- **Hane verisi:** varlık, bakım, kategori, konum, belge, ortak gider, tekrarlayan ödeme, üyeler. Hanenin tüm üyeleri görür.
- **Kişisel veri:** Bütçem kayıtları, kategoriler, sınırlar, hesap bakiyeleri, abonelikler, sağlık özeti. Yalnız bağlantıyı açan kullanıcının kendi kayıtları döner, diğer üyelerinkiler dönmez.
- Şifreli alanlar (ör. seri numarası) API'de dönmez. Belgelerde yalnız üst bilgi döner, dosya içeriği dönmez.

## REST (`https://home.curu.tr/api/v1`, yalnız GET)
| Yol | Döner | Sorgu (`limit`, `offset` hepsinde) |
|---|---|---|
| `/assets` | Varlıklar, `asset_no` sırasıyla (arşivliler yalnız `include_archived=true` ile, `archived_at` alanıyla) | `warranty_before`, `status`, `q`, `updated_since`, `include_archived` |
| `/assets/{asset_no}` | Tek varlık (gövde doğrudan nesne; son bakımlar ve para birimine göre maliyet toplamı dahil) | — |
| `/maintenance/plans` | Bakım planları, `next_due` sırasıyla | `due_before`, `asset_no`, `is_active` |
| `/maintenance/logs` | Tamamlanan bakımlar, yeniden eskiye | `asset_no`, `since`, `until` |
| `/categories`, `/locations` | Hiyerarşi düğümleri (`id`, `parent_id`, `name`; konumda `facade`, `floor`) | — |
| `/finance/entries` | Varlık maliyet kayıtları | `asset_no`, `since`, `until` |
| `/documents` | Belge üst bilgisi | `expires_before`, `asset_no` |
| `/recurring/logs` | Tekrarlayan ödeme kayıtları, yeniden eskiye | `payment_id`, `since`, `until` |
| `/health/daily` | Kendi günlük sağlık özetlerin (sağlık izni + rıza gerekir) | yalnız `from`, `to` (en çok 92 gün) |

- `limit` 1–200 (varsayılan 100). Liste gövdesi `{"data":[...],"count","total","has_more","limit","offset"}`: `total` süzgeçle eşleşen tüm kayıtlar; `has_more` true ise `offset += limit`.
- Yanıt başlıkları: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-HomeOS-Token-Expires` (anahtarın bitişi).

## MCP (`POST https://home.curu.tr/mcp`)
Durumsuz Streamable HTTP, yalnız JSON yanıt (SSE yok, `GET` 405). Protokol `2025-11-25` (`2025-03-26` de kabul edilir). Listeler `{"data":[...],"count","limit","offset","has_more"}` döner; `limit` 1–100 (varsayılan 25), `offset` 0–10000. Argümanlar şemadaki türde olmalı (`25`, `"25"` değil); bilinmeyen argüman `-32602` ile reddedilir. `*` zorunlu argümandır.

**Okuma (her bağlantı):**
| Araç | Argüman |
|---|---|
| `homeos_list_assets` | `limit`, `offset`, `warranty_before`, `status` (`active`/`stored`/`retired`), `q`, `updated_since`, `include_archived` |
| `homeos_get_asset` | `asset_no`* (arşivli varlığı da `archived: true` ile döndürür; bağlı `contacts` ve `insurance` dahil) |
| `homeos_list_maintenance_plans` | `limit`, `offset`, `due_before`, `asset_no`, `is_active` |
| `homeos_list_maintenance_logs` | `limit`, `offset`, `asset_no`, `since`, `until` |
| `homeos_list_categories`, `homeos_list_locations` | `limit`, `offset` |
| `homeos_list_finance_entries` | `limit`, `offset`, `asset_no`, `since`, `until` |
| `homeos_list_members` | — (user_id, ad, rol; e-posta dönmez) |
| `homeos_list_contacts` | `asset_no`, `q`, `limit`, `offset`: dış paydaşlar (`id`, iletişim, bağlı varlıklar ve rolleri) |
| `homeos_list_insurance` | `asset_no`, `expires_before`, `limit`, `offset`: poliçeler (`id`, tür, sağlayıcı, tarih, prim, teminat) |
| `homeos_usage` | —: dakika sınırı ve kalan, aylık kota ve kalan, bağlantı bitişi, protokol sınırı (kotaya sayılmaz) |
| `homeos_list_documents` | `limit`, `offset`, `expires_before`, `asset_no` (`source`: asset, household, personal_entry); her satırda `key` (`asset:2` gibi; kimlik kaynaklar arasında çakışır, tekil tanım `key`'dir) |
| `homeos_due` | `from`*, `to`* (en çok 400 gün), `include_overdue` (varsayılan true): bakım, tekrarlayan ödeme, garanti, hane belgesi (`documents`) ve poliçe (`insurance`) bitişleri; `overdue` kullanıcının saat dilimindeki **bugüne** göredir (sorgunun `from`'una göre değil); `from`'dan önce vadesi geçmiş etkin kalemleri `include_overdue` dahil eder; garanti, belge ve poliçe satırları da aynı kuralla `overdue` alır; arşivdeki varlığın garantisi girmez, `counts.overdue` sayar; garanti satırları da `kind: warranty_expiry` ve `overdue` taşır |
| `homeos_list_shared_expenses` | `since`, `until`, `currency`, `limit`, `offset` (`balances`: üye başına net) |
| `homeos_list_recurring` | `limit`, `offset`, `include_inactive` ya da eşdeğeri `include_archived` (varsayılan yalnız etkinler; pasifler `active: false`); her ödemede `last_paid_on`, `log_count` |
| `homeos_list_payment_logs` | `payment_id`, `since`, `until`, `limit`, `offset`: ödeme kayıtları (`id`, `payment_id`, `title`, `paid_on`, `amount`, `currency`, `paid_by`, `shared_expense_id`, `added_by_agent`); silmek için `id` → `homeos_delete payment_log` |
| `homeos_list_personal_entries` | `since`, `until`, `category`, `currency`, `kind`, `limit`, `offset` (her kayıtta `account` ve `attachment_count`) |
| `homeos_list_personal_categories` | `include_archived` |
| `homeos_list_budgets`, `homeos_list_balances`, `homeos_list_subscriptions` | `currency` |
| `homeos_finance_summary` | `period` (`week`/`month`), `currency` |
| `homeos_list_health_daily` | yalnız sağlık izni ve geçerli rıza varsa görünür |

**Ekleme (yalnız yazma izinli bağlantı; hepsi `request_id`* ister, düzenleme aracı yoktur):**
| Araç | Argüman |
|---|---|
| `homeos_create_asset` | `name`*, `category`, `brand`, `model`, `serial_number`, `location`, `status`, `purchased_on`, `purchase_price`, `currency`, `useful_life_years`, `warranty_until`, `notes`, `usage_warnings` |
| `homeos_create_maintenance_plan` | `asset_no`*, `title`*, `next_due`*, `repeat_rule` (`days` varsayılan / `monthly_day` her ayın `repeat_day`. günü / `monthly_window` her ayın `repeat_day`–`repeat_window_end` arası), `repeat_days` (yalnız `days`), `repeat_day`, `repeat_window_end`, `description`, `is_active` |
| `homeos_log_maintenance` | `plan_id`*, `performed_on`*, `cost`, `currency` (`TRY`/`USD`/`EUR`; boşsa varlığın para birimi), `note` |
| `homeos_add_asset_expense` | `asset_no`*, `occurred_on`*, `kind`* (`repair`/`operation`/`improvement`/`other`), `amount`*, `currency`*, `description`* |
| `homeos_add_expense`, `homeos_add_income` | `amount`*, `currency`, `category`, `account` (hesap adı ya da id, `homeos_list_balances`'tan; bakiye arayüzdeki gibi güncellenir, taslakta onayda), `occurred_on`, `note`, `recurrence` (`one_time`/`recurring`), `draft`, `items` (ad, adet, birim fiyat, tutar; tutar boşsa adet × birim fiyat), `strict_items` (true: kalem toplamı tutmazsa kayıt açılmaz) |
| `homeos_attach_file` | `entry_id`*, `filename`*, `content_base64`* (JPEG/PNG/WebP/PDF, en çok 8 MB, kayıt başına 5 dosya; yalnız kendi Bütçem kaydına) |
| `homeos_set_budget` | `period`* (`week`/`month`), `limit_amount`*, `currency`, `category` (boş = toplam sınır) |
| `homeos_set_balance` | `account`*, `balance`*, `currency`, `floor_amount` |
| `homeos_add_subscription` | `name`*, `amount`*, `renews_on_day`*, `currency`, `period` (`month`/`year`), `renews_month`, `account`, `reminder_days` |
| `homeos_add_shared_expense` | `amount`*, `currency`, `occurred_on`, `category`, `note`, `split_method` (`equal`/`shares`/`amounts`), `participants` (virgüllü user_id; boş = tüm üyeler), `values`, `items`. Ödeyen bağlantının sahibidir. |
| `homeos_mark_paid` | `payment_id`*, `paid_on`, `amount` (ortak ödemede eşit bölüşümlü ortak gider açılır) |
| `homeos_add_recurring_payment` | `title`*, `amount`*, `next_due`*, `currency`, `category`, `repeat_rule` (`days`/`monthly_day`/`monthly_window`), `repeat_days`, `repeat_day`, `repeat_window_end`, `responsible_user_id` (`homeos_list_members`), `shared` |
| `homeos_add_contact` | `name`*, `organization`, `phone`, `email`, `website` (https), `notes`, `asset_no` + `role` (`seller`/`maintainer`; ikisi birlikte) |
| `homeos_add_insurance` | `asset_no`*, `insurance_type`* (`dask/traffic/casco/home/mandatory/health/other`), `starts_on`*, `ends_on`*, `premium`*, `coverage`*, `currency`*, `provider`, `policy_number` |
| `homeos_add_household_document` | `filename`*, `content_base64`* (en çok 8 MB; taranır ve şifrelenir), `title`, `kind` (`lease`/`deposit`/`policy`/`other`), `expires_on`, `remind_days`, `deposit_amount`, `deposit_currency` |
| `homeos_add_asset_document` | `asset_no`* (arşivlenmemiş), `filename`*, `content_base64`* (en çok 8 MB; PDF, JPEG, PNG, WebP, TXT, DOCX, XLSX; taranır ve şifrelenir). Sonuç `homeos_list_documents`'ta `source: asset` olarak görünür |

- `request_id`: 8–64 karakter, `A-Z a-z 0-9 . _ : -`. Aynı kimlikle ikinci çağrı yeni kayıt açmaz, ilk sonucu döner. Ağ hatasından sonra aynı `request_id` ile dene.
- Ajanın eklediği kayıtlar arayüzde "Ajan ekledi" işaretiyle görünür.
- `draft: true` olan kişisel kayıt toplamlara, sınırlara ve bakiyeye girmez; kullanıcı Bütçem'deki "Onay bekleyen fişler" kutusunda onaylar ya da reddeder.

**Arşivleme ve silme (yazma + `mcp:delete`; hepsi `request_id`*, `type`*, `id`* ister; `destructiveHint`):**
| Araç | `type` (★ = yalnız hane sahibi) |
|---|---|
| `homeos_archive` | `asset`★ (`id` = `asset_no`), `maintenance_plan`★, `tax_item`★, `damage_report`, `chore`, `recurring_payment`, `personal_account`, `personal_category`, `personal_budget_limit`, `personal_subscription` |
| `homeos_restore` | `asset`★ (`with_plans: true` ile arşivli planları da geri alır), `maintenance_plan`★ (önce varlığı etkin olmalı), `personal_category`, `recurring_payment` (pasif ödemeyi etkinleştirir; vade geçmişse `next_due` zorunlu, bugün ya da sonrası) |
| `homeos_delete` | `document`★ (+ zorunlu `source`: `asset`/`household`, listedeki satırdan), `document` + `source: personal_entry` (fiş dosyası; yalnız kaydın sahibi), `maintenance_log` (hane sahibi ya da kaydı giren; en son kayıtsa `restore_due` varsayılan true ile `next_due` önceki değerine döner, 0.10.115 öncesi kayıtlarda dönmez), `payment_log` (hane sahibi ya da ödeyen; `with_shared_expense: true` bağlı ortak gideri de siler; en son kayıtsa `restore_due` ile `next_due` geri döner), `household_document`★ (eski; `document` + `source: household` ile aynı), `asset_expense`★, `contact`★, `insurance`, `personal_entry`, `shared_expense`; **yalnız arşivdeyse:** `asset`★ (`id` = `asset_no`), `maintenance_plan`★, `subscription`★ (ev aboneliği), `tax_item`★, `damage_report`, `chore`, `recurring_payment`, `personal_account`, `personal_category`, `personal_budget_limit`, `personal_subscription` |

- `asset` ve `maintenance_plan` arşivinde `confirm_name` zorunludur: varlık adı ya da plan başlığı, tam olarak. Geri almada istenmez.
- Silme kalıcıdır. Arşivlenebilen türler yalnız **arşivdeyse** silinir; etkinse `-32602` (önce `homeos_archive`). `asset` ve `maintenance_plan` silmede de `confirm_name` zorunlu. Varlık silinince belgeleri, fotoğrafları, planları, bakım ve maliyet kayıtları, poliçeleri de gider (dosyalar diskten); hasar kayıtları ve yatırım planları kalır, bağı kopar. Başka haneyle paylaşılan varlık ve abonelik silinemez. Yanıtta `deleted` silinen ya da bağı kalkan bağlı kayıt sayılarıdır; bağlı kayıt yoksa `{}`. Hane ve profil silinemez.
- Kişisel türler yalnız bağlantı sahibinin kayıtlarıdır. Kişisel kayıt silinince hesap bakiyesi geri işlenir; ortak gider silinince paylar kişisel bütçelerden kalkar; belge silinince dosya da silinir.
- Her çağrı denetim kaydına tür ve kimlikle yazılır. Başarılı sonuç: `{"done":"archive|restore|delete","type","id","request_id"}`.

## Alanlar
- **Asset:** `asset_no` (hane içi numara, kimlik budur), `name`, `category`, `location`, `brand`, `model`, `status`, `purchased_on`, `purchase_price` (metin), `currency`, `useful_life_years`, `warranty_until`.
- **MaintenancePlan:** `id`, `asset_no`, `title`, `description`, `next_due`, `repeat_rule` (`days`/`monthly_day`/`monthly_window`), `repeat_days` (null = tek seferlik; aylık kurallarda yaklaşık değer, hesapta kullanma), `repeat_day`, `repeat_window_end`, `repeat_label` (okunur, ör. "Her ayın 5'i"), `is_active`, `assigned_to` (user_id). Sonraki tarih için `next_due` ya da `homeos_due`.
- **MaintenanceLog:** `id`, `asset_no`, `title`, `performed_on`, `cost` (metin), `currency` (0.10.101 öncesi kayıtlar `TRY`), `note`, `plan_archived` (arşivlenmiş planın kaydı da listelenir ve maliyet toplamına girer).
- **FinanceEntry:** `id` (`homeos_delete asset_expense` için), `asset_no`, `occurred_on`, `kind`, `amount` (metin), `currency`, `description`.
- Para birimine göre toplam gibi haritalar boşken de `{}` döner.
- Dosya yüklemede (varlık, hane belgesi, fiş) uzantı içerikle uyuşmalı (ör. `.pdf` gerçekten PDF); uyuşmazsa `filename:` hatası. Arşivdeki varlığa yazma "varlık arşivde" hatası verir: önce `homeos_restore`.
- Tutarlar ondalık **metin** gelir (`"1250.00"`). Tarihler `YYYY-AA-GG`.

## Limitler
- **Dakika sınırı:** bağlantı başına, paket ayarına bağlı (varsayılan paket 1000/dk). Aşılınca `429 rate_limited`, `Retry-After: 60`.
- **Aylık kota:** delegasyon başına (varsayılan paket 20.000/ay); yalnız başarılı veri istekleri sayılır. Aşılınca `429 monthly_quota_exceeded`, `Retry-After: 3600`. Limit artırımı API erişimim sayfasından istenir.
- **MCP protokol istekleri** (`initialize`, `notifications/*`, `ping`, `tools/list`, `resources/list`, `prompts/list`) dakika sınırına ve kotaya sayılmaz; kendi sınırları bağlantı başına 60/dk'dır.
- **Gövde boyutu:** MCP istek gövdesi en çok 12 MB (8 MB dosyanın base64 hâli sığar).
- **Denetim kaydı:** her istek kaydedilir (anahtar, yol, durum, IP).

## Hatalar
| Kod | Gövde | Ne yapılır |
|---|---|---|
| 400 | `invalid_limit` ve benzeri | Sorgu değerini düzelt |
| 401 | `invalid_token` / `unauthorized` | Başlık yok ya da biçim yanlış |
| 401 | `invalid_or_expired_token` | Süre doldu, iptal edildi ya da delegasyon/üyelik bitti: yeniden bağlan ya da yeni anahtar |
| 404 | `not_found` / `unknown_endpoint` | Kayıt ya da yol yok |
| 405 | `method_not_allowed` | REST yalnız GET, MCP yalnız POST |
| 413 | `payload_too_large` | MCP gövdesi 12 MB'ı aştı |
| 429 | `rate_limited` / `monthly_quota_exceeded` | Gövdedeki `retry_after` (saniye) kadar bekle; döngüyle deneme |
| 500 | `internal_error` | Bir kez sonra yeniden dene; sürerse kullanıcıya bildir |
| MCP 403 | `invalid_origin` / `invalid_host` | Adres `https://home.curu.tr/mcp` olmalı (port 443) |
| MCP `-32001` | `write_not_permitted` / `delete_not_permitted` | Bağlantının yazma ya da arşivleme/silme izni yok |
| MCP `-32001` | `owner_required` | İşlemi yalnız hane sahibi yapabilir |
| MCP `-32602` | `message` | Araç adı ya da argüman geçersiz. Yazma araçlarında mesaj alanı söyler (ör. `amount: en çok 2 ondalık`); düzeltip yeni `request_id` ile dene |
