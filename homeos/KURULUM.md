# HomeOS skill'i: kurulum

**Ön koşul:** home.curu.tr'de doğrulanmış e-postalı bir hesap ve bir hanede aktif üyelik. Entegrasyonlar → **API erişimim** sayfası "Yetki yok" diyorsa: hane sahibiysen aynı sayfadan kendine ücretsiz API yetkisi aç; değilsen hane sahibinden iste.

## 1. Skill'i kur
- **Linux / macOS:** `unzip homeos-skill.zip -d ~/.claude/skills/ && chmod +x ~/.claude/skills/homeos/scripts/homeos-api`
- **Windows:** ZIP'i `%USERPROFILE%\.claude\skills\` içine aç.
- **claude.ai:** Ayarlar'daki Skills (Yetenekler) bölümünden ZIP'i yükle. Hesabında bu bölüm yoksa skill'siz de olur: 2. adımdaki bağlayıcı tek başına yeterlidir.

## 2. HomeOS'u bağla (önerilen yol, anahtar gerekmez)
- **claude.ai / Claude Desktop:** Ayarlar → Bağlayıcılar → **Özel bağlayıcı ekle** → URL: `https://home.curu.tr/mcp` → Bağlan.
- **Claude Code:** `claude mcp add --transport http homeos https://home.curu.tr/mcp`. Ardından Claude Code içinde `/mcp` → homeos → kimlik doğrula.

Tarayıcıda HomeOS açılır. Giriş yap, hane seç ve izinleri işaretle:
- **Okuma**: soru sormak için yeterli.
- **Yazma**: Claude'un kayıt eklemesini istiyorsan işaretle.
- **Güncelleme**: Claude'un var olan kaydı (eşya, bakım planı, ödeme, ev aboneliği, konum, kategori) düzeltmesini istiyorsan, yazmayla birlikte işaretle. Claude her değişiklikten önce eski ve yeni değeri gösterip sorar.
- **Arşivleme ve silme**: Claude'un kayıt arşivlemesini ya da silmesini istiyorsan, yazmayla birlikte işaretle. Claude her işlemden önce sorar; silme geri alınamaz.
- **Sağlık**: yalnız sağlık özetini paylaşmak istiyorsan işaretle. Ayrıca rıza ister.

Sonra sor: *"HomeOS'ta önümüzdeki 3 ayda garantisi bitecek ya da bakımı gelecek neler var?"*

**İki not:**
- **Hane seçimi:** onay ekranında hangi hane seçilirse bağlantı yalnız o haneyi görür. Birden fazla hanen varsa doğru haneyi seç. Değiştirmek için bağlantıyı kes (Claude Code'da `/mcp` → homeos), yeniden giriş yap ve diğer haneyi seç.
- **Çift bağlantı:** claude.ai'de bağlayıcı zaten ekliyse Claude Code onu aynı hesapla kendiliğinden görür (`claude.ai HomeOS`). Ayrıca `claude mcp add` gerekmez. Eklersen aynı araçları iki kez görürsün; iki bağlantı farklı hanelere bağlı da olabilir.

Bağlantıyı kesmek için: Entegrasyonlar → API erişimim → bağlantıyı iptal et.

## 3. Yedek yol: API anahtarı (bağlayıcı kullanamıyorsan)
1. Entegrasyonlar → API erişimim → **API anahtarı oluştur**. Kayıt eklenmesini istiyorsan "yazma izni"ni, güncelleme de istiyorsan "güncelleme izni"ni, arşivleme ve silme de istiyorsan ayrıca "arşivleme ve silme izni"ni işaretle. Anahtar (`hos_…`) yalnız bir kez gösterilir; parola kasana kaydet.
2. Anahtarı ortam değişkeni olarak tanımla. **Sohbete yapıştırma.**
   - **Linux / macOS:** `echo "export HOMEOS_API_TOKEN='hos_...'" >> ~/.bashrc` (zsh kullanıyorsan `~/.zshrc`), sonra `source ~/.bashrc`.
   - **Windows (PowerShell, bir kez):** `[Environment]::SetEnvironmentVariable('HOMEOS_API_TOKEN','hos_...','User')`. Değişkeni açık pencereler görmez, yeni pencere aç.
3. Dene: `~/.claude/skills/homeos/scripts/homeos-api check`. Çıktıda "anahtar geçerli" yazmalı. Windows'ta bu komutu **Git Bash**'te çalıştır (Git for Windows ile gelir).
4. Claude'u tamamen kapatıp yeniden aç: ortam değişkeni yalnız yeni başlatılan uygulamaya geçer. Windows'ta sistem tepsisinden de kapat.

Anahtar en fazla 90 gün geçerlidir ve yalnız kendi haneni görür. Süresi bitince aynı sayfadan yenisini oluşturup `HOMEOS_API_TOKEN` değerini değiştir. Anahtar sızdıysa hemen iptal et.
