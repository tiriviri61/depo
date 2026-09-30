# 01 — MarketFlow eksik analizi (kod + tasarım)

> 30 Eylül 2026'da MarketFlow deposunun son hali (web, mobil, uzantı, edge fonksiyonları,
> 76 migration, docs) ve ZYGA vitrin deposu okunarak çıkarıldı. Dosya yolları MarketFlow
> deposuna göredir.
>
> **Not:** Bu depo herkese açık olduğu için güvenlik açıklarının ayrıntısı buraya yazılmadı.
> Onlar MarketFlow'un özel deposundaki `docs/DURUM.md` "Satış öncesi kapı" listesinde ve
> bu analizde "güvenlik" etiketiyle genel olarak geçiyor.

Önem dereceleri:
- **E (Engelleyici):** ortak stok bu çözülmeden doğru çalışamaz.
- **Ö (Önemli):** çözülmezse hatalı sayı ya da operasyon aksaması üretir.
- **İ (İyileştirme):** kalite, hız, tutarlılık.

---

## 1. Özet: bugün ne var, ne yok?

| Konu | Bugün | Ortak stok için gereken |
|---|---|---|
| Ürün kataloğu | `products`, barkod "tek anahtar" kuralı yazılı | Barkodun veritabanında tekil olması |
| Stok | `products.stock`, yalnız elle yazılıyor, çoğu ürün 0 | Hareket defteri + bakiye + açılış sayımı |
| Siparişten stok düşümü | **Hiç yok.** Hiçbir kod sipariş, iptal ya da iade gelince stoğa dokunmuyor | Durumdan türetilen, tekrarlanabilir düşüm motoru |
| Kanallara stok yayını | Trendyol + Hepsiburada elle, tek sayı; idefix fonksiyonu var ama ekrandan çağrılmıyor | Bütün kanallara otomatik, doğrulanan yayın |
| Kanallar arası eşitleme | Kural gereği **yok**; PTT/idefix'e "Trendyol stoğu, tavan 100" | Tek sayıdan bütün kanallar |
| Web sitesi (ZYGA) | Ayrı şemada kendi stoğu ve rezervasyonu var; MarketFlow'a bağ yok | Aynı ortak stoğu kullanması |
| Depo, lokasyon, koli, mal kabul, sayım | Yok | Hepsi |
| Cari, toptan sipariş | Yalnız Notion'dan gelen serbest tablolar | Tipli tablolar ve akış |
| Fatura saklama | Tabloda harici PDF linki; dosya saklama yok | Siparişe bağlı belge saklama |
| Rol / yetki | Kolon var, uygulanmıyor | Depocu, satış, muhasebe rolleri |
| Okutma | Toplama ekranında klavye taklidi okuyucu çalışıyor | Terminal akışları |
| Etiket | Code128 + QR + termal ölçekli basım çalışıyor | Koli ve lokasyon şablonları |

---

## 2. Katalog ve barkod

