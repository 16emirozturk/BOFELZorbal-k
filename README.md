# Güvenli Okul — Bildirim ve Takip Platformu (v2)

Okullarda akran zorbalığı bildirimlerini toplamak, **insan incelemesi** için önceliklendirmek ve süreci
izlenebilir kılmak için hazırlanmış bir Streamlit prototipidir. Karar veren bir sistem **değildir**.

## Çalıştırma
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .streamlit/secrets.toml.example .streamlit/secrets.toml
python tools/parola_hash.py                            # çıkan password_hash'i secrets.toml'a yazın
streamlit run app.py
```
Testler (ek bağımlılık gerekmez): `python -m unittest discover -s testler -v`

## Yapı
| Dosya | Görev |
|---|---|
| `app.py` | Arayüz ve sayfa akışı |
| `db.py` | SQLite katmanı: şema göçü, işlemler, istatistik, CSV, saklama süresi |
| `guvenlik.py` | Parola özeti/doğrulama, kişisel veri maskeleme, CSV enjeksiyon koruması |
| `hakaret_tespiti.py` | Kural tabanlı sinyal: kategori, öncelik, bağlam uyarısı |
| `degerlendirme.py` | Sentetik etiketli küme + precision/recall/F1 |
| `stil.py` | CSS (harici yazı tipi/CDN yok) |
| `ek_sayfalar.py` | Destek rehberi, trend/ortam analizi, gizlilik ve KVKK paneli (onam akışı simülasyonu) |
| `testler/` | Birim testleri |

## v1 → v2: düzeltilen hatalar
- Türkçe karakterli kullanıcı adı/parola `hmac.compare_digest` ile `TypeError` veriyordu → UTF-8 bayt karşılaştırması.
- Sayfa düğmeleri `session_state["ana_menu"]` değerini widget oluştuktan sonra değiştiriyordu → `on_click` geri çağrıları.
- `st.success(); st.rerun()` mesajı yutuyordu → `flash` mekanizması.
- `clear_on_submit=True` doğrulama hatasında da yazılan metni siliyordu → yalnızca başarıda form sıfırlanır.
- `with sqlite3.connect()` bağlantıyı kapatmıyordu → her işlemde kapatan bağlam yöneticisi.
- Saatler sunucu saatindeydi (bulutta UTC) → Europe/Istanbul.
- `ilk_acilista_kayitlari_temizle` bayrak yoksa TÜM kayıtları siliyordu → kaldırıldı; yerine şema sürümü + göç.
- Algılayıcı "gerizekalı" yazımını kaçırıyordu (sözlükte ASCII) → Türkçe'ye duyarlı katlama.
- Bozuk Markdown (`** Gizlilik**`), boş ikon kutuları, çift "10." bölüm numarası, harici Google Fonts çağrısı.

## Yeni özellikler
Öncelik sinyali ve kuyruk sıralaması · kişisel veri maskeleme (T.C. no sağlama toplamı dâhil) · takip kodu ile
durum sorgulama · hatalı giriş kilidi + oturum zaman aşımı + PBKDF2 parola özeti · denetim kaydı ·
yönetici işlemlerinde "kim yaptı" bilgisi · CSV dışa aktarım (kimlik varsayılan hariç) · saklama süresi temizliği ·
anonim kayıtlarda zaman yuvarlama · yanıt süresi metrikleri · otomatik algılayıcı değerlendirmesi.

## Bilimsel doğrulama yol haritası (TÜBİTAK için)
1. Araştırma sorusu ve hipotezlerin yazılması (ör. "Kural tabanlı sinyal, yönetici inceleme sırasını iyileştirir mi?").
2. Etik kurul / okul ve veli izniyle, kimliksizleştirilmiş gerçek veya uzmanca üretilmiş bir veri kümesi.
3. En az iki uzmanın (rehber öğretmen/psikolojik danışman) bağımsız etiketlemesi; Cohen kappa raporu.
4. Eğitim/doğrulama/test ayrımı; kural tabanlı yöntemin başka bir yöntemle (ör. sınıflandırıcı) karşılaştırması.
5. Kullanılabilirlik testi (SUS) ve yanıt süresi ölçümü.
6. Yanlış alarm/kaçırma analizi ve sınırlılıkların açıkça raporlanması.

## Yayına almadan önce
KVKK (aydınlatma, hukuki sebep, saklama-imha, yurt dışı sunucu), kişi bazlı hesap + 2FA, HTTPS, şifreli yedek,
bağımsız güvenlik testi ve okul acil durum protokolü gereklidir. Acil hat numaralarını (`sabitler.py`) resmi
kaynaklardan yeniden doğrulayın.

## v2.1 — Tam anonim bildirim ve sadeleştirilmiş akış
Artık tek bir **Yeni bildirim** ve tek bir **Bildirim sorgula** sayfası var (eski yinelenen sayfalar kaldırıldı).

- **Tam anonim:** formda ad, soyad, sınıf, telefon istenmez ve kaydedilmez; zaman dakika/saniye hassasiyetinde saklanır.
- **Günlük kota:** `app.py` içindeki küçük bir yerel bileşen (çalışma anında geçici klasöre yazılır, ayrı dosya gerekmez); rastgele cihaz jetonunu tarayıcının localStorage'ında
  (yedek: çerez) saklayıp uygulamaya iletir, sayfa yenilense de aynı kalır. Veritabanında jetonun SHA-256 özeti ve gün (`anonim_cihaz`) saklanır; rapora
  bağlanmaz, gün bitince silinir. Aynı gün ikinci bildirimde “Günlük bildirim hakkınız dolmuştur.” görünür.
- **Gizli takip kodu:** `GO-XXXXXXXXXX` (~50 bit). Sorgu sayfası durumu, kısa bir açıklamasını ve tarihleri gösterir.
- **Ek bilgi:** aynı sorgu sayfasında; yalnızca anonim kayıtlarda, bildirimin günü; 60 sn bekleme ve günde en çok 3 ek
  **sunucuda** uygulanır. Ek metin maskelenir, analiz edilir; daha yüksek sinyal çıkarsa kayıt önceliği yükselir.
  İnceleme ekranında “Anonim gönderenin ek bilgileri” ve kuyrukta “+N ek bilgi” görünür.
- **Mesajlaşma yok:** öğrenci yalnızca durumu görür; yönetici notları gösterilmez.
- **Telefon uyumu:** seçenekler alt alta dokunması kolay kartlar, büyük düğmeler, iOS'ta yakınlaştırmayı önleyen yazı
  boyutu, dar ekranda tek sütun.

Bilinen sınırlar: kota tarayıcı çerezine dayanır; çerez silme/gizli sekme ile aşılabilir (hesapsız anonimlikte kesin
kota mümkün değildir). Streamlit Cloud'da SQLite dosyası kalıcı değildir; gerçek kullanımda harici veritabanı gerekir.

## v2.3 — Modern arayüz ve ek sayfalar
- Gradyanlı karşılama alanı ve satır içi SVG illüstrasyon/ikon rozetleri (harici resim, CDN, yazı tipi yok).
- Logo kenar çubuğuna gömülü (base64) olarak yerleştirilir; ek istek yapılmaz.
- Acil hatlar: 112, 183, 444 0 632 (MEBİM). Yayından önce resmî kaynaklardan yeniden doğrulayın.
- Yeni sayfalar: **Destek rehberi** (herkese açık), **Trend ve ortam** (yönetici; küçük hücre bastırmalı),
  **Gizlilik ve KVKK** (aydınlatma taslağı, veri envanteri, saklama/imha, onam akışı simülasyonu).
- Güvenlik düzeltmeleri: kullanıcı metni artık Markdown olarak yorumlanmaz (görsel/bağlantı enjeksiyonu önlenir);
  ek bilgisi olan sonuçlanmış kayıtlar saklama süresi sonunda hatasız silinir; günlük kota denetimi salt okunurdur.


## Ek geliştirmeler
- Arayüzde Türkçe / English dil seçimi (arayüz metinleri için çeviri katmanı).
- Yönetici genel bakış ve inceleme ekranında günlük özet ve yüksek/acil uyarı bandı.
- İnceleme kuyruğunda tarih aralığı filtresi ve bekleme süresi / 24 saat gecikme göstergesi.
- Kategoriye göre rehberlik kontrol listesi.
- Yeni anonim bildirimler ve ek bilgiler için Europe/Istanbul saatinde saniye hassasiyetli zaman damgası.

Not: Eski veritabanında daha önce saat başına yuvarlanmış zaman damgalarının kaybolan dakika/saniye kısmı geriye dönük üretilemez. Yeni kayıtlar doğru hassasiyetle saklanır.


### Uygulama notları
- Dil seçimi oturum boyunca korunur. Menü, ana bileşen başlıkları ve seçenek etiketleri için TR/EN katmanı eklendi; okul rehberi ve uzun bilgilendirme metinlerinin tamamı yayımdan önce iki dilli editoryal kontrolden geçirilmelidir.
- SLA süresi her sayfa yenilemesinde/etkileşimde yeniden hesaplanır. Canlı saniyelik otomatik yenileme kullanılmaz; gereksiz arka plan yenilemesi ve veritabanı yükü oluşturulmaz.
- Rehberlik önerileri karar veya klinik yönlendirme değildir; okul psikolojik danışmanı tarafından onaylanmalıdır.
