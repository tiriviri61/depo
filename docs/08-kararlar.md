# 08 — Karar listesi

> Uygulamaya başlamadan önce sizin vermeniz gereken kararlar. Her birinde önerilen seçenek
> **kalın** yazılı ve plan şu an o seçenekle kurgulandı. Karar verildikçe "Durum" sütunu
> güncellenir ve MarketFlow `docs/VERI-KURALLARI.md`'ye işlenir.

## A. İş kuralları (planın şeklini değiştirir)

| # | Soru | Seçenekler | Neden önemli | Durum |
|---|---|---|---|---|
| A1 | Ortak stok **ne zaman** düşsün? | **Sipariş gelince satılabilir düşsün (ayrılmış), mal çıkınca elde düşsün** · Yalnız kargoya verilince düşsün | Sipariş anında düşmezse aynı ürün başka kanalda tekrar satılır | Bekliyor |
| A2 | Pazaryeri iadesi stoka **ne zaman** dönsün? | **Depoda okutulunca (sağlam/hasarlı seçilerek)** · Pazaryeri "iade" deyince otomatik | Pazaryeri iadeyi mal dönmeden yazıyor; kısmi iadede bütün satırı yazıyor | Bekliyor |
| A3 | Kanallara stok yayını | **Otomatik; kanal bazlı kapatılabilir; 7 gün gölge moddan sonra** · Her yayın elle onaylı | Bugünkü kural "onaylı yayın" diyor; ortak stok otomatik olmadan çalışmaz | Bekliyor |
| A4 | Kanal emniyet payı ve az stok modu | **Her kanala satılabilir − 2; 5 adetin altında yalnız seçilen ana kanal(lar)** · Pay yok | 5 dakikalık gecikmede aynı ürünün iki kanalda satılmasını azaltır | Bekliyor |
| A5 | Amazon siparişlerini kim gönderiyor? | **Biz (FBM) — ortak stoktan düşer** · Amazon deposu (FBA) — ayrı stok | FBA ise o ürünler bizim depoda değildir, ortak stoğa katılmaz | Bekliyor |
| A6 | Kargo etiketi basılan paket stoktan "çıktı" sayılsın mı? | Evet (daha erken, daha doğru elde) · **Hayır, pazaryeri "kargoda" deyince (ilk sürüm)** | Erken çıkış için paket ↔ sipariş satırı bağı kurulmalı | Bekliyor |
| A7 | Alış maliyeti | **Elle girilen alış maliyeti kalsın (bugünkü gibi)** · Mal kabulde ağırlıklı ortalama güncellensin | Kâr hesabı buna bağlı; değişirse eski aylar etkilenmemeli | Bekliyor |

## B. Depo fiziği

| # | Soru | Seçenekler | Durum |
|---|---|---|---|
| B1 | Koli barkodu | **Her fiziksel koliye tekil numara (LPN) + ürün başına koli tipi barkodu** · Yalnız koli tipi barkodu | Bekliyor |
| B2 | Raf/lokasyon takibi | **İlk sürümde yok, tek depo (Faz 7'de eklenebilir)** · Baştan olsun | Bekliyor |
| B3 | Lot / son kullanma / seri no | **Yok** · Bazı ürünlerde (ör. batarya) olsun | Bekliyor |
| B4 | Birden fazla depo var mı / olacak mı? | **Şimdilik tek; model çoklu depoya hazır** · Evet, şu depolar: … | Bekliyor |
| B5 | El terminali marka/model | Bilgi gerekiyor (ör. Zebra TC21, Honeywell EDA52, Urovo DT50) | Bekliyor |
| B6 | Etiket yazıcısı marka/model ve etiket ölçüleri | Bilgi gerekiyor (ZPL destekliyor mu? 100×100 mm var mı?) | Bekliyor |
| B7 | Açılış sayımı ne zaman yapılabilir? | Tarih gerekiyor (devreye alma anı buna bağlı) | Bekliyor |

## C. Toptan ve fatura

| # | Soru | Seçenekler | Durum |
|---|---|---|---|
| C1 | Toptan faturası nasıl kesilecek? | **Fatura panelinde kesilip PDF/XML siparişe yüklensin (ilk sürüm)** · MarketFlow entegratör API'siyle kessin | Bekliyor |
| C2 | e-İrsaliye yükümlülüğünüz var mı? | Var → irsaliye entegratörden, siparişe eklenir · **Yok → sevk belgesi (A4) yeterli** | Bekliyor |
| C3 | Cari ekstre / tahsilat takibi | **Hafif sürüm (Faz 6b): borç/alacak, bakiye, vadesi geçen** · Yok, muhasebe programında | Bekliyor |
| C4 | Toptan fiyatı | **Siparişte elle + cari iskonto oranı** · Cari bazlı fiyat listeleri | Bekliyor |
| C5 | Notion'daki "Müşteriler & Cari Hesaplar" kayıtları aktarılsın mı? | **Evet, tek seferlik** · Hayır, sıfırdan | Bekliyor |

## D. Kapsam ve kanallar

| # | Soru | Seçenekler | Durum |
|---|---|---|---|
| D1 | ZYGA vitrin ürünleri MarketFlow kataloğuna barkodlu ürün olarak açılsın mı? | **Evet (ortak stok için şart)** · Vitrin ayrı stokta kalsın | Bekliyor |
| D2 | n11, Pazarama vb. | **Kapsam dışı (entegrasyon yok)** · Şu kanal da eklensin: … | Bekliyor |
| D3 | PTT ve Teknosa ilanları e-fatura bağlanana kadar kapalı (mevcut karar) | **Karar aynen; stok yayını kanal kapalıyken de hazır bekler** · Değişsin | Bekliyor |

## E. Ekip ve erişim

| # | Soru | Seçenekler | Durum |
|---|---|---|---|
| E1 | Kaç depocu / satış / muhasebe kullanıcısı olacak? | Sayı ve roller gerekiyor | Bekliyor |
| E2 | Depocu maliyet ve kârı görsün mü? | **Hayır** · Evet | Bekliyor |

## F. Kod ve depo

| # | Soru | Seçenekler | Durum |
|---|---|---|---|
| F1 | Kod nerede yazılsın? | **MarketFlow deposunda (web + mobil + veritabanı birlikte; dağıtım hattı hazır). Bu depo plan ve karar kaydı** · Bu depoda ayrı uygulama | Bekliyor |
| F2 | Bu depo (`depo`) **herkese açık**. Gizli yapılsın mı? | **Evet, gizli yapılsın** (iş süreci ve mimari ayrıntısı içeriyor) · Açık kalsın | Bekliyor |