| # | Eksik | Kanıt | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|---|
| K1 | Barkod veritabanında **tekil değil**; tekilliği yalnız uygulama kodu koruyor | `products` üzerinde barkod kısıtı yok; `src/lib/db.ts` içinde elle kontrol | Mükerrer/boş barkod raporu → temizlik → `unique (org_id, barcode)` | E | 0 |
| K2 | Barkodsuz ürün açılabiliyor | SKU/barkod boş bırakılabiliyor (`trendyol-products-list`) | Depoya giren her ürün için barkod zorunlu; kabul ekranı barkodsuzu durdurur | E | 0 |
| K3 | Barkodlar GTIN değil (iç kod); Amazon GTIN istiyor, Teknosa aktarımı kısa barkodları başka bir EAN'a çeviriyor | `VERI-KURALLARI.md §2.1`; `teknosa-product-import` | Ürünün ana barkodu değişmez; kanalın istediği kod **kanal ilan kimliği** olarak ayrı tutulur ([02 §6](02-veri-modeli-ve-stok-motoru.md)) | Ö | 0-3 |
| K4 | `products`, `orders`, `invoices`, görünümler ve bazı RPC'lerin temel tanımı depoda yok (canlıda elle oluşturulmuş) | İlk migration Temmuz 2026; `create table products` hiçbir yerde yok | Canlı şema dökülüp "başlangıç" migration'ı olarak depoya alınır; yeni depo tabloları bunun üstüne kurulur | E | 0 |
| K5 | Ürünü sipariş satırına bağlayan fonksiyon önce **ürün adıyla** eşliyor. Kural "adla sessiz eşleşme yok" diyor | `link_orders_to_products` (migration `20260824190000`, ad eşleşmesi) | Stok için ad eşleşmesi kabul edilmez; adla bağlanan satır "şüpheli eşleşme" kuyruğuna düşer | E | 0 |
| K6 | Sipariş satırının ürün bağı gecikmeli kuruluyor ve SKU değişince sıfırlanıyor | Bağlama upsert'ten sonra ayrı RPC; sıfırlama tetikleyicisi | Stok motoru bekleme penceresiyle çalışır, geçici "bağsız" anı stok hareketine çevirmez ([02 §4.4](02-veri-modeli-ve-stok-motoru.md)) | Ö | 2 |
| K7 | Ürün → pazaryeri ilanı eşleme tablosu yok. İlan anlık görüntüsü yalnız Trendyol için ve ürün bağı yok | `marketplace_listings` yalnız Trendyol | `depo.kanal_ilanlari` (ürün × kanal × ilan kimliği), her kanalın ilan senkronundan dolar | E | 3 |
| K8 | Ürün kartında desi/ağırlık, koli içi adet, kritik stok eşiği yok | `products` kolonları | Koli tipi tablosu + ürün depo ayarları | Ö | 1, 4 |
| K9 | Anahtar normalleştirmesi iki farklı yerde farklı (biri Türkçe küçültme kullanıyor) | `src/lib/inventory.ts` vs `src/lib/productIdentity.ts` | Her yerde tek `productMatchKey` | İ | 0 |

## 3. Stok

| # | Eksik | Kanıt | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|---|
| S1 | Stok **hareket defteri yok**; kaydet mutlak değeri eziyor, eşzamanlı yazmada son yazan kazanır | `saveMasterStocks` (`src/lib/db.ts`) | Salt-ekleme hareket defteri + kilitli bakiye + tek yazma fonksiyonu | E | 1 |
| S2 | Ana stok güvenilmez: Temmuz ölçümünde ürünlerin çoğunda hiç girilmemişti (0) | `TEKNIK-KARARLAR.md §22.3, §24` | **Açılış sayımı** zorunlu; sayılmayan ürün "doğrulanmadı" rozetli ve yayına kapalı | E | 1 |
| S3 | "0 çünkü girilmedi" ile "0 çünkü bitti" ayırt edilemiyor; yayın koruması bu yüzden var | `wouldZeroLiveListing` | "Stok doğrulandı" bayrağı bu korumanın yerini alır | Ö | 1, 3 |
| S4 | Elde / ayrılmış / satılabilir ayrımı yok | tek `stock` kolonu | Üç ayrı sayı ([02 §2](02-veri-modeli-ve-stok-motoru.md)) | E | 1 |
| S5 | Web ve mobil stok uyarısı farklı hesaplanıyor | `mobile/src/lib/today.ts` ham stok; web `effectiveStock` | Uyarılar depo bakiyesinden, tek fonksiyonla | Ö | 1 |
| S6 | Ürün listesi okumalarında 1000 satır sınırı bazı yerlerde sayfalanmıyor | `loadProducts`, eşleşmeyen SKU, alias okumaları | Katalog ve stok okumaları sayfalı | Ö | 0 |

## 4. Sipariş girişi (stok düşümünün kaynağı)

