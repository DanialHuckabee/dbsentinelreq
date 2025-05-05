
# DB-Sentinel Gereksinim ve Kapsam Dokümanı

## Genel Açıklama
DB-Sentinel, üretim (KVT) ve test (HVT) veritabanları arasındaki hassas veri sızıntılarını tespit eden, sürekli denetim sağlayan ve otomatik karşılaştırma yapan bir yazılımdır. SQL, DB2 ve Oracle veritabanlarını destekler.

---

## Temel İşlevler

### 1. Kaynak Veritabanı Analizi (KVT)
- Tüm tablo ve kolonlar analiz edilir.
- Kritik veriler içeren kolonlar (örn. TC No, Ad, Anne Kızlık Soyadı) otomatik olarak belirlenir.
- Otomatik olarak seçilen kolonlar için 2 farklı yöntem kullanılır. Birinci yöntemde kolon adı ve kolonda yazan değere göre verinin kritik olup olmadığı belirlenir. İkinci yöntemde, kolonlar için kardinalite değeri hesaplanır. Her iki yöntem de kolonlardaki verinin kritikliğiyle alakalı bir değer çıkartır. Bu değer 0 ile 1 arasında bir değerdir. Bu değere kritiklik puanı (KP) denir.
- Yapılan analiz sonununda kritik kolonlar kullanıcıya tablo > kolon adı şeklinde ağaç yapısında gösterilir. 
- Kullanıcılar analiz sonuçlarını manuel düzenleyebilir, kolon ekleyip çıkarabilir, veri türlerini değiştirebilir. Ağaç yapısında gösterim sırasında kritik olmayan kolonlar da gösterilerek kullanıcının dilediği kolonu eklemesi, kritiklik puanınu değiştirmesi ve mevcut bir kolonu çıkartması mümkün olacaktır. Yapılan tüm işlemler kullanıcı oturumu ile ilişkili şekilde audit kaydı olarak tutulacaktır.
- Analiz sonuçları sürümlenerek saklanır ve sonraki analizlerde yeniden kullanılır. Analiz sonuçlarına "Veri tabanı Analiz Planı" adı verilir. Bu analiz planları üretiliş tarihleri, kim tarafından yapıldığı, kullanıcı notları gibi ek bilgilerle saklanır.
- Kolonlar kullanıcı tanımlı veri türleri eklenerek genişletilebilir.
- KVT analizi belirli aralıklarla otomatik olarak tekrar edilebilir. Bu işlemin amacı şemanın değişmesi durumunda ilgili kişileri uyarmaktır. Bu nedenle Veri tabanı Analiz Planı (VTAP) tüm veri tabanı şemasını da kendi içinde barındırır. Yöneticilerin belirlediği aralıklarda şema kontrolü yapılarak şema değişikliği durumunda ilgili kişilere e-posta gönderen mekanizma bulunmalıdır.

### 2. Örnekleme Yöntemi
- Kaynak veritabanın tümü değil de, veri tabanının büyüklüğü ve kolonların KP değerine göre örnekleme oranı hesaplanır.
- Hassasiyete göre örnekleme oranlarına örnek:
  - TC No: %10
  - Ad: %5
- Kullanıcılar, veritabanı bazında bu oranları değiştirebilir.
- Tablo büyüklüğü (kayıt sayısı) önemli:
  - 1000 veya daha az kayıt olan tablolar tamamen örneklenir.
  - Büyük tablolarda belirtilen oranlara göre örnekleme yapılır.
- Tüm tabloların kayıt sayıları analiz sırasında tespit edilip saklanmalıdır. Örnek olarak tabloda 10.000 kayıt vardı, 1.000 adet örnekleme yapıldı.
- Örnekleme işleminde ardarda kaytları almak yerine, 1,25,50,75 gibi kayıt sayısı bölü örnekleme oranı şeklinde hesaplama yapılıp o sıradaki kayıtlar örneklenmeli.

