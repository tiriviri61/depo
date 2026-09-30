# 03 — Barkod, koli ve el terminali

> Bu dosya depodaki fiziksel akışı tanımlar: ürün ve koli barkodları, etiket basımı,
> mal kabul, toptan sevk, sayım ve el terminali deneyimi.
> Veri modeli için [02](02-veri-modeli-ve-stok-motoru.md), ekran tasarımı için [05](05-ekranlar-ve-tasarim.md).

---

## 1. Barkod türleri

Depoda dört çeşit barkod dolaşır. Hepsi **Code128** ile basılır; MarketFlow'daki
`code128.ts` zaten 203 dpi termal yazıcıya göre ölçeklenmiş SVG üretiyor ve ZXing ile
okunabilirliği testli. Yeni bir barkod kütüphanesi eklenmez.

| Tür | Örnek | Kim üretir | Okutulunca ne olur |
|---|---|---|---|
| **Ürün barkodu** | `37523433` | MarketFlow kataloğu (`products.barcode`) | 1 adet ürün |
| **Koli tipi barkodu** | `KT-37523433-50` | Depo, ürün başına tanımlanır | O tipin iç adedi kadar ürün (ör. 50) |
| **Koli (LPN) barkodu** | `K26000123` | Depo, her fiziksel koliye tek | O koli ve içindekilerin tamamı |
| **Lokasyon barkodu** *(Faz 7, isteğe bağlı)* | `L-A-01-03` | Depo | Raf/göz seçimi |

Barkodu okutan ekran önce türünü çözer (önek ve kayıtla), sonra işlem kuralını uygular.
Çözümleme tek fonksiyondadır (`barkodCoz()`), web, terminal ve mobil aynısını kullanır.

### 1.1 Ürün barkodu — MarketFlow ile her zaman aynı

**Kural:** Depo kendi ürün tablosunu tutmaz. Ürün barkodunun tek kaynağı MarketFlow
kataloğudur. Depo ekranında barkod **düzenlenemez**; yalnız okunur, basılır, okutulur.
Katalogda barkod değişirse depo aynı anda yeni barkodu görür, çünkü aynı satırı okur.

Bugünkü durumda bu kuralın önünde üç engel var (ayrıntı [01](01-marketflow-eksik-analizi.md)):

1. Veritabanında barkod **tekil değil**. Tekilliği yalnız uygulama kodu koruyor. Faz 0'da
   `unique (org_id, barcode)` kısıtı eklenmeden önce mükerrer ve boş barkodlar temizlenir.
2. Barkodsuz ürün var. Depoya giren her ürünün barkodu olmak zorunda; barkodsuz ürün mal
   kabulde okutulamaz, o yüzden kabul ekranı "önce katalogda barkod ver" diye durdurur.
3. Bazı kanallar farklı kod kullanıyor: Teknosa aktarımı kısa barkodları "29…" ile başlayan
   bir EAN-13'e çeviriyor, Amazon gerçek GTIN istiyor. Bunlar **kanal kodu** olarak
   ayrıca saklanır (bkz. [02 §6](02-veri-modeli-ve-stok-motoru.md)); ürünün kendi barkodu
   değişmez.

**Ek barkodlar (yalnız okutma için):** Tedarikçi kutusundaki üretici barkodu (EAN) ile
bizim barkodumuz farklı olabilir. Mal kabulde her ürünü yeniden etiketlememek için
`depo.urun_ek_barkodlari` tablosu tutulur. Okutulan ek barkod ürüne çözülür ama
**hiçbir kanala gönderilmez, hiçbir etikete basılmaz**. Ana barkod her zaman MarketFlow'daki.

### 1.2 Koli tipi (ambalaj birimi)

Aynı ürünün standart kolisi: "RGB lamba — koli 50 adet". Ürün başına bir veya birkaç tip
olabilir (iç koli 10, dış koli 50). Alanlar: ürün, ad, iç adet, barkod, ölçü (en/boy/yükseklik),
brüt ağırlık, desi. Koli tipi barkodu okutulursa sistem "50 adet" sayar ama **hangi koli
olduğunu bilmez**. Bu yüzden toptan sevkte tek başına kullanılmaz; yalnız mal kabul ve
sayımda hız için kullanılır.

