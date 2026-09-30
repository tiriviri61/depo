# 07 — Skill'ler ve istemler (prompt)

> Uygulama sırasında kullanılacak hazır skill'ler, yazılacak proje skill'i ve her faz için
> yeni bir oturuma yapıştırılacak başlangıç istemleri.

---

## 1. Skill araması — ne yapıldı

30 Eylül 2026'da Vercel'in `skills` komut satırı aracıyla arandı.

- `skills find` kayıt defteri (skills.sh) bu bulut oturumunun ağ politikasında kapalı
  olduğu için arama sonuç döndürmedi. Oturumun ağ ayarlarında `skills.sh` izinli
  alanlara eklenirse kayıt defteri araması da yapılabilir.
- Bunun yerine bilinen skill depoları doğrudan listelendi:
  `vercel-labs/agent-skills`, `supabase/agent-skills`, `anthropics/skills`, `expo/skills`.
- Depo yönetimi (WMS), barkod ya da el terminali için hazır bir skill **bulunamadı**.
  Bu alan için proje skill'i yazılacak (§3).

---

## 2. Kurulacak skill'ler

Kurulum MarketFlow deposunun kökünde, proje düzeyinde yapılır (`.claude/skills/`):

```bash
npx skills add supabase/agent-skills --skill supabase --skill supabase-postgres-best-practices -a claude-code -y
npx skills add vercel-labs/agent-skills --skill web-design-guidelines --skill vercel-react-best-practices --skill vercel-composition-patterns --skill vercel-react-native-skills -a claude-code -y
npx skills add anthropics/skills --skill frontend-design --skill webapp-testing -a claude-code -y
npx skills add expo/skills --skill expo-ui --skill expo-router --skill expo-data-fetching -a claude-code -y
```

| Skill | Kaynak | Ne için | Faz |
|---|---|---|---|
| `supabase-postgres-best-practices` | Supabase | Şema, migration, RLS, tetikleyici, pg_cron, kuyruk; kilitleme ve tekillik | 0-6 (her veritabanı işi) |
| `supabase` | Supabase | Edge fonksiyonları, Storage (fatura kovası), Auth (davet, roller), Realtime | 0-6 |
| `frontend-design` | Anthropic | Web ekranlarının üretim kalitesinde, "yapay zekâ işi" görünmeyen tasarımı | 1, 4-6 |
| `web-design-guidelines` | Vercel | Arayüzü erişilebilirlik, odak, form ve etkileşim kurallarına göre denetleme | Her ekran sonunda |
| `vercel-react-best-practices` | Vercel | Büyük tablolar (binlerce ürün/hareket), yeniden çizim ve paket boyutu | 1, 3 |
| `vercel-composition-patterns` | Vercel | Paylaşılan terminal/okutma bileşenlerinin API'si | 5 |
| `webapp-testing` | Anthropic | Playwright ile terminal akışlarının uçtan uca testi | 5, 6 |
| `vercel-react-native-skills` | Vercel | Expo/React Native liste performansı, animasyon | 5 (mobil) |
| `expo-ui`, `expo-router`, `expo-data-fetching` | Expo | Mobil Depo ekranları, yönlendirme, veri çekme | 5 (mobil) |

Bu oturumda zaten hazır olanlar: `dataviz` (depo paneli grafikleri), `code-review`,
`security-review` (RLS ve roller), `simplify`, `run` (uygulamayı açıp denemek).
Hesabınızda etkin olan `mobile-ux-promax` mobil ekranlarda kullanılır.

Kurulmaması önerilenler: Vercel'e dağıtım skill'leri (`deploy-to-vercel`, `vercel-cli-with-tokens`,
`vercel-optimize`). MarketFlow kendi sunucusunda, GitHub Actions ile dağıtılıyor; Vercel kullanılmıyor.

---

## 3. Yazılacak proje skill'i: `depo-kurallari`

Faz 0'da MarketFlow deposuna `.claude/skills/depo-kurallari/SKILL.md` olarak yazılır.
Amacı, depo koduna dokunan her oturumun aynı değişmezleri bilmesi. İçeriği bu planın
özü olur:

- **Ne zaman yüklenir:** `depo` şemasına, stok hareketine, `orders` tetikleyicisine, kanal
  yayınına, barkod/koli/etikete, terminal ekranlarına ya da ZYGA stok bağlantısına dokunulurken.
- **Değişmezler:** [02 §1](02-veri-modeli-ve-stok-motoru.md)'deki 8 kural.
- **Yasaklar:** `depo.bakiye`'ye doğrudan yazmak; hareket satırı güncellemek/silmek;
  olay sayan (hedef durumsuz) stok tetikleyicisi yazmak; doğrulanmamış ürünü yayınlamak;
  ürün adıyla stok eşlemek; okuyucu Enter'ının bir eylemi tetiklemesi.