### 3. Veri İşleme ve Hashleme
- Verilerin tamamı değil, sadece belirlenen örnekleme oranına göre seçilen kayıtların hash'i alınır.
- Hashleme algoritması: BLAKE2. Hash algoritmaları sonradan eklenebilir olmalıdır. 
- Hash değerleri, sistemin kendi VT'sine yazılır.
- Bu işlemin tümün Kaynak VT İşleme adı verilir. KVTİ. Bu işlem sonunda sistemde KVTİ bilgisi oluşur. Bu işlem yöneticinin belirlediği aralıklarla otomatik veya yetkili kullanıcı tarafından manuel olarak çalıştırılabilir. Otomatik olarak çalışmama durumunda ilgili kullanıcılara uyarı maili gider. Her KVTİ bir VTAP'yle ilişkilendirilir.
- Sonraki analizlerde sadece yeni eklenen verilerin hash'i alınır (incremental analiz).

### 4. Hedef Veritabanı Analizi (HVT)
- KVT analiziyle aynı yöntem ve süreç uygulanır.
- HVT ve KVT arasında kolon bazlı karşılaştırma yapılır (tablo şemaları aynı olmak zorunda değil).

### 5. Veri Karşılaştırma
- Karşılaştırma otomatik (schedule) veya manuel tetiklenebilir.
- Hash veri setleri karşılaştırılır, eşleşmeler BLOOM filtresi ile tespit edilir.
- Her kolonun veri türüne göre farklı ağırlıklandırma yapılır (örn. TC No: 1.0, Ad: 0.6).
- Aynı tabloda belirli sayıda eşleşen kolon bulunursa şüpheli kabul edilir. Eşik değeri kullanıcı tarafından belirlenebilir.

### 6. Risk ve Şüpheli Durum Sınıflandırması
- Şüpheli satır sayısı ve oranına göre risk sınıflandırılır:
  - Kırmızı (yüksek risk): %10+
  - Turuncu (orta risk): %5-10
  - Sarı (düşük risk): %1-5
  - Yeşil (minimal risk): <%1
- Şüpheli durumlar tablolar halinde raporlanır.
- Risk seviyeleri için otomatik bildirimler tetiklenir (özellikle yüksek risk).

### 7. Performans ve Teknolojik Yapı
- Paralel multi-task çalışma prensibi.
- Adaptive Sampling (önceki analiz sonuçlarına göre örnekleme oranlarını otomatik olarak ayarlama).
- Incremental analiz (sadece yeni verilerin analizi).
- Paralel hashing ve karşılaştırma işlemleri.
- Makine öğrenmesi ile proaktif anomali tespiti (opsiyonel).

---

## Kullanıcı Arayüzü (UI) Gereksinimleri

### 1. Dashboard
- **Ana Risk Paneli**: Kırmızı/turuncu/sarı/yeşil göstergelerle görsel risk özeti
- **İnteraktif Grafikler**: Zaman serisi grafikleri ile risk trendlerini gösterme
- **İstatistik Kartları**: Şüpheli tablo/kolon sayısı, son analiz tarihleri
- **Durum Paneli**: Aktif çalışan ve bekleyen analizlerin canlı durum göstergeleri
- **Özet Metrikler**: KVT/HVT karşılaştırma sonuçlarının kümülatif gösterimi

### 2. Veritabanı Yönetimi
- **Hiyerarşik Görünüm**: Ağaç yapısında veritabanı/şema/tablo/kolon hiyerarşisi
- **Sürükle-Bırak Fonksiyonalitesi**: Hassasiyet ayarları için kolay kullanım
- **Veri Tipi Gösterimi**: Kolonların veri tiplerini görsel olarak ayırt edilebilir şekilde gösterme
- **Tablo Büyüklük Göstergeleri**: Kayıt sayısı ve depolama boyutu bilgileri 
- **Filtreleme Araçları**: Hızlı erişim için çeşitli filtreleme seçenekleri

