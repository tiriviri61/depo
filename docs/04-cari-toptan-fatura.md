# 04 — Cari, toptan sipariş ve fatura

> Toptan satışın veri modeli ve iş akışı: müşteri (cari) kayıtları, toptan siparişler,
> sevk ve siparişin içinde saklanan fatura.
> Fiziksel sevk okutması için [03 §3.2](03-barkod-koli-el-terminali.md).

---

## 1. Bugün MarketFlow'da ne var?

| Alan | Var olan | Eksik |
|---|---|---|
| Müşteri | Çalışma Alanı'nda Notion'dan gelen "Müşteriler & Cari Hesaplar" tablosu (ad, not, tip) | Vergi no, vergi dairesi, adres, iletişim kişisi, bakiye; tipli ve ilişkisel tablo |
| Toptan satış | Çalışma Alanı'nda "Şirket Toptan Satış & Tahsilatlar" (yalnız tahsilat kayıtları, dekont) | Sipariş başlığı, ürün satırları, adet, KDV, durum, stok bağlantısı |
| Fatura | `invoices` tablosu (numara, tutar, tarih, harici PDF linki, sipariş no boş olabilir) | Dosya saklama (PDF/XML), toptan siparişe bağ, fatura kesme motoru |
| Alıcı bilgisi | Pazaryeri siparişlerinde satır bazlı alıcı adı/vergi no | Tekil müşteri kaydı |

MarketFlow'un kendi kuralı "finansal veri ilişkisel ve sorgulanabilir tabloda durur" der;
gider tablosu da bu gerekçeyle Çalışma Alanı'ndan tipli tabloya taşınmıştı. Aynı yol
izlenir: **cari ve toptan sipariş tipli tablolarda** tutulur, Çalışma Alanı'nda
"sistem tablosu" olarak da görünür (gider tablosu gibi).

Toptan siparişler **pazaryeri `orders` tablosuna yazılmaz.** O tablo pazaryeri cirosu,
komisyon ve sipariş başı kargo hesabını besler; toptan satış karışırsa kârlılık raporları bozulur.

---

## 2. Cari (müşteri) kaydı

### 2.1 Alanlar

| Grup | Alanlar |
|---|---|
| Kimlik | Kod (otomatik, `C0001`), tip (kurumsal / şahıs / bayi / pazaryeri dışı perakende), ünvan veya ad soyad, kısa ad |
| Vergi | VKN (10 hane) veya TCKN (11 hane), vergi dairesi, e-Fatura mükellefi mi (sorgu sonucu + tarih) |
| İletişim | Telefon, e-posta, yetkili kişiler (ad, görev, telefon, e-posta) |
| Adresler | Fatura adresi (tek), sevk adresleri (birden çok, varsayılan işaretli) |
| Ticari | Fiyat listesi / iskonto oranı, vade (gün), kredi limiti, para birimi, not |
| Durum | Aktif / pasif, oluşturma ve güncelleme zamanı, oluşturan kişi |

- Vergi numarası biçim denetiminden geçer (10/11 hane, TCKN algoritması). Aynı vergi
  numarasıyla ikinci kayıt açılamaz (org içinde tekil).
- Silme yoktur; kullanılmayan cari **pasife** alınır. Geçmiş siparişler ve faturalar
  cari kaydını kaybetmemeli.
- Notion'dan gelen eski "Müşteriler & Cari Hesaplar" kayıtları tek seferlik bir aktarımla
  bu tabloya taşınır. Eski kimlik saklanır ki aktarım iki kez çalışırsa mükerrer oluşmasın.

### 2.2 Cari ekranı

Liste (arama, tip, aktiflik, bakiye) ve detay. Detayda sekmeler:
**Özet** (son siparişler, açık sipariş, toplam alım) · **Siparişler** · **Faturalar** ·
**Hareketler/Ekstre** *(Faz 6b, isteğe bağlı)* · **Belgeler** (sözleşme, vergi levhası).

### 2.3 Cari ekstre ve tahsilat *(karar bekliyor)*

Tam cari hesap (borç/alacak, vade takibi, tahsilat) ayrı bir muhasebe modülüdür. İki seçenek:

- **Hafif:** her sipariş/fatura tutarı borç, her tahsilat alacak; bakiye ve vadesi geçmiş
  tutar gösterilir. Çalışma Alanı'ndaki tahsilat kayıtları (dekontlarıyla) cariye bağlanır.
- **Yok:** muhasebe dış programda kalır, MarketFlow yalnız sipariş ve fatura tutar.

Öneri: hafif sürüm Faz 6b'de. Bkz. [08](08-kararlar.md).

---

## 3. Toptan sipariş

### 3.1 Başlık ve satırlar

**Başlık:** sipariş no (`TS-2026-0001`), cari, sevk adresi, sipariş tarihi, istenen sevk
tarihi, durum, para birimi, iskonto, not, müşteri sipariş/PO no, oluşturan.

**Satır:** ürün (MarketFlow kataloğundan, barkodla seçilir), adet, birim (adet / koli tipi),
birim fiyat, iskonto %, KDV % (ürünün KDV oranından gelir, satırda değiştirilebilir),
satır tutarı, sevk edilen adet (sevkten hesaplanır).

- Ürün seçimi barkod okutarak da yapılabilir (masa başı USB okuyucu).
- Birim "koli" seçilirse adet = koli sayısı × koli tipinin iç adedi.
- Tutarlar sunucuda hesaplanır; arayüzdeki hesap yalnız önizlemedir.

### 3.2 Durum makinesi ve stok etkisi

