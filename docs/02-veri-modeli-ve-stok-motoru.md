# 02 — Veri modeli ve stok motoru

> Ortak stoğun nasıl tutulduğu, siparişlerden nasıl düştüğü ve kanallara nasıl yayınlandığı.
> Bu dosyadaki kurallar uygulama başladığında MarketFlow'un `docs/VERI-KURALLARI.md`
> dosyasına **"Depo ve ortak stok"** bölümü olarak taşınır. Kural değişirse önce orası güncellenir.

---

## 1. Değişmezler (asla bozulmayacak kurallar)

1. **Tek ürün kaynağı:** ürün, barkod, ad, KDV MarketFlow `products` tablosundadır.
   Depo ürün kopyalamaz, `products.id`'ye bağlanır. Ürün barkodu her yerde aynıdır.
2. **Stok bir sayı değil, bir defterdir:** her değişiklik `depo.hareketler` tablosuna
   **eklenen** bir satırdır. Satır güncellenmez, silinmez. Hata ters kayıtla düzeltilir.
3. **Bakiye defterin toplamıdır:** `depo.bakiye` hız için tutulur ama her zaman
   `sum(hareketler)`e eşittir. Saatlik bütünlük kontrolü bunu doğrular.
4. **Tek yazma kapısı:** bakiyeyi değiştiren tek yol `depo.hareket_yaz()` fonksiyonudur.
   Ekran, terminal, sipariş motoru, vitrin hepsi bundan geçer.
5. **Her hareketin tekil anahtarı vardır:** aynı iş iki kez gelirse ikinci kez yazılmaz
   (`unique (org_id, anahtar)`). Tekrar deneme, çift tıklama, yeniden senkron güvenlidir.
6. **Kanallardaki sayılar toplanmaz:** Trendyol 4500, HB 4500 gösteriyorsa stok 9000 değil
   4500'dür (MarketFlow kuralı §1a-3 aynen korunur).
7. **Bilinmeyen sıfır değildir:** sayılmamış ürünün stoğu "doğrulanmadı" durumundadır,
   0 olarak yayınlanmaz.
8. **Devreye alma anından önceki siparişler stoğa dokunmaz.**

---

## 2. Stok sayıları

Her ürün × depo için:

| Sayı | Anlamı | Örnek |
|---|---|---|
| **Elde** | Depoda fiziksel olarak duran sağlam ürün | 5000 |
| **Ayrılmış** | Satıldı ya da onaylandı, henüz depodan çıkmadı | 500 |
| **Satılabilir (ortak stok)** | `elde − ayrılmış` → kanallara yayınlanan sayı | 4500 |
| Hasarlı | Depoda ama satılamaz | 20 |
| İade bekleniyor | Pazaryeri "iade" dedi, mal henüz gelmedi (bilgi) | 3 |

**Kullanıcının örneği bu modelde:** 5000 RGB lamba mal kabulle girer → elde 5000.
İdefix 100, Amazon 100, Trendyol 200, PTT 100 satar → ayrılmış 500, **satılabilir 4500**
(bütün kanallara 4500 yayınlanır). Paketler kargoya çıktıkça elde ve ayrılmış birlikte
düşer: hepsi gidince elde 4500, ayrılmış 0, satılabilir yine 4500. Ekranda "ortak stok"
her zaman satılabilir sayıdır; elde ve ayrılmış yanında küçük yazılır.