| # | Eksik | Kanıt | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|---|
| G1 | Senkronlar her turda penceredeki **bütün satırları yeniden yazıyor** (Trendyol her 5 dk'da 2 gün, gece 180 gün; Teknosa her 5 dk'da bütün siparişler; idefix 30 gün) | `*-sync` fonksiyonları, cron tablosu | Stok motoru "olay" değil **"hedef durum"** ile çalışır; değişmeyen satır hiçbir şey yazmaz | E | 2 |
| G2 | Durum ileri-geri gidebiliyor: Trendyol aynı turda teslim → iade yazabiliyor; bilinmeyen durum "hazırlanıyor"a düşüyor; iki farklı HB eşlemesi `UnDelivered` için zıt sonuç veriyor | `trendyolDurum.ts`, `hepsiburada-sync`, `_shared/hb-statu.ts` | Stok sınıfı ham pazaryeri durumundan, test edilen tek fonksiyonla ve **yalnız ileri** yönde hesaplanır | E | 2 |
| G3 | Stok açısından yanlış eşleşen durumlar: Trendyol `UnDelivered` → iptal (mal çıktı, geri dönüyor); iade talebi mal dönmeden "iade" yazılıyor; kısmi iadede bütün satır "iade" oluyor | `trendyolDurum.ts`, `trendyol-sync` claims döngüsü | Stok motoru bunları "çıktı + iade bekleniyor" sayar; stoka dönüş yalnız **fiziksel iade okutmasıyla** olur | Ö | 2 |
| G4 | idefix'te satır adedi her zaman 1 yazılıyor | `_shared/idefix.ts` | Gerçek adet okunur (ciroyu da etkiler) | E | 0 |
| G5 | Amazon için zamanlanmış senkron yok; tur başına en çok 60 sipariş; teslim ve iade durumu hiç yazılmıyor | `amazon-sync`, `_shared/amazon.ts` | Cron + sayfalama + durum eşlemesi; FBA/FBM ayrımı ([08](08-kararlar.md)) | E | 0 |
| G6 | Teknosa iadeleri eşlenmiyor | `_shared/mirakl.ts` | İade durumu eşlenir | Ö | 2 |
| G7 | PTT'nin kullanıcı adı/şifre ile çalışan API'si Ekim sonunda kapanıyor; o gün PTT sipariş senkronu durur | `TEKNIK-KARARLAR.md §27.2` | API anahtarı yapısına geçiş **ilk iş** | E | 0 |
| G8 | Hepsiburada senkronu yalnız son ~24 saatin açık paketlerini görüyor; sevk öncesi iptal ancak gecelik işte ve 7 gün sonra fark ediliyor | `hepsiburada-sync`, `hepsiburada-statu-senkron` | Stokta "ayrılmış" bekleyen HB satırları için saatlik durum kontrolü | Ö | 2 |
| G9 | En kısa gecikme 5 dk; Trendyol webhook'u yazılmış ama canlıda değil (ve açılırsa her çağrıda 180 günlük tarama yapıyor) | `trendyol-webhook` | Webhook yalnız ilgili siparişi çekecek şekilde düzeltilip açılır; kanal emniyet payı gecikmeyi karşılar | Ö | 3 |
| G10 | `orders` tablosunda `updated_at` ve durum geçmişi yok | şema | Stok motorunun kendi olay kuyruğu + hareket defteri geçmişi taşır | Ö | 2 |
| G11 | Kargo hazırlık/paket okutma kayıtları sipariş satırına bağlı değil (pazaryeri paket numarasıyla tutuluyor) | `package_prep`, `package_pick` | Paket → sipariş satırı bağı; "etiket basıldı = fiziksel çıktı" seçeneği | İ | 5 |
| G12 | Geçmiş veri dolduran işler bazı satırları farklı satır kimliğiyle ya da kiracı bilgisi olmadan yazıyor | `trendyol-backfill`, `hepsiburada-backfill` | Stok motoru devreye alma anından eski satırları ve dolgu satırlarını görmez | Ö | 2 |
| G13 | Web sitesi (ZYGA) ve n11 için sipariş girişi yok; pazaryeri listesinde "site" değeri yok | `marketplace_id` enum | ZYGA doğrudan veritabanından bağlanır ([02 §7](02-veri-modeli-ve-stok-motoru.md)); n11 kapsam dışı ([08](08-kararlar.md)) | E | 2 |