### 3. Analiz Merkezi
- **İlerleme Göstergeleri**: Paralel çalışan analizlerin gerçek zamanlı ilerleme çubukları
- **Takvim Görünümü**: Zamanlanmış analizlerin görsel takvim üzerinde gösterimi
- **Sonuç Özeti**: Tamamlanan analizlerin özet sonuçları ve geçmiş trendleri
- **Analiz Kontrolü**: Incremental/tam analiz başlatma seçenekleri
- **Tablo Analizi**: Seçilen tabloların detaylı analiz sonuçlarını gösterme paneli

### 4. Karşılaştırma Paneli
- **Çift Görünüm**: Yan yana KVT/HVT karşılaştırması
- **Vurgu Sistemi**: Eşleşen veya şüpheli verilerin görsel vurgulanması
- **Ağırlık Ayarları**: Veri türlerine göre ağırlık ayarlamak için sezgisel arayüz
- **Eşik Değeri Kontrolü**: Risk sınıflandırma eşiklerini ayarlamak için kaydırıcılar
- **Detay Görünümü**: Şüpheli eşleşmelerin detaylı inceleme paneli

### 5. Rapor ve Alarm Merkezi
- **İnteraktif Raporlar**: Filtrelenebilir ve özelleştirilebilir rapor görünümleri
- **Veri Görselleştirme**: Heat map, çubuk grafik, pasta grafik gibi çeşitli görselleştirmeler
- **Bildirim Yönetimi**: Alarm konfigürasyonu ve bildirim geçmişi
- **Dışa Aktarma**: PDF, Excel, CSV formatlarında rapor dışa aktarma
- **Paylaşım Seçenekleri**: Raporları e-posta veya sistem içi paylaşım özellikleri

### 6. Kullanıcı Deneyimi Özellikleri
- **Tema Desteği**: Koyu/açık tema seçenekleri
- **Kısayol Tuşları**: Verimli kullanım için klavye kısayolları
- **Yardım Sistemi**: Bağlama duyarlı yardım baloncukları ve ipuçları
- **Kişiselleştirme**: Kullanıcıya özel görünüm ayarları
- **Duyarlı Tasarım**: Farklı ekran boyutlarına uyumlu arayüz
- **Sürüklenebilir Paneller**: Kullanıcının panelleri yeniden düzenleyebilmesi

### 7. Navigasyon
- **Hibrit Menü**: Sol yan menü + üst çubuk kombinasyonu
- **İşlev Odaklı Gruplandırma**: İlgili fonksiyonların mantıksal gruplandırılması
- **Hızlı Erişim**: Son kullanılan analiz/tablo/rapor kısayolları
- **Global Arama**: Tüm sistem içinde hızlı arama yapabilme
- **Breadcrumb Navigasyonu**: Kullanıcının nerede olduğunu gösteren yol haritası

### 8. Tasarım İlkeleri
- Material Design prensipleri uygulanacak
- Tepkisel (responsive) tasarım ile farklı ekranlara uyum
- Renk kodlaması ile risk seviyelerinin tutarlı gösterimi
- Sezgisel ve kolay öğrenilebilir arayüz
- Erişilebilirlik standartlarına uyum

---

## Kullanıcı Ayarları ve Özelleştirilebilirlik
- Her veritabanı için örnekleme oranı özelleştirebilme.
- Veri türleri ve risk eşikleri parametrik olarak değiştirilebilir.
- Kullanıcı tanımlı kolon ve veri türleri eklenebilir.
- Kullanıcı tanımlı uyarı ve alarm kuralları oluşturulabilir.

---

## Raporlama ve Alarm
- Şüpheli durumlar için detaylı raporlama ve risk sınıflandırması yapılır.
- Yüksek riskli durumlar için otomatik bildirim sistemi (email vb.) entegre edilir.
- Raporlar kullanıcı tarafından filtrelenip indirilebilir (PDF, Excel vb.)

---

Bu doküman, DB-Sentinel uygulamasının geliştiriciler tarafından net ve doğru biçimde anlaşılmasını sağlayarak geliştirme sürecini verimli hale getirmeyi amaçlamaktadır.