```
taslak ──onayla──► onaylandi ──sevk başladı──► hazirlaniyor ──sevki tamamla──► sevk_edildi ──fatura eklendi──► tamamlandi
   │                  │                            │                               │
   │ sil              │ iptal                      │ kısmi sevk + kapat             └─► (iade: yeni mal kabul, bkz. 03 §3.5)
   ▼                  ▼                            ▼
 (silinir)         iptal                      kismi_sevk
```

| Geçiş | Stok etkisi |
|---|---|
| taslak → onaylandi | Her satır için **ayrılmış +adet** (satılabilir düşer, kanallara yayınlanır) |
| onaylandi → iptal | Ayrılmış geri bırakılır |
| sevk okutmaları + "sevki tamamla" | Sevk edilen adet: **elde −, ayrılmış −** |
| kısmi sevk + "kalanı kapat" | Kalan ayrılmış serbest bırakılır |
| onaylı siparişte satır değişikliği | Fark kadar ayırma ayarlanır (ör. 600 → 500: ayrılmış −100) |

Yetersiz satılabilir stokta onay **reddedilmez ama uyarı verir** ("RGB lamba için 600
istendi, satılabilir 450. Yine de onaylarsan satılabilir −150'ye düşer ve kanallarda
tükenmiş görünür."). Karar yönetici yetkisindedir.

Bütün stok etkileri [02](02-veri-modeli-ve-stok-motoru.md)'deki tek hareket fonksiyonundan
geçer, kendi anahtarıyla tekilleştirilir (aynı onay iki kez uygulanamaz).

### 3.3 Belgeler

- **Toplama listesi (A4):** sipariş barkodu, satırlar, önerilen koliler (hangi LPN'ler
  depoda). Depocu bu listeyle toplar, terminalle okutur.
- **Sevk belgesi / irsaliye özeti (A4):** sevk edilen koliler ve adetler, teslim alan imza alanı.
- **e-İrsaliye:** şirketin e-İrsaliye yükümlülüğü varsa yasal belge entegratörden kesilir.
  MarketFlow'un bu belgeyi kesmesi ilk sürümde yok; kesilen belge siparişe dosya olarak
  eklenir. Yükümlülük durumu kullanıcıdan öğrenilecek (bkz. [08](08-kararlar.md)).

---

## 4. Fatura — siparişin içinde saklanır

### 4.1 Tek kaynak kuralı

MarketFlow'un kuralı: **bir siparişin faturasını tek kaynak keser**, yoksa satış GİB'e
iki kez bildirilir. Pazaryeri siparişlerinin faturası bugün dış fatura panelinde
kesiliyor ve MarketFlow'un fatura motoru henüz yazılmadı. Toptan ve "serbest" faturalar
için karar "sonra" diye bırakılmıştı.

**Öneri (Faz 6):** toptan faturası dış fatura panelinde kesilir, **PDF ve XML'i
toptan siparişe yüklenir**. MarketFlow belgeyi saklar, gösterir, paylaşır ama kesmez.
Fatura kesme API'si (entegratör) sonra ayrı bir faz olarak gelir; veri modeli bunu
bekleyecek şekilde kurulur.

### 4.2 Veri

- Mevcut `invoices` tablosu kullanılır: `marketplace = 'toptan'`, `order_number = TS-…`,
  yön `giden`, tür `efatura`/`earsiv`, sağlayıcı `manual` (elle yüklendi) ya da entegratör adı.
  Tabloya **toptan sipariş bağlantısı** eklenir.
- **Fatura dosyaları** için özel (herkese kapalı) bir depolama kovası açılır.
  Yol: `{org}/{fatura_id}/{dosya}`. Dosyalar kısa ömürlü imzalı bağlantıyla açılır;
  bu desen MarketFlow'daki gider ekleriyle aynıdır.
- `fatura_dosyalari` tablosu: fatura, tür (pdf/xml/diğer), dosya yolu, boyut, sha256,
  yükleyen, zaman. Aynı dosya iki kez yüklenirse sha256 ile fark edilir.
- XML yüklenirse (UBL-TR), fatura no, tarih, ETTN, toplam ve KDV **XML'den okunur** ve
  elle girilen değerlerle karşılaştırılır. Uyuşmazlık uyarı olarak gösterilir.

### 4.3 Ekran

Toptan sipariş detayında **Fatura** kartı:
- Fatura yoksa: "Fatura bekleniyor" + "PDF/XML yükle" (sürükle bırak, 25 MB, yalnız pdf/xml/görsel).
- Fatura varsa: numara, tarih, tutar, ETTN; belge önizleme; indir; müşteriye e-posta
  ile gönder *(sonraya)*.
- İade faturası aynı karta ikinci belge olarak eklenir (yön: iade).

Faturalar sayfası (mevcut) "Toptan" filtresini kazanır. Toptan faturalar ciroya
pazaryeri cirosu gibi **karıştırılmaz**; ayrı bir "Toptan satış" kırılımında raporlanır.

---

## 5. Yetkiler

| Rol | Cari | Toptan sipariş | Fatura | Fiyat/maliyet görür |
|---|---|---|---|---|
| Yönetici | tam | tam | tam | evet |
| Satış | ekle/düzenle | taslak + onay | görür, yükler | fiyat evet, maliyet hayır |
| Depocu | görmez (yalnız sevk adı) | yalnız onaylı siparişi sevk eder | görmez | hayır |
| Muhasebe | görür | görür | tam | evet |

Bugün MarketFlow'da rol kolonu var ama hiçbir yerde uygulanmıyor. Her üye her şeyi
görüp değiştirebiliyor. Depocu hesabı açılmadan önce rol denetimi veritabanı seviyesinde
yazılmalı (ayrıntı [01](01-marketflow-eksik-analizi.md), iş [06 Faz 0](06-fazlar-ve-kabul-kriterleri.md)).