## 5. Kanallara stok yayını

| # | Eksik | Kanıt | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|---|
| Y1 | Yayın elle, iki kanal, tek düğme | `src/pages/StockPrice.tsx` | Otomatik yayın kuyruğu (bekletmeli, birleştirmeli) | E | 3 |
| Y2 | PTT, Amazon, Teknosa için stok gönderimi yok; idefix fonksiyonu ekrandan çağrılmıyor | fonksiyon listesi | Her kanal için bağdaştırıcı (adapter) | E | 3 |
| Y3 | Hepsiburada gönderimi sonucu doğrulamıyor | `hepsiburada-price-inventory` | Gönderim sonrası geri okuma, kanal durumu tablosu | Ö | 3 |
| Y4 | Trendyol ve HB kapatma anahtarı tek satır ve kiracıdan bağımsız | `order_settings id=1` | Kiracı × kanal bazlı anahtar | Ö | 3 |
| Y5 | "Tavan 100" ve "Trendyol stoğunu kopyala" kuralı | `VERI-KURALLARI.md §2.2` | Eşitleme canlıya alınınca kural güncellenir, tavan kalkar | Ö | 3 |
| Y6 | Kural "yayın onaylı ve ayrı adımdır" diyor | `VERI-KURALLARI.md §1a-3` | Kural değişikliği: otomatik ama kayıtlı ve kanal bazlı kapatılabilir. **Önce kural dosyası güncellenir** | E | 0 |

## 6. Cari, toptan, fatura

| # | Eksik | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|
| C1 | Tipli müşteri tablosu yok (vergi no, adres, yetkili yok) | `depo.cariler` + Notion'dan aktarım | E | 6 |
| C2 | Toptan sipariş başlığı/satırı yok | `depo.toptan_siparisler` + satırlar | E | 6 |
| C3 | Fatura dosyası saklanmıyor (yalnız harici link) | Özel kova + `fatura_dosyalari` | E | 6 |
| C4 | Fatura kesme motoru yok, sağlayıcı bağdaştırıcısı iskelet halinde; mevcut hazırlık kodu tek KDV oranı kabul ediyor | İlk sürüm: dışarıda kes, yükle. Sonra entegratör | Ö | 6+ |
| C5 | Faturalar ekranı yalnız iki pazaryerini filtreliyor | "Toptan" filtresi ve kırılımı | İ | 6 |

## 7. Yetki ve güvenlik (genel)

| # | Eksik | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|
| Y-1 | Rol kolonu var ama hiçbir yerde uygulanmıyor; her üye her veriyi görüp değiştirebiliyor | Rol tablosu + veritabanı seviyesinde yetki fonksiyonu; depo yazmaları yalnız RPC'den | E | 0 |
| Y-2 | Davet ile kullanıcı açma ve kullanıcıya kiracı atama akışı yok | Davet akışı (depocu hesapları için) | E | 0 |
| Y-3 | Üyelik ve rol yönetimi sıkılaştırılmalı (güvenlik) | Rol değişikliği yalnız yöneticiye | E | 0 |
| Y-4 | Çok kiracılı izolasyonda açık maddeler (güvenlik) — MarketFlow `DURUM.md` listesi | Depo tabloları ilk günden kiracı-doğru; stok yazan uçlar listedekiler kapatılmadan açılmaz | Ö | 0 |
| Y-5 | Depolama kovası ve politika tanımları depoda değil | Migration'a alınır | Ö | 0 |

## 8. Tasarım ve ön yüz

