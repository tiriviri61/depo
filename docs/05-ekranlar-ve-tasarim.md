# 05 — Ekranlar ve tasarım

> Depo modülünün web paneli, el terminali, mobil uygulama ve Chrome uzantısındaki
> ekranları. Tasarım MarketFlow'un mevcut dilini izler; yeni bir tasarım sistemi kurulmaz.

---

## 1. Tasarım ilkeleri

1. **MarketFlow'un bileşenleri kullanılır:** kart, tablo, çekmece, modal, rozet, alan,
   segment, boş durum, iskelet, sayfa başlığı, sekmeli hub. Renkler ve yazı tipi
   (Geist, sayılar için tabular) değişmez. Tailwind'deki anlamlı renk adları kullanılır
   (`pos`, `neg`, `warn`, `info`, `accent`), yeni ham renk eklenmez.
2. **Her ekran "sıradaki iş ne?" sorusunu cevaplar:** bekleyen kabul, sevke hazır sipariş,
   fazla satış alarmı, eşleşmeyen satış ekranın üstünde sayılarıyla durur.
3. **Sayılar tek yerden:** ekranda gösterilen stok her zaman `depo.bakiye`'den gelir;
   istemci stok hesaplamaz.
4. **Yükleniyor, veri yok ve hata üç ayrı durumdur.** Hata hiçbir zaman boş liste gibi
   görünmez. Veri eksik okunduysa üstteki "eksik veri" şeridi çıkar (MarketFlow'da var).
5. **Geri alınamayan işlem iki adımlıdır:** kabul onayı, sevki tamamla, sayım farkı onayı,
   düzeltme hareketi. Onay metni **sonucu** söyler ("RGB lamba elde 5000 → 4980 olacak"),
   işlemin adını değil.
6. **Klavye ve okuyucu dostu:** masa başında USB okuyucu da klavye taklidiyle çalışır.
   Okuyucunun Enter'ı hiçbir zaman bir düğmeyi tetiklemez.
7. **Masaüstünde yoğun, terminalde iri:** web ekranları bilgi yoğun tablolar; terminal
   tek iş, iri sayı, iri düğme.

---

## 2. Bilgi mimarisi (web paneli)

Kenar menüye yeni **Depo** grubu eklenir. MarketFlow menüyü bilerek 23'ten 15 öğeye
indirdi; bu yüzden depo ekranları üç menü öğesinde sekmeli hub olarak toplanır:

```
Depo
├── Stok               (hub)  Ortak Stok · Hareketler · Sayım · Kanal Yayını · Eşleşmeyenler
├── Depo İşleri        (hub)  Mal Kabul · Koliler · Sevkiyat · İadeler · Etiketler
└── Toptan Satış       (hub)  Siparişler · Cariler
```

- **Terminal** menüde değil; Depo İşleri sayfasının sağ üstünde "Terminali aç" düğmesi ve
  doğrudan `/terminal` adresi.
- Mevcut **Stok & Fiyat** ekranındaki stok kısmı zamanla Ortak Stok'a taşınır; fiyat kısmı
  yerinde kalır. Geçiş bitene kadar iki ekran arasında bağlantı bulunur.
- Ayarlar sayfasına **Depo** bölümü: depolar, devreye alma anı, gölge/canlı mod,
  kanal ayarları (otomatik yayın, emniyet payı, tavan, kapatma anahtarı), roller, etiket yazıcısı.

---

## 3. Web ekranları

### 3.1 Ortak Stok (ana ekran)

- **Üst şerit (metrik):** toplam satılabilir adet · stok değeri (maliyetle) · kritik
  stoktaki ürün · fazla satış · doğrulanmamış ürün · eşleşmeyen satış. Her kutu tıklanınca
  listeyi o duruma süzer.
- **Tablo:** görsel · ad · barkod (mono) · **satılabilir** (büyük) · elde · ayrılmış ·
  hasarlı · kanal çipleri · son 30 gün satış hızı · kaç günlük stok · durum rozeti.
- **Kanal çipleri:** her kanal için küçük logo + yayınlanan sayı. Yayın hedefle aynıysa
  sade; farklıysa turuncu nokta; hata varsa kırmızı. Üzerine gelince son gönderim zamanı ve hata.
- **Süzgeçler:** arama (ad, barkod, okutma), durum (kritik, fazla satış, doğrulanmadı),
  kanal, marka/kategori.
- **Ürün çekmecesi** (sağdan açılır):
  - Özet: dört sayı, kritik eşik, hedef stok, stok doğrulama durumu.
  - **Kanal dağılımı:** "Son 30 gün: Trendyol 200 · idefix 100 · Amazon 100 · PTT 100 ·
    Site 0 · Toptan 0" (kullanıcının örneği tam olarak burada görünür).
  - **Hareketler:** zaman, tür, ± elde, ± ayrılmış, kaynak (tıklanınca sipariş/fiş), kişi/cihaz.
  - Koli tipleri ve depodaki koliler.
  - Kanal ilanları (eşleşme durumu).
- Toplu işlemler: seçili ürünlere etiket bas, sayım başlat, kritik eşik ayarla.

### 3.2 Hareketler (defter gezgini)

Tarih aralığı, ürün, tür, kaynak, kişi süzgeçleri. Sayfalı okuma (1000 satır sınırı).
CSV dışa aktarma. Her satırdan kaynağına gidilir. Düzeltme hareketi buradan, yetkiyle,
açıklama zorunlu olarak girilir.

### 3.3 Sayım

Sayım listesi (açık/tamamlanan) → sayım detayı: ürün, sistemin başlangıç sayısı, sayılan,
fark, sebep. Farklar renkli; eşik üstü fark yönetici onayı ister. "Açılış sayımı"
sihirbazı: bütün ürünler, ilerleme çubuğu, "sayılmadı" listesi.

### 3.4 Kanal Yayını

Kanal kartları: otomatik yayın açık/kapalı, son başarılı gönderim, bekleyen, hatalı,
kapatma anahtarı. Altında ürün × kanal ızgarası: hedef, gönderilen, durum, hata metni,
"şimdi yeniden dene". Gölge modda bütün ekran üstte bir şeritle "Gölge mod — kanallara
gönderilmiyor" der.

### 3.5 Eşleşmeyenler

Devreye alma sonrası ürüne bağlanamayan sipariş satırları ve eşleşmemiş kanal ilanları.
Mevcut **SKU Eşleştirme** ekranının deseniyle: satır → ürün seç → kaydet → motor hemen
uygular. "Şüpheli eşleşme" (adla bağlanmış) sekmesi.

### 3.6 Mal Kabul

Liste (taslak, sayılıyor, onaylandı) → detay: beklenen/sayılan/hasarlı satırlar, fark
vurgusu, koli üretme ("100 × 50"), etiket bas, onay. Okutma alanı sayfanın üstünde her
zaman açık (masa başı okuyucu).

### 3.7 Koliler

LPN listesi: barkod, içerik, durum, konum, bağlı kabul/sipariş. Koli detayı: içerik,
geçmiş (her okutma). Etiket yeniden bas, koli aç, koli birleştir.

### 3.8 Sevkiyat ve İadeler

Sevkiyat: onaylı toptan siparişlerin sevk durumu (okutulan/toplam), açık sevk oturumları,
tamamlanan sevkler ve belgeleri. İadeler: "iade bekleniyor" (pazaryeri dedi, mal gelmedi),
"iade alındı", "30 günü geçti — gelmedi".

### 3.9 Toptan Siparişler ve Cariler

[04](04-cari-toptan-fatura.md)'teki alanlarla. Sipariş detayı üç sütunlu:
sol satırlar, orta sevk durumu (koli koli), sağ fatura kartı + belgeler + zaman çizelgesi.

---

## 4. El terminali (`/terminal`)

Tam ekran, kenar menü ve asistan yok, oturum zorunlu. Ana ekran büyük kutucuklar:

```
┌──────────────────────────────┐
│  Depo Terminali   Ali · T-02 │
├──────────────┬───────────────┤
│  MAL KABUL   │     SEVK      │
│   2 bekliyor │   3 hazır     │
├──────────────┼───────────────┤
│    SAYIM     │     İADE      │
│              │   5 bekleniyor│
├──────────────┴───────────────┤
│        BARKOD SORGULA        │
└──────────────────────────────┘
```

Sevk ekranı örneği:

```
┌──────────────────────────────┐
│ ← TS-2026-0042  Yıldız Elek. │
├──────────────────────────────┤
│      7 / 12 koli             │   ← 48 px
│    350 / 600 adet            │
│ ████████████░░░░░░░          │
├──────────────────────────────┤
│ RGB Lamba        350/600  ●  │
│ Mouse Pad 90x40    0/40   ○  │
├──────────────────────────────┤
│ ✓ K26000123 · RGB Lamba · 50 │   ← son okutma, yeşil
├──────────────────────────────┤
│ [ Son okutmayı geri al ]     │   ← 56 px
│ [ Sevki tamamla… ]           │
└──────────────────────────────┘
```

Okutma sonucu tam ekranı 600 ms boyunca renklendirir (yeşil/turuncu/kırmızı),
sesle ve titreşimle birlikte. Kırmızıda ekran, depocu "Tamam"a basana kadar kalır ve
nedeni büyük yazar. Ayrıntılı kurallar: [03 §4.2](03-barkod-koli-el-terminali.md).

**Tema:** yüksek kontrastlı açık tema varsayılan (MarketFlow'da sayfaya özel açık tema
örneği Kargo Hazırlık'ta var). Terminalde koyu tema isteğe bağlı.

---

## 5. Mobil uygulama

MarketFlow'un mobil paritesi kuralına göre depo özellikleri mobile **uyarlanarak** gelir.
Mobil sekme sayısı 5 ile sınırlı; depo **Menü → Depo** altında durur:

| Ekran | İçerik |
|---|---|
| Depo özeti | Satılabilir toplam, kritik stok, fazla satış, bekleyen kabul/sevk |
| Stok sorgu | Arama + Bluetooth okuyucu ile barkod; ürünün dört sayısı ve kanal dağılımı |
| Hızlı sayım | Tek ürün say, farkı gönder (onay yöneticiye düşer) |
| Toptan onay | Taslak siparişleri gör, onayla/reddet |
| Alarmlar | Fazla satış ve kritik stok anlık bildirimleri |

- **Bugün** ekranına "Depo" kartı (fazla satış ve kritik stok sayısı).
- Web ve mobildeki stok uyarısı **aynı** fonksiyonla hesaplanır (bugün farklı; [01 S5](01-marketflow-eksik-analizi.md)).
- Kamera ile okutma yerel modül ve yeni mağaza sürümü gerektirir; ayrı bir sürüm olarak
  planlanır. O zamana kadar Bluetooth okuyucu (klavye taklidi) çalışır.
- Saf mantık (`barkodCoz`, stok sınıfı, sayım farkı) web ile paylaşılan `src/lib/depo/`
  modülünde durur ve yalnız göreli içe aktarma kullanır (MarketFlow'un paylaşım kuralı).

---

## 6. Chrome uzantısı (SONAR Asistan)

Uzantı pazaryeri panellerinde sipariş ve fiyat rozeti gösteriyor. Eklenecek:
ilan sayfasında **ortak stok rozeti** (satılabilir, kanalın yayın durumu, fark varsa uyarı).
Uzantı yalnız okur; stok değiştirmez.

---

## 7. Tasarım süreci

1. **Önce tel kafes (wireframe):** her ana ekran için metin tabanlı ya da basit HTML tel
   kafes, kullanıcı onayı.
2. **Sonra yüksek çözünürlüklü ekran:** MarketFlow bileşenleriyle, gerçek veriye benzeyen
   ama sahte olduğu açık veriyle (MarketFlow kuralı: demo veri gerçek gibi sunulmaz).
3. **Gözden geçirme:** `web-design-guidelines` skill'i ile erişilebilirlik ve arayüz kuralları
   denetimi; terminal ekranları gerçek cihazda depo ışığında denenir.
4. Her arayüz değişikliğinden sonra tarayıcı açılıp konsol okunur (MarketFlow kuralı).

Kullanılacak skill'ler ve istemler: [07](07-skill-ve-promptlar.md).
