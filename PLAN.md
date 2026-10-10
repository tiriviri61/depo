# Depo Sistemi — Ana Plan

> MarketFlow'a entegre, ortak stoklu, barkod ve el terminaliyle çalışan depo yönetim sistemi.
> Durum: **Kararlar onaylandı (30 Eyl 2026). Faz 0 ve Faz 1 veritabanında canlı (2–3 Eki; web ekranları
> PR birleşince). Faz 2 sipariş motoru canlıda, gölge modda (3 Eki): siparişler ortak stoğu
> düşürüyor, kanallara yazılmıyor. Faz 2b: ZYGA vitrini aynı ortak stoktan satıyor (3 Eki).
> Faz 3 kanal yayını canlıda, gölge modda (4 Eki): kanala ne gideceği hesaplanıyor, hiçbir şey
> yazılmıyor; kanal kanal canlıya geçiş en erken 10 Eki. Faz 4 (barkod, koli, etiket, mal kabul)
> veritabanında canlı (4 Eki); gerçek terminal ve yazıcı denemesi bekleniyor. Faz 5 el terminali
> (sevk, sayım, iade okutma, yerel kuyruk) veritabanında canlı (4 Eki); gerçek terminal ve Wi-Fi
> denemesi bekleniyor. Faz 6 (cari, toptan sipariş, fatura dosyası, hafif cari ekstre) veritabanında
> canlı (5 Eki); gerçek veriyle ilk deneme ve Notion aktarımı kullanıcıda. Faz 7a depo paneli ve stok
> değeri/yaşlanma raporu canlı (11 Eki); lokasyon/lot kararı bekliyor.**
> Açık bilgi maddeleri: [docs/08](docs/08-kararlar.md). Raporlar MarketFlow deposunda
> (`docs/depo/FAZ0.md`, `FAZ1.md`, `FAZ2.md`, `FAZ2B.md`, `FAZ3.md`, `FAZ4.md`, `FAZ5.md`, `FAZ6.md`, `FAZ7.md`).
> Tarih: 30 Eylül 2026.

---

## 1. Ne istiyoruz

1. **Tek ürün listesi, tek stok.** MarketFlow'daki ürün listesi ortak stok listesidir.
   Bütün pazaryerlerinden (Trendyol, Hepsiburada, idefix, Amazon, PTT AVM, Teknosa) ve web
   sitesinden (ZYGA) gelen siparişler bu ortak stoktan düşer.
   > Örnek: 5000 RGB lamba geldi. idefix 100, Amazon 100, Trendyol 200, PTT AVM 100 sattı →
   > ortak stok **4500**. Bütün kanallar 4500 gösterir.
2. **Barkod her yerde aynı.** Ürün barkodu MarketFlow'daki barkoddur; depo kendi ürününü
   tutmaz, barkod değiştirmez.
3. **Ürün ve koli barkodları.** Her koli etiketlenir; el terminaliyle koli okutarak
   toptan satış için mal çıkışı yapılır.
4. **Müşteri (cari) kayıtları ve toptan siparişler.** Siparişin içinde satırlar, sevk
   bilgisi ve **faturası** saklanır.
5. **Gerçek bir depo sistemi.** Mal kabul, sayım, iade kabul, hareket defteri, yetkiler,
   denetim izi; operasyonu en üst seviyede yönetecek şekilde.
6. **Her yüzeyde.** Web paneli (saas.sonarteknoloji.com), mobil uygulama, Chrome uzantısı
   ve el terminali. Dağıtım mevcut sunucu ve GitHub Actions hattıyla.

## 2. Bugün MarketFlow'da durum (kısaca)

MarketFlow'da tek ana stok alanı ve "barkod tek anahtar" kuralı var. Ama:
**sipariş, iptal ya da iade gelince stoğu değiştiren hiçbir kod yok**, ana stok yalnız
elle giriliyor (Temmuz ölçümünde çoğu üründe hiç girilmemişti), kanallara yayın elle ve
yalnız iki kanala. Depo, koli, mal kabul, cari ve
toptan sipariş hiç yok. Web sitesinin (ZYGA) kendi ayrı stoğu var; ortak stok kararı
onun tarafında da "bekliyor" diye not düşülmüş. Tam döküm:
**[docs/01 — MarketFlow eksik analizi](docs/01-marketflow-eksik-analizi.md)**.

## 3. Mimari

```
 Trendyol  Hepsiburada  idefix  Amazon  PTT AVM  Teknosa           ZYGA vitrin
    │          │          │       │        │        │                    │
    └──────── sipariş senkronları (5 dk'da bir) ─────┘                    │ rezervasyon (anlık)
                         │                                               │
                  public.orders ─ değişim tetikleyicisi ─► depo.siparis_kuyrugu
                                                   │
                                   mutabakat işçisi (hedef durum, farkı yazar)
                                                   ▼
 mal kabul · sevk · sayım · iade ──► depo.hareket_yaz() ──► depo.hareketler (defter, salt ekleme)
 (web · el terminali · mobil)                   │
                                               ▼
                                         depo.bakiye ── satılabilir ──► zyga.stock (görünüm)
                                               │
                                        depo.yayin_kuyrugu
                                               ▼
                           kanal bağdaştırıcıları (gönder + geri okuyarak doğrula)
                                               ▼
                 Trendyol  Hepsiburada  idefix  Amazon  PTT AVM  Teknosa
```

