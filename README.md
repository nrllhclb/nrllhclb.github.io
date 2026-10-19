# Doğrulama Defteri — SystemVerilog & UVM Blogu

[Hugo](https://gohugo.io) ile yapılmış, GitHub Pages üzerinde **ücretsiz** yayınlanan bir blog.
Yazılar Markdown ile yazılır; SystemVerilog kodları otomatik renklendirilir.

---

## 1. Siteyi yayınlamak (bir kerelik, ~10 dakika)

1. **GitHub hesabı aç:** https://github.com/signup
2. **Yeni bir depo (repository) oluştur:** https://github.com/new
   - İsim olarak `KULLANICIADI.github.io` yaz (KULLANICIADI = GitHub kullanıcı adın).
     Böylece site doğrudan `https://KULLANICIADI.github.io` adresinde açılır.
   - "Public" seçili kalsın, başka hiçbir kutuyu işaretleme, **Create repository**'ye bas.
3. **Dosyaları yükle:**
   - Açılan sayfada **"uploading an existing file"** bağlantısına tıkla.
   - Bu klasördeki **tüm dosya ve klasörleri** sürükleyip bırak
     (`.github` klasörü dahil — gizli görünüyorsa aşağıdaki nota bak).
   - **Commit changes**'e bas.
4. **Pages'i aç:** Depoda **Settings → Pages** bölümüne git,
   *Build and deployment → Source* kısmında **GitHub Actions**'ı seç.
5. **Actions** sekmesine geç; "Siteyi yayınla" işi 1–2 dakikada yeşil tik alır.
   Siten artık yayında: `https://KULLANICIADI.github.io` 🎉

> **`.github` klasörü görünmüyorsa:** macOS'ta Finder'da `Cmd + Shift + .`,
> Windows'ta Gezgin'de *Görünüm → Gizli öğeler* ile gizli dosyaları göster.
> Alternatif olarak [GitHub Desktop](https://desktop.github.com) kullanabilirsin; gizli dosyaları otomatik yükler.

## 2. Kendi bilgilerini gir

`hugo.toml` dosyasını aç ve `DEĞİŞTİR` yazan satırları düzenle:
site adı, açıklama, adın, GitHub/LinkedIn bağlantıların.
`content/hakkinda.md` dosyasını da kendine göre yaz.

GitHub üzerinde bir dosyayı düzenlemek için dosyaya tıkla → kalem ✏️ simgesi → değiştir → **Commit changes**.
Her kaydetmede site otomatik olarak yeniden yayınlanır.

## 3. Yeni yazı eklemek

`content/yazilar/` klasörüne yeni bir `.md` dosyası ekle. Dosya adı adresi belirler:
`uvm-sequence-ler.md` → `https://.../yazilar/uvm-sequence-ler/`

Dosyanın başı şöyle olmalı:

````markdown
---
title: "UVM Sequence'lar"
date: 2026-10-26T10:00:00+03:00
description: "Listelerde görünecek kısa özet."
kategoriler: ["UVM"]          # veya ["SystemVerilog"]
etiketler: ["sequence", "stimulus"]
draft: false                  # true ise yayınlanmaz
---

Yazının metni buradan başlar...

## Alt başlık

```systemverilog
class benim_seq extends uvm_sequence #(benim_islem);
  // ...
endclass
```
````

Faydalı ipuçları:

- Kod bloğuna dosya adı başlığı eklemek için: ` ```systemverilog {title="driver.sv"} `
- Bilgi kutusu için satırı `> ` ile başlat.
- Resimleri `static/resimler/` içine koy, yazıda `![açıklama](/resimler/dosya.png)` ile kullan.
- **Gelecek tarihli yazılar yayınlanmaz** — tarih geldiğinde bir sonraki yayında görünür.

## 4. Bilgisayarında önizleme (isteğe bağlı)

[Hugo'yu kur](https://gohugo.io/installation/) (extended sürüm, 0.146 veya üstü), sonra bu klasörde:

```bash
hugo server -D
```

Tarayıcıda `http://localhost:1313` adresini aç; dosyaları kaydettikçe sayfa anında güncellenir.
Komut satırından yeni yazı taslağı oluşturmak için: `hugo new yazilar/yeni-yazi.md`

## Klasör yapısı

```text
content/            Yazılar ve sayfalar (Markdown)
  yazilar/          Blog yazıları
  hakkinda.md       Hakkında sayfası
layouts/            HTML şablonları
assets/css/         Tasarım (renkler: main.css içindeki :root)
static/             Olduğu gibi kopyalanan dosyalar (resim, favicon)
hugo.toml           Site ayarları
.github/workflows/  Otomatik yayın ayarı
```

## İleride: kendi alan adın

Bir alan adı (ör. `svuvm.com`) alırsan **Settings → Pages → Custom domain** kısmına yazman
ve alan adı sağlayıcında GitHub'ın gösterdiği DNS kayıtlarını eklemen yeterli.