Satılabilir **eksiye düşebilir** (iki kanal aynı dakikada son ürünü sattı). Pazaryeri
siparişi reddedilemez; sistem bunu gizlemez, "fazla satış" alarmı üretir ([§5.4](#54-fazla-satış)).

---

## 3. Tablolar (`depo` şeması)

Depo tabloları ayrı bir `depo` şemasında durur. ZYGA'nın `zyga` şeması aynı projede
aynı yöntemle çalışıyor. Her tablo `org_id` taşır, varsayılanı `current_org()`'dur;
okuma kiracı politikasıyla, **yazma yalnız fonksiyonlarla** yapılır (istemci doğrudan
insert/update yapamaz).

### 3.1 Çekirdek

```
depo.depolar            id, org_id, kod, ad, varsayilan, aktif
depo.ayarlar            org_id (pk), aktif, devreye_alma_ani, gecis_modu (golge|canli),
                        varsayilan_depo_id, fazla_satis_alarm_esigi, ...
depo.urun_ayarlari      org_id, product_id (pk) → products.id,
                        stok_dogrulandi (bool), dogrulama_zamani, kritik_esik,
                        hedef_stok, takip_yok (bool: hizmet/sanal ürünler)
depo.urun_ek_barkodlari org_id, barkod (unique), product_id, aciklama   -- yalnız okutma için

depo.hareketler         id (bigint), org_id, depo_id, product_id,
                        tur (enum, §3.2), elde_fark int, ayrilmis_fark int, hasarli_fark int,
                        kaynak (enum: siparis|site|toptan|mal_kabul|sayim|iade|duzeltme|acilis|koli),
                        kaynak_ref text,          -- ör. sipariş satırı id, fiş id
                        anahtar text,             -- tekillik: unique (org_id, anahtar)
                        koli_id, lokasyon_id,     -- isteğe bağlı
                        aktor uuid, cihaz text, aciklama text,
                        olusturuldu timestamptz
                        -- UPDATE/DELETE bir tetikleyiciyle reddedilir

depo.bakiye             org_id, depo_id, product_id (pk üçlü),
                        elde, ayrilmis, hasarli,
                        satilabilir (generated: elde - ayrilmis),
                        son_hareket_id, guncellendi
```

### 3.2 Hareket türleri

| Tür | elde | ayrılmış | Ne zaman |
|---|---|---|---|
| `acilis` | +n | | Açılış sayımı |
| `mal_kabul` | +n | | Kabul fişi onayı (hasarlı kısmı `hasarli_fark`) |
| `satis_ayir` | | +n | Pazaryeri/vitrin siparişi geldi |
| `satis_birak` | | −n | Sevk öncesi iptal |
| `satis_cikis` | −n | −n | Sipariş depodan çıktı |
| `toptan_ayir` / `toptan_birak` / `toptan_cikis` | | | Toptan siparişte aynı mantık |
| `iade_kabul` | +n (sağlam) | | İade okutuldu (hasarlıysa `hasarli_fark`) |
| `sayim_farki` | ±n | | Sayım onayı |
| `hasar` | −n | | Elden hasarlıya (`hasarli_fark` +n) |
| `duzeltme` | ±n | ±n | Yetkili ters kayıt, açıklama zorunlu |
| `transfer_cik` / `transfer_gir` | ∓n | | Çoklu depo gelirse |

### 3.3 Koli, kabul, sayım, okutma

```
depo.koli_tipleri       id, org_id, product_id, ad, ic_adet, barkod (unique), en, boy, yukseklik, agirlik, desi
depo.koliler            id, org_id, barkod (unique, K26000123), durum (enum), depo_id, lokasyon_id,
                        ust_koli_id (palet için), mal_kabul_id, toptan_siparis_id, sevk_id,
                        olusturuldu, sevk_zamani
depo.koli_icerik        koli_id, product_id, adet
depo.mal_kabuller       id, org_id, no, tedarikci, ithalat_kaydi_ref, durum, onaylayan, onay_zamani
depo.mal_kabul_satir    mal_kabul_id, product_id, beklenen, sayilan, hasarli
depo.sayimlar           id, org_id, kapsam, durum, basladi, bitti, onaylayan
depo.sayim_satir        sayim_id, product_id, sistem_baslangic, sayilan, fark, sebep
depo.okutmalar          id (cihazın ürettiği uuid → tekrar gelirse yok sayılır), org_id,
                        is_turu (kabul|sevk|sayim|iade|sorgu), is_id, barkod, cozum (urun|koli_tipi|koli|lokasyon|bilinmiyor),
                        sonuc (kabul|red|uyari), neden, aktor, cihaz, zaman
depo.lokasyonlar        (Faz 7) id, org_id, depo_id, kod, barkod, tur
```

### 3.4 Kanal yayını

```
depo.kanal_ayarlari     org_id, kanal (pk ikili), otomatik_yayin (bool), emniyet_payi, tavan,
                        kapatma_anahtari (bool), son_basarili, son_hata
depo.kanal_ilanlari     org_id, product_id, kanal, ilan_kimligi (kanalın istediği kod),
                        ek (jsonb: variant id, ASIN vb.), dogrulandi, kaynak, guncellendi
depo.yayin_kuyrugu      org_id, product_id (pk ikili), istendi, deneme, sonraki_deneme
depo.kanal_yayinlari    org_id, product_id, kanal, hedef_adet, gonderilen_adet, durum,
                        istek_ref (toplu işlem id), hata, gonderildi, dogrulandi
```

### 3.5 Cari, toptan, fatura

Ayrıntı [04](04-cari-toptan-fatura.md). Tablolar: `depo.cariler`, `depo.cari_adresler`,
`depo.cari_kisiler`, `depo.toptan_siparisler`, `depo.toptan_siparis_satir`,
`depo.sevkler`, `public.invoices`'a toptan bağı, `depo.fatura_dosyalari` + özel kova.

### 3.6 Yetki

```
depo.roller             org_id, user_id, rol (yonetici|satis|depocu|muhasebe|izleyici)
depo.yetki(p_is text) → boolean   -- her yazma fonksiyonunun ilk satırı
```

ZYGA vitrin panelinde aynı desen (`zyga.yetki('stok')`) çalışıyor; o örnek alınır.

---

## 4. Siparişten stok düşümü — hedef durum motoru

### 4.1 Neden basit tetikleyici olmaz

MarketFlow'daki senkronlar her 5 dakikada aynı satırları yeniden yazıyor, durum aynı
turda ileri-geri gidebiliyor, adet ve ürün bağı sonradan değişebiliyor ([01 §4](01-marketflow-eksik-analizi.md)).
"Sipariş geldi → stok −1" diye olay sayan bir tetikleyici bu ortamda **kesin** çift düşüm
ve kaçak üretir.

### 4.2 Yöntem: her sipariş satırı için "olması gereken" etkiyi hesapla, farkı yaz

Her pazaryeri sipariş satırının stok üzerinde **olması gereken** etkisi, satırın o anki
haline bakılarak hesaplanır:

```
hedef(satir) = { product_id → (ayrilmis, elde_cikis) }
```

| Stok sınıfı | Ayrılmış | Elde çıkış | Örnek ham durumlar |
|---|---|---|---|
| `aktif` | qty | 0 | Created, Picking, Invoiced, ReadyToShip, Unshipped, açık paket |
| `cikti` | 0 | qty | Shipped, Delivered, AtCollectionPoint, UnDelivered (geri dönüyor), iade talebi |
| `iptal` | 0 | 0 | Cancelled, UnSupplied (hiç gönderilmedi) |

Motor, o satır için defterde **şimdiye kadar yazılmış** toplam etkiyi okur (aynı
`kaynak_ref` ile), hedeften çıkarır ve yalnız **farkı** yazar:

```
fark = hedef − yazilmis
fark sıfırsa → hiçbir şey yazılmaz
```

Bu yöntemin getirdikleri:

- Aynı satır bin kez yeniden yazılsa da fark sıfırdır; çift düşüm olmaz.
- Durum teslim → iade → teslim gidip gelse de stok sınıfı hep `cikti`dır; hareket yazılmaz.
- Adet 1'den 3'e düzelirse +2 ayrılır. Ürün bağı X'ten Y'ye değişirse X'teki etki geri alınır, Y'ye yazılır.
- Motor çökse, kuyruk kaybolsa bile bir sonraki tam mutabakat turu her şeyi doğru yere getirir.

### 4.3 Stok sınıfını belirleyen fonksiyon

- **Ham pazaryeri durumundan** çalışır (satırın `raw` alanı), MarketFlow'un kaba
  5 değerli `status` alanından değil. Çünkü `UnDelivered` ile `UnSupplied` stok açısından
  zıt şeylerdir ama ikisi de bugün `iptal` yazılıyor.
- Saf fonksiyondur, `_shared` altında durur, her pazaryeri için tablo testiyle korunur.
- **Yalnız ileri gider:** `aktif → cikti`, `aktif → iptal`. `cikti` hiçbir zaman `aktif`e
  dönmez. Bilinmeyen yeni bir durum gelirse sınıf değişmez ve "tanınmayan durum" kaydı
  düşer (sessiz geri dönüş yok).
- `iptal → aktif` yalnız ham durum açıkça tanınan bir aktif duruma döndüyse olur
  (pazaryeri siparişi yeniden açtı) ve alarm üretir.

### 4.4 Tetikleme ve bekleme

1. `orders` üzerine **hafif** bir AFTER tetikleyicisi yazılır: yalnız `status`, `qty`,
   `product_id` ya da `raw` durum alanı **gerçekten değiştiyse** ve satır devreye alma
   anından sonraysa, `depo.siparis_kuyrugu`'na (org, satır id) ekler. Tetikleyici hiçbir
   koşulda hata fırlatmaz (senkronu asla durdurmaz).
2. Bir işçi (pg_cron, dakikada bir) kuyruktaki satırları **en az 2 dakikalık** olanlardan
   başlayarak işler (`for update skip locked`). Bekleme, senkronun "önce yaz, sonra ürüne
   bağla" arasındaki geçici bağsız anını atlatır.
3. Ürün bağı kopmuş (`product_id` boş) satırın mevcut etkisi hemen geri alınmaz.
   30 dakika boyunca boş kalırsa geri alınır ve satır "eşleşmeyen satış" kuyruğuna düşer.
4. **Tam mutabakat:** gece bir kez ve elle istendiğinde, devreye alma sonrasındaki bütün
   satırlar için hedef ↔ defter karşılaştırılır. Fark bulunursa yazılır ve raporlanır.
   Normal çalışmada bu rapor boş olmalıdır.

### 4.5 Kapsam dışı satırlar

- `ordered_at < devreye_alma_ani` olan her satır (geçmiş 180 gün, yeniden taramalar).
- Hakediş dolgusundan gelen satırlar (`line_uid` `settle-` ile başlayan).
- Depo modülü açık olmayan kiracılar (demo kiracı dahil).
- Ürün kartında `takip_yok` işaretli ürünler (hizmet, dijital).

### 4.6 İadeler

Pazaryeri "iade" dediğinde mal henüz depoda değildir. Stok sınıfı `cikti` kalır ve
satır **iade bekleniyor** listesine girer. Stok yalnız depocu iadeyi okuttuğunda
(`iade_kabul`) artar ([03 §3.5](03-barkod-koli-el-terminali.md)). Kısmi iade (3 adetten 1'i)
bu sayede doğru sayılır; pazaryerinin bütün satırı "iade" yazması stoku bozmaz.

### 4.7 Paketleme ile erken çıkış *(isteğe bağlı)*

Kargo Hazırlık'ta etiket basılan paket fiziksel olarak depodan çıkmıştır; pazaryeri
"kargoda" demesi saatler sürebilir. Seçenek açılırsa etiket basımı ilgili satırları
`cikti` sınıfına taşır (ileri yönde olduğu için güvenli). Bunun için paket → sipariş
satırı bağı kurulmalıdır ([01 G11](01-marketflow-eksik-analizi.md)). Varsayılan kapalı.

---

## 5. Kanallara yayın

### 5.1 Akış

```
hareket_yaz() → bakiye değişti → yayin_kuyrugu'na (org, ürün) eklenir (varsa üzerine)
        ↓  (işçi, dakikada bir; ürün 30 sn sakin kaldıysa)
her kanal için: hedef = min(max(satilabilir − emniyet_payi, 0), tavan)
        ↓
hedef ≠ son gönderilen ise → kanal bağdaştırıcısı → gönder → doğrula → kanal_yayinlari'na yaz
```

- **Birleştirme:** 50 sipariş arka arkaya gelse de ürün başına tek yayın gider.
- **Doğrulama:** kanal toplu işlem numarası veriyorsa sonucu ürün bazında okunur
  (Trendyol'da bu var). Vermiyorsa ilan senkronundan geri okunur. "İstek kabul edildi"
  başarı sayılmaz (MarketFlow kuralı aynen).
- **Hata:** kanal hatası ürünü kuyrukta tutar, artan beklemeyle yeniden dener,
  5 denemeden sonra alarm üretir. Bir kanalın hatası diğerlerini durdurmaz.
- **Kapatma anahtarı:** kiracı × kanal bazında. Kapalı kanala hiçbir şey gönderilmez.
- **Doğrulanmamış ürün yayınlanmaz** (açılış sayımı yapılmamış).
- **Eşleşmemiş ilan yayınlanmaz** (`kanal_ilanlari` kaydı yok ya da doğrulanmamış).

### 5.2 Kanal bağdaştırıcıları

| Kanal | Bugün | Yapılacak |
|---|---|---|
| Trendyol | Fiyat-stok fonksiyonu var, toplu sonucu doğruluyor | Ortak bağdaştırıcıya taşınır, kiracı bazlı anahtar |
| Hepsiburada | Stok yükleme var, doğrulama yok | Geri okuma eklenir |
| idefix | Fonksiyon var, ekrandan çağrılmıyor | Bağlanır |
| PTT AVM | Yok (yalnız aç/kapat) | Yeni API yapısıyla stok güncelleme |
| Amazon | Yok | SP-API ile adet güncelleme (SellerSKU); yalnız kendi gönderdiğimiz (FBM) ilanlar |
| Teknosa | Yok (teklif akışı hiç yazılmadı) | Teklif oluşturma + stok güncelleme |
| ZYGA vitrin | — | Gönderim yok; vitrin ortak stoğu doğrudan okur (§7) |

### 5.3 Emniyet payı ve gecikme

Sipariş girişi en iyi 5 dakikada bir. İki kanal aynı 5 dakikada son ürünleri
satabilir. Bunu azaltmak için:

- **Kanal emniyet payı:** her kanala `satilabilir − pay` gönderilir (ör. 2).
- **Az stok modu:** satılabilir belirli bir eşiğin altına düşünce (ör. 5) yalnız
  seçilen ana kanallar satışta kalır, diğerlerine 0 gider. İsteğe bağlı.
- **Gecikmeyi kısaltma:** Trendyol webhook'u (yalnız ilgili siparişi çekecek şekilde)
  ve aktif satırlar için daha sık durum kontrolü.

### 5.4 Fazla satış

Satılabilir eksiye düştüğünde:
1. Alarm: web bildirim merkezi + mobil anlık bildirim (MarketFlow'da altyapı var).
2. Ekran: eksiye düşüren son siparişler, kanalları, zamanları.
3. Öneri: hangi siparişin iptal/tedarik edilebileceği. **Otomatik iptal yapılmaz.**

---

## 6. Ürün ↔ kanal ilan eşlemesi

Ürünün ana barkodu her yerde aynı olmalıdır (MarketFlow kuralı). Yine de bazı kanallar
farklı kod ister ya da eski ilanlar farklı kodla açılmıştır. `depo.kanal_ilanlari`
bu farkı taşır:

| Kanal | `ilan_kimligi` | Nereden dolar |
|---|---|---|
| Trendyol | barkod | İlan anlık görüntüsü (var) |
| Hepsiburada | merchantSku (+ hbSku ek alanda) | HB ilan listesi (fonksiyon var) + alias'lar |
| idefix | barkod | idefix ilan listesi |
| PTT AVM | barkod | PTT ürün listesi |
| Amazon | SellerSKU (+ ASIN) | Amazon ilan raporu |
| Teknosa | offer_sku (+ üretilen EAN) | Mirakl teklif listesi |

- Eşleme ekranı "yayına hazır mı?" sorusunu ürün × kanal ızgarasında gösterir:
  eşleşti / eşleşmedi / çakışma (aynı ilan iki ürüne bağlı).
- Barkodu ana barkodla aynı olan ilan otomatik doğrulanır. Farklı olanı kullanıcı onaylar.
- Sipariş satırının ürüne bağlanması da bu tabloyu kullanır (alias tablosunun yerini zamanla alır).

---

## 7. ZYGA vitrin entegrasyonu

Vitrin aynı Supabase projesinde, `zyga` şemasında. Rezervasyon fonksiyonları sağlam
(sıralı kilit, tekrar korumalı, süre dolumu). Ortak stoğa geçiş:

1. Her aktif vitrin ürününe `marketflow_product_id` yazılır (zorunlu olur). Önce
   vitrin ürünleri MarketFlow kataloğunda barkodlu ürün olarak açılır.
2. `zyga.stock` tablosu bir **görünüme** döner: `musait = depo.bakiye.satilabilir`.
   Vitrin ekranları değişmeden çalışır.
3. `zyga.rezerve_et` → `depo.hareket_yaz(satis_ayir, anahtar='zyga-rez:'||id)`.
   Satılabilir yetmezse vitrin **reddeder** (pazaryerlerinden farklı olarak vitrin
   siparişi reddedebilir; bu doğru davranış).
4. `rezervasyon_kapat('iptal'|'suresi_doldu')` → `satis_birak`.
   `siparise_dondu` → ayırma siparişe devredilir; kargoya verilince `satis_cikis`.
5. Vitrin paneli stok **yazmaz**; stok düzeltmesi MarketFlow depo ekranından yapılır.
   Vitrinin `stok_hareket` tablosu salt okunur arşiv olarak kalır.
6. Vitrin siparişleri MarketFlow `orders`'a `site` kanalıyla da yazılır (kâr raporu için),
   ama stok etkisi **iki kez** uygulanmaz: `site` satırları stok motorunun kapsamı dışındadır,
   çünkü etkiyi rezervasyon zaten yazdı.

Bu değişiklikler ZYGA deposunda yapılır; o deponun kendi çalışma kuralları geçerlidir.

---

## 8. Bütünlük kontrolleri

MarketFlow'daki saatlik bütünlük kontrolü işine eklenir. Hepsi normalde **0 satır** döner:

| Kontrol | Ne yakalar |
|---|---|
| bakiye ≠ defter toplamı | Tek yazma kapısının dışından yazma |
| elde < 0 ya da ayrılmış < 0 | Mantık hatası |
| devreye alma sonrası satır: hedef ≠ yazılmış (15 dk'dan eski) | Motor gecikmesi ya da hata |
| koli içerik toplamı > elde | Koli / stok tutarsızlığı |
| yayınlanan ≠ hedef (1 saatten eski) | Kanal yayın sorunu |
| eşleşmeyen satış (devreye alma sonrası) | Barkod/ilan eşleme eksiği |
| tanınmayan pazaryeri durumu | Yeni durum dizesi |

---

## 9. Devreye alma: gölge mod → canlı

1. **Gölge mod (en az 7 gün):** defter, motor ve ekranlar çalışır; **kanallara yayın kapalıdır**.
   Açılış sayımı yapılmış ürünlerde her gün elle sayılan birkaç ürünle sistem sayısı
   karşılaştırılır.
2. **Kabul:** 7 gün boyunca mutabakat raporu boş, sayım farkı eşiğin altında.
3. **Canlı:** kanal bazında teker teker (önce en düşük hacimli kanal) otomatik yayın açılır.
   "Tavan 100" kuralı kaldırılır.
4. **Geri dönüş:** tek anahtarla bütün kanallara yayın durur. Defter çalışmaya devam eder,
   veri kaybolmaz.
