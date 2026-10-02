# 06 — Fazlar ve kabul ölçütleri

> Her faz tek başına canlıya çıkabilir ve bir öncekinin üstüne kurulur. Bir faz ancak
> **kabul ölçütlerinin tamamı** ölçülerek sağlandığında biter ("çalışıyor gibi" yetmez).
> Sıralama [01](01-marketflow-eksik-analizi.md)'deki engelleyicilere göre yapıldı.

Kod MarketFlow deposunda yazılır (web `src/`, mobil `mobile/`, veritabanı `supabase/`).
ZYGA'ya ait değişiklikler ZYGA deposunda yapılır. Bu depo plan ve karar kaydıdır.

---

## Faz 0 — Zemin (engelleyicileri kaldır)

**Amaç:** ortak stok yazılmadan önce yanlış sayı üretecek her şeyi düzeltmek.

| İş | Çıktı |
|---|---|
| Canlı şemayı (products, orders, invoices, görünümler, RPC'ler, kovalar, politikalar) depoya "başlangıç" migration'ı olarak al | Depo = canlı şema |
| Barkod temizliği: mükerrer, boş, yer tutucu barkod raporu → kullanıcıyla düzelt → `unique (org_id, barcode)` | Tekil barkod kısıtı |
| ZYGA vitrin ürünlerini MarketFlow kataloğuna barkodlu ürün olarak aç ve bağla | Her aktif vitrin ürünü → MarketFlow ürünü |
| `link_orders_to_products`: ad eşleşmesini stok için geçersiz say, "şüpheli" işaretle | Adla bağlanan satır stoğa dokunmaz |
| idefix ve Hepsiburada ham durumu sakla (idefix adedi 1 kasıtlı: birim başına satır); Amazon zamanlanmış senkron + sayfalama + durum eşleme | Satır adedi ve durumu doğru |
| **PTT API anahtarı yapısına geçiş** (yeni yapı Ekim sonu; eski yapının kapanışı duyurulacak) | PTT senkronu kesintisiz |
| Rol tablosu + yetki fonksiyonu + davet akışı + üyelik tablosu izinleri | Depocu hesabı açılabilir |
| Katalog/stok okumalarında sayfalama; tek `productMatchKey` | 1000 satır sınırında kayıp yok |
| `VERI-KURALLARI.md`'ye "Depo ve ortak stok" bölümü (§1a-3 ve §2.2 değişiklikleri dahil) | Kural önce yazıldı |
| Skill'lerin kurulumu, proje skill'i `depo-kurallari` | [07](07-skill-ve-promptlar.md) |

**Kabul** (durum 2 Eki 2026 — kod yazıldı, canlıya alma onay bekliyor):
- [x] Canlı şemanın anlık görüntüsü depoda, canlıyla md5 eşit. Kalan: yalnız canlıda olan 6
      migration'ın MarketFlow `main`'ine push edilmesi (kullanıcıda).
- [x] Normalleştirilmiş barkodla mükerrer → 0 satır (ölçüldü). Kısıt yazıldı ve testli;
      canlıya uygulanması onay bekliyor.
- [ ] Depoda takip edilecek bütün ürünlerin barkodu dolu — 5 barkodsuz ürün, kullanıcı kararı bekliyor.
- [x] idefix: satır sayısı = birim sayısı (ölçüt düzeltildi; adet 1 kasıtlı). Ham durum saklanıyor (deploy bekliyor).
- [ ] Amazon — kullanıcı bağlantıyı duraklattı; Faz 3'e taşındı.
- [ ] PTT yeni yapıyla 48 saat hatasız senkron — PTT belgesi ve anahtar bekleniyor.
- [x] Depocu rolündeki test hesabı maliyet/kâr/fatura verisi **okuyamıyor** (RLS testi, canlı
      şema üzerinde PGlite'ta). Canlıda migration sonrası tekrar ölçülecek.

---

## Faz 1 — Stok çekirdeği

**Amaç:** defter, bakiye, açılış sayımı ve Ortak Stok ekranı. Henüz siparişten düşüm yok.

| İş | Çıktı |
|---|---|
| `depo` şeması: depolar, ayarlar, ürün ayarları, ek barkodlar, hareketler, bakiye | Migration + RLS |
| `depo.hareket_yaz()` (kilitli, sıralı, tekil anahtarlı), düzeltme ve güncelleme yasağı tetikleyicisi | Tek yazma kapısı |
| Bütünlük kontrolleri (bakiye = defter, eksi yok) saatlik işe eklenir | Kontrol raporu |
| Açılış sayımı sihirbazı + "stok doğrulandı" bayrağı | Sayım ekranı |
| Ortak Stok ekranı, ürün çekmecesi, Hareketler ekranı | Web |
| Stok uyarılarını tek fonksiyona taşı (web + mobil) | Parite |
| Bildirim bileşenini uygulama kabuğuna taşı | Kabuk |

**Kabul** (durum 2 Eki 2026 — kod yazıldı ve ölçüldü, canlıya alma onay bekliyor):
- [x] 10.000 hareketlik yük testinde bakiye = defter toplamı.
- [x] Aynı anahtarla 100 eşzamanlı çağrı → tek hareket (gerçek Postgres, 100 bağlantı).
      Ölçüm sırasında bir yarış hatası bulundu ve düzeltildi: kilitsiz "oku → yaz" sayımı
      eşzamanlı istekte iki kez yazıyordu. Kural: önce kilit, sonra oku.
- [x] Hareket satırı UPDATE/DELETE ile değiştirilemiyor (test; süper kullanıcıya bile).
- [x] Açılış sayımı tamamlanan ürünler "doğrulandı", kalanlar rozetli.
- [x] Ekranlar yükleniyor/boş/hata durumlarını ayrı gösteriyor; konsol temiz.

Ertelenen: ek barkodlar (Faz 4), kanal dağılımı ve satış hızı (Faz 2, depocu sipariş
finansını göremediği için defterden gelecek), stok değeri metriği (Faz 7).

---

## Faz 2 — Siparişten otomatik düşüm (+ ZYGA)

**Amaç:** bütün kanalların siparişleri ortak stoğu doğru düşürür. Kanallara yayın hâlâ kapalı (gölge).

| İş | Çıktı |
|---|---|
| Stok sınıfı fonksiyonu (ham durumdan, ileri-yalnız), her pazaryeri için tablo testi | `_shared/depo/stokSinifi.ts` |
| `orders` üzerinde değişim-yalnız tetikleyici → `depo.siparis_kuyrugu` | Hafif, hata fırlatmaz |
| Mutabakat işçisi (2 dk bekleme, 30 dk bağsız toleransı) + gece tam mutabakat | Hedef durum motoru |
| Devreye alma anı, kapsam dışı satırlar (dolgu, demo, takip dışı) | Ayar |
| İade bekleniyor listesi; iade kabul (web) | İade akışı |
| HB aktif satırlar için saatlik durum kontrolü; Teknosa iade eşlemesi | Durum doğruluğu |
| ZYGA: `marketflow_product_id` zorunlu, `zyga.stock` görünüm, rezervasyon → depo hareketi, vitrin siparişi → `orders` (`site`) | Vitrin ortak stokta |
| Eşleşmeyenler ekranı | Web |

**Kabul:**
- [ ] Kullanıcının örneği test verisiyle: elde 5000; idefix 100, Amazon 100, Trendyol 200, PTT 100 → satılabilir **4500**; sevk sonrası elde 4500, ayrılmış 0.
- [ ] Aynı satır 1000 kez aynı değerle yeniden yazıldığında hareket sayısı değişmiyor.
- [ ] Teslim → iade → teslim salınımında hareket yok.
- [ ] Ürün bağı X → boş → X (2 dk içinde) salınımında hareket yok.
- [ ] Sevk öncesi iptal → ayırma geri bırakılıyor.
- [ ] Vitrinde satılabilir 0 iken rezervasyon reddediliyor.
- [ ] **Gölge mod 7 gün:** gece mutabakat raporu her gün boş; günlük 5 ürünlük elle sayım farkı 0 (ya da açıklanmış).

---

## Faz 3 — Kanallara otomatik yayın

**Amaç:** satılabilir sayı her kanala otomatik ve doğrulanarak gider.

| İş | Çıktı |
|---|---|
| `depo.kanal_ilanlari` + her kanalın ilan listesinden doldurma + eşleme ızgarası | İlan eşlemesi |
| Yayın kuyruğu + işçi (birleştirme, artan beklemeli yeniden deneme) | Kuyruk |
| Bağdaştırıcılar: Trendyol, HB (+ geri okuma), idefix, PTT (yeni API), Amazon (FBM), Teknosa (teklif) | 6 kanal |
| Kiracı × kanal kapatma anahtarı, emniyet payı, tavan, az stok modu | Ayarlar |
| Kanal Yayını ekranı; fazla satış alarmı (web + mobil anlık bildirim) | Web + mobil |
| Trendyol webhook'u (yalnız ilgili siparişi çeken) | Gecikme ↓ |
| `VERI-KURALLARI.md §2.2` "tavan 100" kaldırılır | Kural |

**Kabul:**
- [ ] Her kanalda bir test ürününün stoğu değişince **2 dakika içinde** kanal paneli aynı sayıyı gösteriyor (geri okumayla doğrulandı).
- [ ] 50 ardışık siparişte ürün başına en fazla 1-2 gönderim (birleştirme çalışıyor).
- [ ] Bir kanalın API'si kapalıyken diğer kanallar etkilenmiyor; kapalı kanal açılınca kuyruk boşalıyor.
- [ ] Doğrulanmamış ürün hiçbir kanala gönderilmiyor.
- [ ] Kanallar tek tek açıldı (düşük hacimliden başlayarak), her biri 48 saat izlendi.

---

## Faz 4 — Barkod, koli, etiket, mal kabul

| İş | Çıktı |
|---|---|
| Koli tipleri, koliler (LPN), koli içeriği, durum makinesi | Tablolar + RPC |
| `barkodCoz()` (ürün / ek barkod / koli tipi / koli / lokasyon) | Paylaşılan saf modül |
| Koli ve koli tipi etiket şablonları (SVG), isteğe bağlı ZPL | Etiketler |
| Mal kabul ekranı (web): beklenen/sayılan/hasarlı, kolileme, onay | Web |
| İthalat kaydından kabul fişi açma | Çalışma Alanı bağlantısı |

**Kabul:**
- [ ] Basılan koli etiketi ZXing testinden ve gerçek terminal okuyucusundan geçiyor.
- [ ] 5000 adetlik kabul, 100 koliyle 10 dakikanın altında tamamlanıyor (gerçek deneme).
- [ ] Onaylanmış kabul değiştirilemiyor; düzeltme ters kayıtla.

---

## Faz 5 — El terminali

| İş | Çıktı |
|---|---|
| Kabuksuz `/terminal` rotası, terminal boyut ölçeği, açık yüksek kontrast tema | Kabuk |
| Terminal akışları: mal kabul, sevk, sayım, iade kabul, barkod sorgu | 5 akış |
| Okutma olayı tablosu, tekil okutma kimliği, yerel kuyruk | Kesinti dayanıklılığı |
| Ses/titreşim geri bildirimi, geri al | UX |
| Playwright uçtan uca testleri (okuyucu taklidi) | Test |
| Mobil: Menü → Depo ekranları (stok sorgu, hızlı sayım, onay) | Mobil parite |
| Paket → sipariş satırı bağı; "etiket basıldı = çıktı" seçeneği | Kargo Hazırlık bağı |

**Kabul:**
- [ ] Gerçek terminalde 100 koliyle deneme sevki: çift okutma reddi, geri al, eksik sevk uyarısı çalışıyor.
- [ ] Wi-Fi 30 sn kesilip gelince hiçbir okutma kaybolmuyor, hiçbiri iki kez sayılmıyor.
- [ ] Okuyucunun Enter'ı hiçbir durumda "tamamla"yı tetiklemiyor (test).

---

## Faz 6 — Cari, toptan satış, fatura

| İş | Çıktı |
|---|---|
| Cariler (+ Notion'dan tek seferlik, tekrar güvenli aktarım) | Tablo + ekran |
| Toptan sipariş: başlık, satır, durum makinesi, stok ayırma/bırakma/çıkış | Tablo + RPC + ekran |
| Toplama listesi, sevk belgesi (A4) | Belgeler |
| Fatura kartı: PDF/XML yükleme, özel kova, XML'den alan okuma ve karşılaştırma | Belge saklama |
| Faturalar ekranında "Toptan" filtresi; raporlarda toptan kırılımı | Rapor |
| *(6b, karara bağlı)* hafif cari ekstre + tahsilat bağlantısı | Ekstre |

**Kabul:**
- [ ] Toptan sipariş onayı satılabilir stoğu düşürüyor ve kanallara yansıyor.
- [ ] Terminalle sevk tamamlanınca elde ve ayrılmış doğru düşüyor, koliler "sevk edildi".
- [ ] Yüklenen fatura PDF'i siparişte görünüyor, yalnız yetkili rol açabiliyor (imzalı bağlantı).
- [ ] Toptan satış pazaryeri cirosunu değiştirmiyor.

---

## Faz 7 — Raporlar, lokasyon, uzantı, ince ayar

| İş |
|---|
| Depo paneli: günlük giriş/çıkış, satış hızı, kaç günlük stok, sipariş önerisi (yeniden sipariş noktası) |
| Stok değeri ve yaşlanma raporu |
| *(karara bağlı)* Lokasyon/raf barkodları, yerleştirme ve toplama rotası |
| *(karara bağlı)* Lot / son kullanma / seri no takibi |
| Chrome uzantısında ortak stok rozeti |
| Mobil kamera ile okutma (yeni mağaza sürümü) |
| Fatura kesme entegrasyonu (entegratör API'si) |

---

## Her fazda değişmeyen iş kuralları

- Yazmadan önce ilgili kural dosyası güncellenir (`VERI-KURALLARI.md`).
- Her yeni tablo ilk günden `org_id` + kiracı politikası taşır; yazma yalnız fonksiyonla.
- Yeni kod tip denetiminde yeni hata eklemez; depo modülü testleri CI'da koşar.
- Her arayüz değişikliğinde tarayıcı açılıp konsol okunur; mobil için `npx tsc --noEmit`.
- Pazaryeri eklenir/değişirse mobil aynı oturumda güncellenir (mobil paritesi).
- Gerçek kullanıcı verisi test için değiştirilmez; testler ayrı kiracı ya da geçici kayıtla yapılır.
- Faz sonunda MarketFlow `docs/DURUM.md` ve Obsidian `Saas/handoff.md` güncellenir.