### 1.3 Koli (LPN — her fiziksel koliye tekil numara)

Gerçek bir depo için temel birim budur. Her fiziksel koliye mal kabulde (ya da koli
toplarken) **tekil** bir numara verilir ve etiketi basılıp üstüne yapıştırılır.

- **Biçim:** `K` + 2 hane yıl + 6 hane sıra (`K26000123`). Kısa, elle de yazılabilir,
  Code128 ile basılır. Sıra veritabanı dizisinden gelir, çakışmaz.
- **İçerik:** tek ürün (ör. 50 × RGB lamba) ya da karışık (toptan için hazırlanan koli).
  İçerik `depo.koli_icerik` tablosunda satır satır tutulur.
- **Durum makinesi:**

```
            mal kabul / koli topla
                    │
                    ▼
   ┌──────────► depoda ──────────┐
   │             │   │           │
   │  sevke ayır │   │ koli aç   │ sayımda bulunamadı
   │             ▼   ▼           ▼
   │        ayrildi  acildi   kayip (sayım farkı hareketi)
   │             │   (içerik tekil stoka döner)
   │  sevkten    │
   └─ geri çek ──┤
                 ▼ el terminaliyle sevk okutması
            sevk_edildi (geri dönüşsüz; iade → yeni mal kabul)
```

- **Kurallar:**
  - Bir koli **bir kez** sevk edilir. İkinci okutmada terminal kırmızı ekranla
    "Bu koli 12.10 14:32'de TS-0042 siparişiyle çıktı" der.
  - Koli içeriği değişmeden sevk edilir; eksik göndermek gerekirse önce "koli aç"
    yapılır, içerik tek tek okutulur.
  - Koli stoğu ayrı bir sayı değildir; koli içindeki ürünler ürünün **elde** stokunun
    parçasıdır. Koli yalnız "hangi ürün hangi kutuda" bilgisini taşır.

### 1.4 Palet *(sonraya)*

Palet de bir LPN'dir (`P26000012`), içinde koliler olur. İlk sürümde yok; veri modeli
koli içinde koliye izin verecek şekilde (`ust_koli_id`) kurulur, böylece sonra
eklenince göç gerekmez.

---

## 2. Etiket basımı

MarketFlow'da tarayıcıdan termal etikete basım zaten çalışıyor (gizli iframe +
`@page { size: … }`, 203 dpi'ye oturan barkod, ölçü testi etiketi). Depo aynı altyapıyı
kullanır, yeni şablonlar ekler:

| Etiket | Ölçü | İçerik |
|---|---|---|
| Ürün etiketi | 50×30 / 40×30 mm (mevcut) | Ürün barkodu + kısa ad |
| **Koli etiketi** | 100×100 mm (varsayılan), 100×150 | Büyük LPN barkodu, QR (LPN), ürün adı(ları), adet, mal kabul tarihi, varsa sipariş no |
| **Koli tipi etiketi** | 100×50 mm | Koli tipi barkodu + "50 adet" |
| **Sevk/irsaliye özeti** | A4 | Toptan sevkin koli listesi, imza alanı |
| Lokasyon etiketi *(Faz 7)* | 100×50 mm | Lokasyon barkodu, büyük harf kod |

**Doğrudan termal yazıcı (ZPL):** tarayıcı baskısı ölçü sapması ve yazdırma penceresi
yüzünden depoda yavaş kalabilir. Faz 4'te isteğe bağlı olarak ZPL üretimi eklenir
(Zebra ve ZPL uyumlu yazıcılar). Etiket yerleşimi tek yerde tanımlanır; hem SVG hem ZPL
aynı tanımdan çıkar. Yazıcı modeli kullanıcıdan öğrenilecek (bkz. [08](08-kararlar.md)).

