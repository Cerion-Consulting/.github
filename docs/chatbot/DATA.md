# Chatbot — Veri (Data) Dokümanı

> **Durum:** Taslak · **Sahibi:** Data sorumlusu · **Son güncelleme:** 2026-11-10
>
> Bu doküman, Cerion Consulting web sitesindeki chatbot'un hangi verileri kullandığını, bu verilerin nasıl toplandığını, işlendiğini, saklandığını ve korunduğunu tanımlar. "Önerilen" diye işaretlenen maddeler ekipçe onaylanana kadar karar değildir.

---

## İçindekiler

1. [Amaç ve kapsam](#1-amaç-ve-kapsam)
2. [Veri kaynakları](#2-veri-kaynakları)
3. [Bilgi tabanı (knowledge base) hazırlığı](#3-bilgi-tabanı-knowledge-base-hazırlığı)
4. [Konuşma verileri](#4-konuşma-verileri)
5. [Veri modeli (şema)](#5-veri-modeli-şema)
6. [Gizlilik, KVKK ve GDPR](#6-gizlilik-kvkk-ve-gdpr)
7. [Saklama ve silme](#7-saklama-ve-silme)
8. [Güvenlik ve erişim](#8-güvenlik-ve-erişim)
9. [Veri kalitesi ve güncelleme](#9-veri-kalitesi-ve-güncelleme)
10. [Ölçümleme ve raporlama](#10-ölçümleme-ve-raporlama)
11. [Roller ve sorumluluklar](#11-roller-ve-sorumluluklar)
12. [Açık sorular](#12-açık-sorular)

---

## 1. Amaç ve kapsam

**Chatbot'un amacı:** Web sitesi ziyaretçilerinin hizmetlerimiz, çalışma şeklimiz ve iletişim süreçleri hakkındaki sorularını yanıtlamak ve uygun ziyaretçileri iletişime / toplantı talebine yönlendirmek.

**Bu dokümanın kapsamı:**

- Chatbot'un yanıt üretirken dayandığı içerik (bilgi tabanı)
- Ziyaretçiyle yapılan konuşmalardan oluşan veriler
- Bu verilerin saklanması, korunması ve analiz edilmesi

**Kapsam dışı:** Arayüz tasarımı, model/sağlayıcı seçimi ve altyapı kurulumu (ilgili dokümanlarda ele alınır).

---

## 2. Veri kaynakları

| # | Kaynak | İçerik | Format | Sahibi | Güncelleme sıklığı |
|---|--------|--------|--------|--------|--------------------|
| 1 | Web sitesi sayfaları | Hizmetler, hakkımızda, iletişim | HTML / Markdown | Web ekibi | Sayfa değiştikçe |
| 2 | SSS (FAQ) listesi | Sık sorulan sorular ve onaylı cevaplar | Markdown / YAML | Data sorumlusu | Aylık gözden geçirme |
| 3 | Blog / vaka çalışmaları | Yayımlanmış içerikler | HTML / Markdown | İçerik ekibi | Yeni yazı yayımlandıkça |
| 4 | Sabit bilgiler | Çalışma saatleri, iletişim kanalları, ofis bilgisi | YAML / JSON | Data sorumlusu | Değiştikçe |
| 5 | Konuşma kayıtları | Ziyaretçi mesajları ve bot yanıtları | Veritabanı | Data sorumlusu | Sürekli |

**Kurallar:**

- Bilgi tabanına **yalnızca kamuya açık ve onaylanmış** içerik girer.
- Müşteri sözleşmeleri, teklifler, fiyat listeleri, iç yazışmalar ve müşteri kişisel verileri bilgi tabanına **girmez**.
- Her kaynağın bir sahibi olmalıdır; sahibi olmayan içerik eklenmez.

---

## 3. Bilgi tabanı (knowledge base) hazırlığı

Chatbot'un doğru ve güncel cevap vermesi için içerik aşağıdaki adımlardan geçer (önerilen akış — RAG yaklaşımı):

```
Kaynaklar ──► Toplama ──► Temizleme ──► Parçalama (chunking) ──► Embedding ──► Vektör veritabanı
                                                                                     │
Ziyaretçi sorusu ──────────────────────────────► En ilgili parçaları bul ◄───────────┘
                                                        │
                                                        ▼
                                              Model, bu parçalara dayanarak cevap verir
```

### 3.1 Toplama

- Web sayfaları sitemap üzerinden veya içerik deposundan (Markdown) alınır.
- SSS ve sabit bilgiler tek bir klasörde, sürüm kontrolü altında tutulur (önerilen: `content/chatbot/`).

### 3.2 Temizleme

- Menü, footer, cookie banner gibi tekrar eden kısımlar çıkarılır.
- Boş, yinelenen (duplicate) veya taslak sayfalar elenir.
- Metin UTF-8'e normalize edilir; Türkçe karakterler korunur.

### 3.3 Parçalama (chunking)

| Parametre | Önerilen değer | Not |
|-----------|----------------|-----|
| Parça boyutu | ~300–500 kelime | Başlık bazlı bölme tercih edilir |
| Örtüşme (overlap) | ~10–15% | Bağlam kopmasını azaltır |
| SSS | Her soru-cevap çifti tek parça | Bölünmez |

### 3.4 Meta veri

Her parçada en az şu alanlar bulunur:

- `source_url` veya `source_file` — cevabın kaynağını göstermek için
- `title` — sayfa / bölüm başlığı
- `language` — `tr` / `en`
- `last_updated` — içeriğin son güncellenme tarihi
- `category` — örn. `hizmetler`, `iletisim`, `sss`

### 3.5 Dil

- Site hangi dillerde yayındaysa bilgi tabanı da o dillerde tutulur.
- Ziyaretçi hangi dilde yazarsa, öncelikle o dildeki parçalar getirilir.

---

## 4. Konuşma verileri

### 4.1 Toplanan veriler

| Veri | Neden | Zorunlu mu? |
|------|-------|-------------|
| Oturum ID (anonim) | Konuşmayı bir arada tutmak | Evet |
| Mesaj metni (kullanıcı + bot) | Cevap kalitesini ölçmek ve iyileştirmek | Evet |
| Zaman damgası | Analiz, saklama süresi takibi | Evet |
| Sayfa URL'si (sohbetin başladığı sayfa) | Hangi sayfada soru sorulduğunu anlamak | Evet |
| Dil | Raporlama | Evet |
| Geri bildirim (👍 / 👎) | Kalite ölçümü | Hayır |
| İletişim bilgisi (ad, e-posta) | Yalnızca ziyaretçi **kendi isteğiyle** iletişim talep ederse | Hayır |

### 4.2 Toplanmayan veriler

- IP adresinin tamamı (gerekirse yalnızca kısaltılmış / anonimleştirilmiş hali)
- Tarayıcı parmak izi (fingerprinting)
- Ziyaretçinin onayı olmadan herhangi bir kişisel veri
- Konuşma içinden otomatik olarak çıkarılan kimlik bilgileri

### 4.3 Kişisel veri maskeleme

Ziyaretçiler mesajlara kendiliğinden kişisel veri yazabilir. Kayıt öncesinde şu desenler maskelenir (önerilen):

- E-posta adresleri → `[EMAIL]`
- Telefon numaraları → `[TELEFON]`
- TC kimlik no / IBAN / kart numarası → `[GIZLI]`

> Not: İletişim formu üzerinden **bilerek** paylaşılan iletişim bilgisi maskelenmez; ayrı bir tabloda, ayrı erişim yetkisiyle tutulur (bkz. `lead` tablosu).

---

## 5. Veri modeli (şema)

Önerilen tablolar (veritabanı türü henüz kesinleşmedi):

### `conversation`

| Alan | Tip | Açıklama |
|------|-----|----------|
| `id` | UUID | Birincil anahtar |
| `started_at` | timestamp | Konuşmanın başlangıcı |
| `ended_at` | timestamp, null | Son mesaj zamanı |
| `page_url` | text | Sohbetin başladığı sayfa |
| `language` | text | `tr` / `en` |
| `handoff` | boolean | İnsana / iletişim formuna yönlendirildi mi |

### `message`

| Alan | Tip | Açıklama |
|------|-----|----------|
| `id` | UUID | Birincil anahtar |
| `conversation_id` | UUID | `conversation.id` |
| `role` | enum | `user` / `assistant` |
| `content` | text | Maskelenmiş mesaj metni |
| `sources` | JSON, null | Cevapta kullanılan bilgi tabanı parçaları |
| `created_at` | timestamp | Mesaj zamanı |

### `feedback`

| Alan | Tip | Açıklama |
|------|-----|----------|
| `id` | UUID | Birincil anahtar |
| `message_id` | UUID | `message.id` |
| `rating` | enum | `up` / `down` |
| `comment` | text, null | İsteğe bağlı açıklama |
| `created_at` | timestamp | |

### `lead` (ayrı erişim yetkisi)

| Alan | Tip | Açıklama |
|------|-----|----------|
| `id` | UUID | Birincil anahtar |
| `conversation_id` | UUID | `conversation.id` |
| `name` | text | Ziyaretçinin verdiği ad |
| `email` | text | Ziyaretçinin verdiği e-posta |
| `consent_at` | timestamp | Onay (açık rıza) zamanı |
| `consent_text_version` | text | Gösterilen aydınlatma metninin sürümü |

---

## 6. Gizlilik, KVKK ve GDPR

- **Aydınlatma:** Chatbot penceresinde kısa bir bilgilendirme ve Gizlilik Politikası bağlantısı gösterilir ("Bu sohbet hizmet kalitesini iyileştirmek için kaydedilir").
- **Açık rıza:** İletişim bilgisi yalnızca ziyaretçi bir onay kutusunu işaretledikten sonra alınır.
- **Veri minimizasyonu:** Amaç için gerekmeyen hiçbir veri toplanmaz (bkz. §4.2).
- **Üçüncü taraf sağlayıcılar:** Kullanılan model / barındırma sağlayıcısının veri işleme sözleşmesi (DPA) incelenir; mesajların sağlayıcı tarafında model eğitimi için kullanılmadığı teyit edilir.
- **Yurt dışına aktarım:** Sağlayıcının sunucu konumu belirlenir ve KVKK/GDPR açısından gerekiyorsa Gizlilik Politikası'na eklenir.
- **İlgili kişi talepleri:** Erişim / silme talepleri en geç 30 gün içinde yanıtlanır; oturum ID'si veya e-posta ile kayıt bulunup silinebilmelidir.

---

## 7. Saklama ve silme

| Veri | Saklama süresi (önerilen) | Süre sonunda |
|------|---------------------------|--------------|
| Konuşma ve mesajlar | 12 ay | Otomatik silinir |
| Geri bildirim | 12 ay | Otomatik silinir |
| Lead (iletişim bilgisi) | İletişim süreci bitene kadar, en fazla 24 ay | Silinir veya CRM'e taşınır |
| Anonim toplu istatistikler | Süresiz | — (kişisel veri içermez) |
| Bilgi tabanı | Kaynak yayında olduğu sürece | Kaynak kaldırılınca indeksten de silinir |

- Silme işlemi zamanlanmış bir görevle (ör. günlük) yapılır ve loglanır.
- Yedeklerdeki veriler de aynı süre politikasına tabidir.

---

## 8. Güvenlik ve erişim

- API anahtarları ve veritabanı şifreleri **asla** depoya (repo) yazılmaz; ortam değişkenleri veya secret manager kullanılır.
- Veriler aktarımda (TLS) ve depolamada şifrelenir.
- Erişim rolleri:

| Rol | `conversation` / `message` | `feedback` | `lead` |
|-----|----------------------------|------------|--------|
| Data sorumlusu | Okuma / silme | Okuma | Okuma / silme |
| Geliştirici | Yalnızca test ortamı | Yalnızca test ortamı | Erişim yok |
| Satış / iletişim | Erişim yok | Erişim yok | Okuma |

- Test ve geliştirme ortamlarında **gerçek konuşma verisi kullanılmaz**; sentetik örnek veri kullanılır.

---

## 9. Veri kalitesi ve güncelleme

- **Tek doğru kaynak:** Bir bilgi (ör. iletişim e-postası) yalnızca tek bir yerde tutulur.
- **Yeniden indeksleme:** Kaynak içerik değiştiğinde bilgi tabanı yeniden oluşturulur (önerilen: içerik deposuna yapılan her merge sonrası otomatik).
- **Aylık gözden geçirme:**
  - 👎 alan cevaplar incelenir
  - Cevaplanamayan sorular listelenir → SSS'ye eklenir veya içerik güncellenir
  - Güncelliğini yitirmiş içerik (`last_updated` > 6 ay) kontrol edilir
- **Test seti:** En az 30–50 soru ve beklenen cevaptan oluşan bir değerlendirme seti tutulur; bilgi tabanı veya ayarlar her değiştiğinde bu set ile kontrol yapılır.

---

## 10. Ölçümleme ve raporlama

| Metrik | Tanım | Hedef (önerilen) |
|--------|-------|------------------|
| Konuşma sayısı | Haftalık / aylık başlatılan konuşma | Takip |
| Cevaplanma oranı | Bot'un bilgi tabanından cevap verebildiği soru oranı | ≥ %85 |
| Olumlu geri bildirim oranı | 👍 / (👍 + 👎) | ≥ %80 |
| Yönlendirme (handoff) oranı | İletişim formuna / insana aktarılan konuşma oranı | Takip |
| Lead dönüşümü | İletişim bilgisi bırakılan konuşma oranı | Takip |
| En çok sorulan konular | Kategori bazında soru dağılımı | Aylık rapor |

Raporlar yalnızca **anonim ve toplu** verilerle hazırlanır.

---

## 11. Roller ve sorumluluklar

| Görev | Sorumlu |
|-------|---------|
| Bu dokümanın bakımı | Data sorumlusu |
| Bilgi tabanı içeriğinin doğruluğu | İlgili içerik sahibi (bkz. §2) |
| SSS listesinin güncellenmesi | Data sorumlusu |
| Gizlilik metni ve KVKK uyumu | Data sorumlusu + hukuk danışmanı |
| Altyapı, şifreleme ve erişim ayarları | Geliştirici |
| Aylık kalite raporu | Data sorumlusu |

---

## 12. Açık sorular

- [ ] Hangi model / sağlayıcı kullanılacak? Sunucu konumu ve DPA durumu nedir?
- [ ] Hangi veritabanı ve vektör veritabanı kullanılacak?
- [ ] Site hangi dillerde yayında olacak (yalnızca TR mi, TR + EN mi)?
- [ ] Saklama süreleri (§7) onaylanıyor mu?
- [ ] Lead verileri bir CRM'e aktarılacak mı? Aktarılacaksa hangisine?
- [ ] Chatbot fiyat / teklif sorularına nasıl cevap verecek (yönlendirme mi, genel bilgi mi)?
- [ ] Gizlilik Politikası'nın chatbot bölümünü kim yazacak / onaylayacak?