| # | Eksik | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|
| T1 | Menüde depo bölümü yok; stok ekranı iki kanal sütunuyla sınırlı; "tek depo, tek ana stok" metni sabit | Yeni **Depo** grubu, sekmeli hub ([05](05-ekranlar-ve-tasarim.md)) | E | 1 |
| T2 | Stok hareket geçmişini gösteren ekran yok | Ürün çekmecesinde "Hareketler" sekmesi + defter gezgini | E | 1 |
| T3 | Uygulama kabuğu her sayfaya kenar menü + asistan ekliyor; tam ekran terminal rotası yok | Kabuksuz ama oturum korumalı `/terminal` dalı | E | 5 |
| T4 | Web düğmeleri 32-36 px; terminal için küçük | Terminal için ayrı boyut ölçeği (≥56 px) | Ö | 5 |
| T5 | Servis çalışanı (service worker) ve çevrimdışı kuyruk yok | Okutma kuyruğu + tekil okutma kimliği | Ö | 5 |
| T6 | Açılır bildirim bileşeni yalnız Çalışma Alanı'nda bağlı | Uygulama kabuğuna taşınır | İ | 1 |
| T7 | Mobil: kamera ile okutma yok (yerel modül → mağaza sürümü), sekme sınırı 5, yalnız koyu tema | Depo ekranları Menü altında; kamera okutma ayrı mağaza sürümü olarak planlanır | Ö | 5 |
| T8 | Tip denetiminde bilinen 35 hata var ve CI tip denetimi çalıştırmıyor; lint yok | Yeni kod hata eklemez; depo modülü için tip denetimi CI'a eklenir | İ | 0 |
| T9 | Uzantı (SONAR Asistan) pazaryeri panelinde stok göstermiyor | Ortak stok rozeti | İ | 7 |

## 9. ZYGA vitrin (web sitesi)

| # | Durum | Ne yapılacak | Önem | Faz |
|---|---|---|---|---|
| Z1 | Vitrinin kendi `stock` tablosu ve rezervasyon fonksiyonları var (sıralı kilit, tekrar korumalı, süre dolumu) — iyi tasarım | Aynı desen ortak stokta kullanılır | — | — |
| Z2 | Vitrin ürününde `marketflow_product_id` alanı var; notu "ortak stok kararı bekleniyor" diyor | Bu plan kararı verir: **ortak stok**. Alan zorunlu olur | E | 2 |
| Z3 | Vitrin SKU biçimi (`ZYG-XXX-000`) ve barkodu MarketFlow'dan bağımsız | Her vitrin ürünü MarketFlow kataloğunda barkodlu bir ürüne bağlanır | E | 0 |
| Z4 | Vitrin siparişleri MarketFlow sipariş tablosuna düşmüyor (kâr raporunda yok) | "site" kanalı eklenir, sipariş satırları MarketFlow'a yazılır | Ö | 2 |
| Z5 | Vitrin panelinin kendi stok ayarlama ekranı ve stok hareket tablosu var | Ortak stoktan sonra vitrin paneli stok **yazmaz**, yalnız okur; ayar MarketFlow depo ekranından | Ö | 2 |

---

## 10. Sonuç: sıralama

1. **Önce zemin (Faz 0):** şemayı depoya al, barkodu tekilleştir, ad eşleşmesini stoktan ayır,
   idefix adedi, Amazon senkronu, PTT API göçü, roller ve davet, kural dosyası güncellemesi.
2. **Sonra çekirdek (Faz 1-2):** hareket defteri, açılış sayımı, siparişten hedef durumlu düşüm, ZYGA bağlantısı.
3. **Sonra dışa yayın (Faz 3):** kanallara otomatik stok.
4. **Sonra fiziksel depo (Faz 4-6):** koli, etiket, terminal, cari, toptan, fatura.

Ayrıntılı iş listesi ve kabul ölçütleri: [06](06-fazlar-ve-kabul-kriterleri.md).
