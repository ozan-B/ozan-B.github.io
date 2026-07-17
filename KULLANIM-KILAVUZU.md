# Blog Kullanım Kılavuzu

Bu doküman, `ozan-B.github.io` adresindeki Chirpy temalı Jekyll blogunuzu nasıl güncelleyeceğinizi, yeni yazı ekleyeceğinizi ve yöneteceğinizi anlatır.

- **Canlı site**: https://ozan-b.github.io/
- **Repo**: https://github.com/ozan-B/ozan-B.github.io
- **Tema dokümantasyonu**: https://github.com/cotes2020/jekyll-theme-chirpy/wiki

---

## 1. Genel Klasör Yapısı

```
ozan-B.github.io/
├── _config.yml        → Site geneli ayarlar (başlık, dil, sosyal linkler, vb.)
├── _posts/             → Yayınlanmış blog yazıları
├── _drafts/             → Taslak yazılar (isteğe bağlı, henüz yok, kendiniz oluşturabilirsiniz)
├── _tabs/              → Üst menüdeki sekmeler (About, Archives, Categories, Tags)
├── assets/              → Görseller, favicon, self-hosted kütüphaneler
├── _data/               → Ek veri dosyaları (ör. iletişim linkleri)
└── .github/workflows/   → Otomatik deploy (GitHub Actions) tanımı
```

---

## 2. Yeni Blog Yazısı Ekleme

### 2.1 Dosya adı kuralı

`_posts/` klasörüne şu formatta bir dosya ekleyin:

```
_posts/YYYY-MM-DD-baslik-kelimeleri.md
```

Örnek: `_posts/2026-08-01-yeni-projem.md`

> Dosya adındaki tarih, front matter'daki `date` alanıyla **aynı gün** olmalı, aksi halde Jekyll yazıyı beklenmedik bir tarihte yayınlayabilir.

### 2.2 Front matter (dosyanın başındaki `---` bloğu)

```yaml
---
title: Yazı Başlığı
date: 2026-08-01 10:00:00 +0300
categories: [Kategori1, Kategori2]
tags: [etiket1, etiket2]
---

Yazının içeriği buradan başlar (Markdown).
```

Sık kullanılan ek alanlar:

| Alan | Açıklama |
|---|---|
| `pin: true` | Yazıyı ana sayfanın en üstüne sabitler |
| `image: /assets/img/kapak.jpg` | Yazının kapak görseli |
| `image: { path: ..., alt: "..." }` | Kapak görseli + alt metin |
| `comments: false` | Bu yazıda yorumları kapatır (genel ayar `_config.yml`'de) |
| `toc: false` | Bu yazıda içindekiler tablosunu kapatır |
| `math: true` | LaTeX/matematik formüllerini etkinleştirir |
| `mermaid: true` | Mermaid diyagramlarını etkinleştirir |

### 2.3 Taslak olarak yazma

Henüz yayınlamak istemediğiniz yazıları `_drafts/` klasörüne **tarihsiz** dosya adıyla koyabilirsiniz (ör. `_drafts/gelecek-yazi.md`). Taslakları yerelde görmek için:

```bash
bundle exec jekyll serve --drafts
```

Taslaklar `git push` ile GitHub'a gönderilse bile canlı sitede **görünmez** (Jekyll varsayılan build'de `_drafts` klasörünü işlemez).

### 2.4 Görsel ekleme

Görselleri `assets/img/` altına koyup yazı içinde şöyle kullanın:

```markdown
![Açıklama](/assets/img/gorsel-adi.png)
```

---

## 3. `_config.yml` — Site Ayarları

Değiştirmek isteyebileceğiniz başlıca alanlar:

| Alan | Ne işe yarar |
|---|---|
| `title` | Site başlığı (tarayıcı sekmesi, üst menü) |
| `tagline` | Başlığın altındaki kısa açıklama |
| `description` | SEO ve RSS açıklaması |
| `social.name` / `social.email` | Yazı altındaki yazar adı ve iletişim |
| `social.links` | Footer ve About sayfasındaki sosyal medya ikonları |
| `avatar` | Sol menüdeki profil fotoğrafı (`assets/img/avatar.jpg` gibi bir yol verin) |
| `theme_mode` | Boş bırakılırsa kullanıcı sistem temasına göre değişir; `light` veya `dark` ile sabitlenebilir |
| `comments.provider` | Yorum sistemi: `disqus`, `utterances` veya `giscus` (aşağıda detay var) |
| `analytics.google.id` | Google Analytics ölçüm ID'si |