Temel kararlar:

- **Stok bir defterdir.** Her değişiklik eklenen bir harekettir; bakiye defterin toplamıdır.
  Her hareketin tekil anahtarı var, çift sayım yapısal olarak imkânsız.
- **Siparişten düşüm "hedef durum" ile.** MarketFlow senkronları aynı satırları her 5 dakikada
  yeniden yazıyor ve durumlar ileri-geri gidiyor. Motor her satırın **olması gereken**
  etkisini hesaplayıp yalnız farkı yazar. Olay sayan basit bir tetikleyici bu ortamda
  kesin hata üretir ([docs/02 §4](docs/02-veri-modeli-ve-stok-motoru.md)).
- **Üç sayı:** elde, ayrılmış, **satılabilir (ortak stok)**. Sipariş gelince satılabilir
  düşer, mal çıkınca elde düşer.
- **İade, depoda okutulunca stoka döner.**
- **Kanallara otomatik yayın**, birleştirmeli, doğrulamalı, kanal bazlı kapatılabilir.
- **Gölge mod:** canlıya almadan önce en az 7 gün kanallara yazmadan paralel çalışır.
- **Kod MarketFlow deposunda** (web + mobil + veritabanı), ayrı `depo` şemasında.
  ZYGA vitrinin stoğu ortak stoğun görünümüne döner.

## 4. Belgeler

| Dosya | İçerik |
|---|---|
| [01 — MarketFlow eksik analizi](docs/01-marketflow-eksik-analizi.md) | Kod ve tasarım eksikleri, kanıt, önem, faz |
| [02 — Veri modeli ve stok motoru](docs/02-veri-modeli-ve-stok-motoru.md) | Değişmezler, tablolar, hedef durum motoru, kanal yayını, ZYGA, bütünlük, devreye alma |
| [03 — Barkod, koli, el terminali](docs/03-barkod-koli-el-terminali.md) | Barkod türleri, koli (LPN), etiketler, mal kabul, toptan sevk, sayım, iade, terminal |
| [04 — Cari, toptan, fatura](docs/04-cari-toptan-fatura.md) | Müşteri kaydı, toptan sipariş, durum makinesi, fatura saklama, roller |
| [05 — Ekranlar ve tasarım](docs/05-ekranlar-ve-tasarim.md) | Menü yapısı, web ekranları, terminal, mobil, uzantı, tasarım süreci |
| [06 — Fazlar ve kabul ölçütleri](docs/06-fazlar-ve-kabul-kriterleri.md) | Faz 0-7 iş listesi ve ölçülebilir kabul ölçütleri |
| [07 — Skill'ler ve istemler](docs/07-skill-ve-promptlar.md) | Skill araması, kurulacak skill'ler, proje skill'i, faz istemleri |
| [08 — Karar listesi](docs/08-kararlar.md) | Sizin vermeniz gereken kararlar ve önerilenler |

## 5. Fazlar (özet)

| Faz | Ad | Sonunda ne olur |
|---|---|---|
| 0 | Zemin | Barkod tekil, sipariş adetleri/durumları doğru, PTT göçü, roller ve davet, kurallar yazılı |
| 1 | Stok çekirdeği | Defter + bakiye + açılış sayımı + Ortak Stok ekranı |
| 2 | Siparişten düşüm | Bütün kanalların siparişleri ortak stoğu doğru düşürür; ZYGA bağlı; gölge mod |
| 3 | Kanal yayını | Satılabilir sayı 6 kanala otomatik ve doğrulanarak gider; "tavan 100" kalkar |
| 4 | Barkod, koli, mal kabul | Koli (LPN), etiketler, mal kabul ekranı |
| 5 | El terminali | `/terminal`: kabul, sevk, sayım, iade, sorgu; mobil depo ekranları |
| 6 | Cari, toptan, fatura | Cariler, toptan sipariş, terminalle sevk, fatura saklama |
| 7 | Raporlar ve genişleme | Depo paneli, sipariş önerisi, lokasyon, uzantı rozeti, kamera okutma, fatura kesme |

Faz 1 ile Faz 4'ün ekran tasarımı paralel yürüyebilir. Faz 5 ve 6 birbirine bağlı
(toptan sevk terminalle yapılır), birlikte planlanır.

## 6. Hemen başlamadan önce

1. **Kararlar:** [docs/08](docs/08-kararlar.md) — özellikle A1-A3, A5, B1, B5-B7, C1, F1-F2.
2. **Yakın risk:** PTT AVM'nin yeni API anahtarı yapısı **Ekim sonunda** devreye giriyor;
   kullanıcı adı/şifreli erişim bir süre daha çalışacak, kapanış tarihi duyurulacak.
   Kapandığı gün PTT siparişleri MarketFlow'a gelmez, stok da düşmez. Geçiş Faz 0'da
   hazırlandı; PTT'nin teknik belgesi ve entegratör anahtarı bekleniyor.
3. **Açılış sayımı için gün:** sistem ancak sayılmış stokla canlıya alınabilir.
4. **Donanım bilgisi:** el terminali ve etiket yazıcısı modeli.