- **Kontrol listesi:** yeni hareket türü → tür tablosu + test; yeni kanal → bağdaştırıcı +
  geri okuma + kapatma anahtarı + mobil parite; yeni ekran → yükleniyor/boş/hata + konsol.
- **Doğrulama sorguları:** [02 §8](02-veri-modeli-ve-stok-motoru.md)'deki bütünlük kontrolleri.

İkinci bir küçük skill (`depo-terminal-ux`) terminal ekranı kurallarını ([03 §4.2](03-barkod-koli-el-terminali.md))
taşıyabilir; Faz 5'te gerekirse yazılır.

---

## 4. Faz istemleri

Her faz için yeni bir Claude Code oturumu açılır (MarketFlow deposu, gerekirse ZYGA
deposu eklenir). Aşağıdaki metin ilk mesaj olarak yapıştırılır. İstemler bilerek kısa;
ayrıntı bu depodaki plan dosyalarında.

### Faz 0

```
MarketFlow'a ortak stoklu depo sistemi ekliyoruz. Plan: tiriviri61/depo deposu
(PLAN.md ve docs/). Önce docs/01 ve docs/06'nın Faz 0 bölümünü oku.
Faz 0'ı uygula: canlı şemayı migration'a al, barkod temizlik raporunu çıkar (düzeltmeyi
bana sor), link_orders_to_products ad eşleşmesini stoktan ayır, idefix adedini ve Amazon
senkronunu düzelt, PTT API göçünü planla, rol/davet/yetki altyapısını kur, VERI-KURALLARI'na
"Depo ve ortak stok" bölümünü ekle, supabase ve vercel skill'lerini kur, depo-kurallari
proje skill'ini yaz. Her maddeyi kabul ölçütüyle doğrula. Gerçek veriyi test için değiştirme.
```

### Faz 1

```
Depo planı Faz 1: docs/02 §1-3 ve docs/06 Faz 1. depo şemasını, hareket defterini,
bakiye tablosunu ve depo.hareket_yaz() fonksiyonunu yaz (tekil anahtar, sıralı kilit,
güncelleme yasağı). Açılış sayımı sihirbazını ve Ortak Stok / Hareketler ekranlarını
MarketFlow bileşenleriyle yap. supabase-postgres-best-practices ve frontend-design
skill'lerini kullan; ekranları web-design-guidelines ile denetle. Kabul ölçütlerini ölçerek bitir.
```

### Faz 2

```
Depo planı Faz 2: docs/02 §4 ve §7, docs/06 Faz 2. Siparişten stok düşümünü hedef durum
yöntemiyle yaz: ham durumdan ileri-yalnız stok sınıfı fonksiyonu (her pazaryeri için tablo
testi), orders üzerinde değişim-yalnız tetikleyici, 2 dk beklemeli mutabakat işçisi, gece
tam mutabakat. Devreye alma anından önceki satırlara dokunma. ZYGA vitrinini ortak stoğa
bağla (zyga deposu). Gölge modda bırak, kanallara yayın yapma. Kullanıcının 5000 → 4500
örneğini test olarak yaz.
```

### Faz 3

```
Depo planı Faz 3: docs/02 §5-6, docs/06 Faz 3. Kanal ilan eşlemesini ve yayın kuyruğunu
yaz; Trendyol, Hepsiburada, idefix, PTT, Amazon (FBM), Teknosa bağdaştırıcılarını geri
okumalı doğrulamayla yap. Kiracı × kanal kapatma anahtarı, emniyet payı, az stok modu.
Kanalları tek tek aç; her birini açmadan önce bana sor.
```

### Faz 4-5

```
Depo planı Faz 4 ve 5: docs/03 ve docs/05 §4. Koli (LPN) modelini, barkodCoz() modülünü,
koli etiketlerini ve mal kabul ekranını yap; sonra kabuksuz /terminal rotasında mal kabul,
sevk, sayım, iade ve sorgu akışlarını yaz. Okutma olaylarını tekil kimlikle kaydet,
kesintide kaybetme. Playwright ile okuyucu taklitli uçtan uca test yaz (webapp-testing).
Mobilde Menü → Depo ekranlarını ekle (mobil paritesi).
```

### Faz 6

```
Depo planı Faz 6: docs/04. Cariler, toptan sipariş (durum makinesi + stok ayırma/çıkış),
toplama listesi, sevk belgesi ve fatura kartını (özel kovaya PDF/XML yükleme, XML'den alan
okuma) yaz. Notion'daki müşteri kayıtlarını tekrar güvenli aktar. Toptan satış pazaryeri
cirosuna karışmasın. Rol yetkilerini RLS testleriyle doğrula (security-review).
```