Her değişiklikten sonra dosyayı kaydedip commit+push yapmanız yeterli (bkz. Bölüm 6).

---

## 4. Sekmeleri (Tabs) Düzenleme

`_tabs/` klasöründeki her `.md` dosyası üst menüde bir sekme oluşturur:

- `about.md` — Hakkımda sayfası
- `archives.md`, `categories.md`, `tags.md` — otomatik oluşturulan, genelde dokunulmaz

Yeni bir sekme eklemek için `_tabs/` altına front matter'lı yeni bir `.md` dosyası oluşturmanız yeterli:

```yaml
---
icon: fas fa-briefcase
order: 5
---

Sekme içeriği buraya.
```

`order` değeri menüdeki sırasını belirler.

---

## 5. Profil Fotoğrafı Ekleme (Avatar)

1. Fotoğrafı `assets/img/avatar.jpg` (veya `.png`) olarak kaydedin.
2. `_config.yml` içinde:
   ```yaml
   avatar: /assets/img/avatar.jpg
   ```
3. Commit + push.

---

## 6. Değişiklikleri Yayınlama (Deploy)

Bu site **GitHub Actions** ile otomatik deploy oluyor — `main` dalına her `push` yaptığınızda site birkaç saniye içinde otomatik güncellenir. Manuel bir "build" veya "deploy" komutuna gerek yok.

```bash
cd ozan-B.github.io
git add .
git commit -m "Yeni yazı: <başlık>"
git push origin main
```

Deploy durumunu kontrol etmek için:
- Repo sayfasında **Actions** sekmesi → "Build and Deploy" workflow'unun yeşil tik alıp almadığına bakın
- Ya da terminalden: `gh run list --repo ozan-B/ozan-B.github.io --limit 5`

Deploy tamamlandıktan sonra değişiklikler https://ozan-b.github.io/ adresinde görünür (bazen tarayıcı önbelleği yüzünden sert yenileme — `Ctrl+F5` — gerekebilir).

---

## 7. Yerelde Test Etme

Yazıyı yayınlamadan önce bilgisayarınızda önizlemek isterseniz:

```bash
cd ozan-B.github.io
bundle install          # bağımlılıklar değiştiyse
bundle exec jekyll serve
```

Ardından tarayıcıda **http://127.0.0.1:4000** adresini açın. `Ctrl+C` ile sunucuyu durdurabilirsiniz.

> Not: `jekyll serve --detach` komutu bu Windows kurulumunda çalışmaz (fork() desteklenmiyor), sunucuyu her zaman ön planda (`--detach` olmadan) çalıştırın.

---

## 8. Yorum Sistemi Ekleme (opsiyonel)

Şu an yorumlar kapalı. Açmak için `_config.yml` içindeki `comments:` bölümünü doldurun, örneğin **giscus** (GitHub Discussions tabanlı, ücretsiz) için:

1. https://giscus.app adresinden reponuz için ayar bilgilerini alın.
2. `_config.yml`:
   ```yaml
   comments:
     provider: giscus
     giscus:
       repo: "ozan-B/ozan-B.github.io"
       repo_id: "..."
       category: "..."
       category_id: "..."
   ```
3. Commit + push.

---

## 9. Sık Karşılaşılan Sorunlar

| Sorun | Çözüm |
|---|---|
| Yazı sitede görünmüyor | `date` alanının gelecekte olmadığından ve dosya adının doğru formatta olduğundan emin olun |
| Yerel `jekyll serve` hata veriyor | `bundle install` çalıştırdığınızdan emin olun; hata mesajını okuyup eksik gem'i `bundle add` ile ekleyin |
| Push sonrası site güncellenmedi | Repo **Actions** sekmesinden workflow'un başarılı olup olmadığını kontrol edin |
| Görsel görünmüyor | Yolun `/assets/img/...` ile başladığından ve dosyanın gerçekten o klasörde olduğundan emin olun |
| CSS/tema bozuk görünüyor | `git submodule update --init --recursive` çalıştırın (`assets/lib` bir alt modüldür) |

---

## 10. Hızlı Referans — Yeni Yazı Ekleme Akışı

```bash
cd ozan-B.github.io

# 1. Yeni dosya oluştur
#    _posts/2026-08-01-baslik.md

# 2. Front matter + içeriği yaz

# 3. (opsiyonel) yerelde kontrol et
bundle exec jekyll serve

# 4. Yayınla
git add _posts/2026-08-01-baslik.md
git commit -m "Yeni yazı: Başlık"
git push origin main
```

Birkaç saniye içinde https://ozan-b.github.io/ adresinde canlı olur.