---

## 3. Akışlar

Her akış hem web ekranında (masa başı) hem el terminalinde çalışır. Terminal okutma
odaklıdır; web ekranı düzeltme ve toplu iş içindir.

### 3.1 Mal kabul — "5000 RGB lamba geldi"

1. **Kabul fişi açılır:** tedarikçi, geliş tarihi, varsa ithalat konteyner kaydı
   (MarketFlow Çalışma Alanı'ndaki "İthalat & Ödeme" kaydına bağlanır), beklenen satırlar
   (ürün × adet). Beklenen yoksa "serbest kabul".
2. **Okutma:**
   - Ürün barkodu → +1
   - Koli tipi barkodu → +iç adet
   - "Adet gir" → büyük sayı klavyesi (ör. 5000), okutmadan da girilebilir
3. **Kolileme (isteğe bağlı ama önerilen):** "100 koli × 50 adet" gibi bölüşüm girilir ya da
   gelen koli sayısı kadar LPN üretilir. Etiketler aynı ekrandan basılır ve kolilere
   yapıştırılır.
4. **Fark kontrolü:** beklenen 5000, sayılan 4980 → eksik 20 satırı kırmızı görünür.
   Hasarlı adet ayrıca girilir ve **hasarlı** kovasına gider (satılabilir stoğa girmez).
5. **Onay:** tek düğme ("Kabulü tamamla"). Bu anda tek bir işlemde (transaction):
   - her satır için `mal_kabul` hareketi yazılır (elde +4980, hasarlı +20),
   - koliler `depoda` olur,
   - bakiye güncellenir ve kanal yayın kuyruğu tetiklenir,
   - maliyet seçeneği açıksa ağırlıklı ortalama alış maliyeti güncellenir (bkz. [08](08-kararlar.md)).
6. Onaydan sonra fiş **değişmez**. Hata varsa "düzeltme" hareketi ile ters kayıt yapılır.

### 3.2 Toptan sevk — el terminaliyle koli okutarak mal çıkışı

Ön koşul: [04](04-cari-toptan-fatura.md)'teki toptan sipariş **onaylanmış** olmalı
(onayda stok ayrılır).

1. Depocu terminalde **Sevk** → siparişi seçer. Sipariş barkodu (toplama listesinde basılı)
   okutulabilir ya da listeden seçilir.
2. Ekranda sipariş satırları: "RGB lamba — 600 adet — 0/12 koli".
3. **Koli okutulur (LPN):**
   - Koli bu siparişin ürünlerini içeriyor ve kalan adedi aşmıyorsa → yeşil, sayaç ilerler.
   - Koli zaten sevk edilmiş, başka siparişe ayrılmış, açılmış ya da içindeki ürün
     siparişte yoksa → kırmızı tam ekran, sesli uyarı ve **nedeni yazılı** gösterilir.
   - Koli fazla geliyorsa (sipariş 600, kolilerle 650 oluyor) → turuncu uyarı:
     "Bu koliyle 50 fazla olur. Koliyi aç ya da siparişi güncelle."
4. **Tek ürün okutma:** koli dışındaki tekil ürünler ürün barkoduyla okutulur.
5. **Son okutmayı geri al:** her zaman görünür; yanlış koliyi çıkarır.
6. **Sevki tamamla:** eksik varsa "eksik sevk" onayı istenir (kısmi sevk izinli). Tek işlemde:
   - okutulan koliler `sevk_edildi` olur,
   - `toptan_cikis` hareketleri yazılır (elde −, ayrılmış −),
   - sevk belgesi (irsaliye özeti) oluşur ve yazdırılabilir,
   - sipariş durumu `sevk_edildi` ya da `kismi_sevk` olur.
7. Her okutma bir **okutma olayı** olarak kaydedilir: cihaz, kişi, zaman, sonuç.
   Denetim ve hata ayıklama için "bu koliyi kim, ne zaman okuttu" her zaman görünür.

### 3.3 Pazaryeri siparişlerinin paketlenmesi

Bu akış MarketFlow'da **zaten var**: Kargo Hazırlık ekranı, çok kalemli pakette okutma
doğrulaması, termal kargo etiketi. Depo bunu değiştirmez, bağlanır:

- Paket "hazırlandı / etiket basıldı" olduğunda, pazaryeri henüz "kargoda" demese bile
  stok **fiziksel olarak çıkmış** sayılabilir. Bu erkenden çıkış kaydı isteğe bağlıdır;
  varsayılan olarak pazaryeri durumu beklenir (bkz. [02 §4](02-veri-modeli-ve-stok-motoru.md)).
- Toplama ekranındaki okutma mantığı (`toplama.ts`) depo terminalinin okutma mantığıyla
  aynı çözümleyiciyi kullanacak şekilde birleştirilir.

### 3.4 Sayım

- **Açılış sayımı (zorunlu, bir kez):** sistem canlıya alınmadan önce bütün ürünler sayılır.
  Sayılan değer açılış hareketi olarak yazılır. Sayılmamış ürün "stok doğrulanmadı"
  rozetini taşır ve **kanallara otomatik yayınlanmaz**. Bu, MarketFlow'da yaşanan
  "girilmemiş 0 stok ilanları kapatırdı" hatasını yapısal olarak engeller.
- **Dönemsel sayım:** seçili ürünler ya da lokasyon için. Sayım sürerken o ürünlerin
  hareketleri kilitlenmez; fark, sayımın başladığı andaki bakiyeye göre hesaplanır.
- Fark varsa **onaylı** `sayim_farki` hareketi yazılır, sebebi seçilir
  (fire, kayıp, yanlış kabul, bilinmiyor). Belirli bir eşiğin üstündeki farklar yönetici onayı ister.

### 3.5 İade kabul

Pazaryerinden gelen iade, ürün depoya dönüp **okutulunca** stoka girer:

1. Terminal → **İade kabul** → kargo/iade kodu ya da ürün barkodu okutulur.
2. Sistem bekleyen iadelerden eşleşeni bulur (ör. "Trendyol 1234567 — RGB lamba × 1").
3. Durum seçilir: **sağlam** → elde +1 · **hasarlı** → hasarlı +1 · **eksik parça** → hasarlı +1 ve not.
4. `iade_kabul` hareketi yazılır, kanal yayını tetiklenir.

Pazaryeri "iade" dediği halde 30 gün içinde okutulmayan iadeler raporda
"iade bekleniyor — gelmedi" olarak listelenir (kayıp takibi).

### 3.6 Diğer

- **Koli sorgu:** herhangi bir barkodu okut → ne olduğu, içeriği, durumu ve geçmişi.
- **Koli aç / birleştir:** koli içeriği tekil stoka döner ya da yeni koliye taşınır.
- **Transfer** *(çoklu depo gelirse)*: çıkış + giriş hareket çifti.
- **Hasar / fire bildir:** elde → hasarlı ya da stoktan düşüm, fotoğraf eklenebilir.

---

## 4. El terminali

### 4.1 Cihaz ve okuma yöntemi

Depo el terminalleri çoğunlukla Android tabanlı endüstriyel cihazlardır (Zebra TC serisi,
Honeywell EDA/CT, Urovo, Newland, Chainway vb.). Hepsi **klavye taklidi** (keyboard wedge)
modunu destekler: okuyucu barkodu bir klavye gibi yazar ve sonuna Enter ekler.
MarketFlow'un toplama ekranı bu yöntemi web'de ve mobilde zaten kullanıyor.

**Karar önerisi: terminal önce web'de, kendi tam ekran rotasında (`/terminal`) yapılır.**

| | Web rotası (önerilen) | Expo uygulama ekranı |
|---|---|---|
| Okuyucu desteği | Klavye taklidi, her cihazda | Klavye taklidi; DataWedge intent ya da kamera için yerel modül ve **yeni mağaza sürümü** gerekir |
| Güncelleme | Her push'ta otomatik deploy | JS değişikliği OTA; yerel modül mağaza incelemesi ister |
| Etiket basımı | Hazır (tarayıcı + ağ yazıcısı) | expo-print (AirPrint/paylaş) |
| Kurulum | Chrome'da "Ana ekrana ekle" (manifest hazır) | Mağazadan ya da APK |
| Çevrimdışı | Yok, yazılması gerekir | Yok, yazılması gerekir |
| Telefon kamerası | Web kamera API'siyle mümkün | Yerel modül gerekir |

Mobil uygulamaya sonra **Menü → Depo** altında stok sorgu, sayım ve onay ekranları gelir.
Tam okutma akışları gerekirse aynı saf mantık modülünden (`src/lib/depo/*`) beslenir.
Bu MarketFlow'un "mobil paritesi" kuralına uyar.

### 4.2 Terminal ekranı tasarım kuralları

- **Okutma öncelikli:** görünmez ama her zaman odakta bir okutma alanı vardır. Dokunmatik
  klavye açılmaz (`inputMode="none"`). Başka bir alana dokunulsa bile okutma geri odaklanır.
- **Sonuç tam ekran renk + ses + titreşim:** yeşil (eklendi), turuncu (dikkat, onay
  gerekiyor), kırmızı (reddedildi). Her sonuç **tek cümle nedenle** gelir; renk tek başına
  bilgi taşımaz (renk körlüğü, parlak depo ışığı).
- **Büyük hedefler:** eldivenle kullanılabilmesi için en az 56 px dokunma alanı, 18 px
  üzeri gövde yazısı, sayılar 32 px ve üstü, sabit genişlikli rakamlar.
- **Yüksek kontrast açık tema** varsayılan (depo ışığında koyu ekran yansır). MarketFlow'da
  sayfaya özel açık tema örneği zaten var (Kargo Hazırlık).
- **Sayaç her zaman görünür:** "7 / 12 koli · 350 / 600 adet".
- **Geri al** her zaman bir dokunuş uzakta.
- **Enter güvenliği:** okuyucunun gönderdiği Enter hiçbir zaman "tamamla" gibi bir
  eylemi tetiklemez. Tamamlama yalnız dokunarak ve onayla olur.
- **Kimlik:** her depocunun kendi hesabı olur; terminalde kişi ve cihaz adı üstte yazar.
  Paylaşılan tek hesap kullanılmaz (denetim izi kaybolur).

### 4.3 Kesinti ve çevrimdışı dayanıklılık

Depoda Wi-Fi kopabilir. Okutma kaybolmamalı, iki kez de sayılmamalıdır.

- Her okutma cihazda **tekil bir kimlik** (UUID) alır ve önce yerel kuyruğa yazılır.
- Sunucu okutmayı bu kimlikle kaydeder. Aynı kimlik ikinci kez gelirse yok sayılır ve
  ilk sonuç döner. Böylece bağlantı dönüp kuyruk yeniden gönderildiğinde çift sayım olmaz.
- Çevrimdışıyken yalnız "kuyruğa alındı" (gri) sonucu gösterilir; yeşil/kırmızı karar
  **sunucudan** gelir. Kuyrukta bekleyen varken "sevki tamamla" kapalıdır.
- İlk sürümde tam çevrimdışı çalışma hedeflenmez. Hedef, kısa kopmalarda veri kaybetmemek.

### 4.4 Test

- Okuyucu, testte klavye olayıyla taklit edilir: karakterler + Enter. Playwright ile
  terminal akışlarının uçtan uca testi (mal kabul, sevk, çift okutma, geri al,
  bağlantı kopması) yazılır.
- Barkod okunabilirliği mevcut ZXing testine koli etiketleri eklenerek ölçülür.
- Gerçek cihazda kabul testi: en az bir endüstriyel terminal + bir termal yazıcı ile
  100 koliyle deneme sevki (bkz. [06](06-fazlar-ve-kabul-kriterleri.md)).
