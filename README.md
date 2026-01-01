# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [ ]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [X]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [X]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [ ]  LRU / CLOCK gibi algoritmaları
- [X]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [X]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [X]  WAL (Write Ahead Log) İlkesi
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo

| Kavram      | Bellek          | Disk / DB      |
| ----------- | --------------- | -------------- |
| Adresleme   | Pointer         | Page + Offset  |
| Hız         | O(1)            | Page IO        |
| PK          | Yok             | Index anahtarı |
| Veri yapısı | Array / Pointer | B+Tree         |
| Cache       | CPU cache       | Buffer Pool    |

---

# Video [Linki](https://www.youtube.com/watch?v=Nw1OvCtKPII&t=2635s) 
Ekran kaydı. 2-3 dk. açık kaynak V.T. kodu üzerinde konunun gösterimi. Video kendini tanıtma ile başlamalıdır (Numara, İsim, Soyisim, Teknik İlgi Alanları). 

---

# Açıklama

Bu doküman, PostgreSQL'in neden satır bazlı değil sayfa (page) bazlı çalıştığını; bellek yönetimi, veri yapıları ve yazma güvenliği perspektiflerinden ele alan teknik bir açıklamadır. Açıklamalar doğrudan PostgreSQL kaynak kodlarıyla ilişkilendirilmiştir.

1. Sistem Perspektifi: Sayfa Yapısı ve Blok Bazlı Erişim

Modern işletim sistemleri ve disk donanımları, veriye karakter bazlı erişmek yerine blok adı verilen sabit boyutlu birimlerle erişim sağlar. Veritabanları, bu donanım mimarisine uyum sağlamak ve disk I/O maliyetini düşürmek amacıyla sayfayı kullanır.

PostgreSQL kaynak kodlarında src/include/storage/bufpage.h içerisinde tanımlanan PageHeaderData yapısı, veritabanının neden satır bazlı değil de sayfa bazlı okuma yaptığı gösteriyor. Koddaki önemli satırlardan birisi olan pd_linp dizisi, sayfanın başında yer alan bir içerik indeksidir.

Eğer veritabanı diskten her seferinde tek bir satır okuyor olsaydı, sayfa başında o sayfa içindeki tüm satırların adreslerini tutan bir dizinin bulunmasına gerek duyulmazdı. pd_lower ve pd_upper değişkenleri ise sayfa alanı içinde verilerin ve işaretçilerin birbirine çarpmadan nasıl yerleştirileceğini, yani Slotted Page mimarisini yönetir.

Bu yapı sayesinde veritabanı, işletim sisteminden tek bir I/O çağrısıyla tüm sayfayı ram'e alır ve içindeki onlarca satıra işlemci seviyesinde hızlarla erişebilir. Bu yapı, disk kafasının rastgele hareketini minimize ederek toplam sistem performansını artıran temel bir sistem programlama yaklaşımıdır.

2. Bellek Yönetimi: Buffer Pool ve I/O Minimizasyonu

Veritabanları, diskteki fiziksel yavaşlığı telafi etmek için kendi yazılımsal önbellek mekanizmaları olan Buffer Pool yapısını kullanır. PostgreSQL'de bu mekanizmanın merkezi, src/backend/storage/buffer/bufmgr.c dosyasında yer alan ReadBuffer_common fonksiyonudur.

Sistem bir veri talep ettiğinde süreç PinBufferForBlock fonksiyonu ile başlar. Bu fonksiyon, istenen sayfanın RAM'de olup olmadığını bir Hash Table üzerinden kontrol eder:

Sayfa bellekteyse bu durum Buffer Hit olarak adlandırılır ve fiziksel disk erişimi tamamen baypas edilir. Veriye yaklaşık O(1) karmaşıklığında erişilir.

Sayfa bellekte değilse Buffer Miss oluşur ve StartReadBuffer fonksiyonu devreye girerek pread gibi düşük seviyeli I/O sistem çağrılarını tetikler.

Diskten okunan veri RAM'deki boş bir buffer alanına kopyalanır ve kullanımda olduğu sürece pin edilir. Bu mekanizma, sık erişilen verilerin bellekte kalmasını sağlayarak sistemin en yavaş bileşeni olan disk erişimlerini dramatik biçimde azaltır.

3. Veri Yapıları Perspektifi: B+ Tree İndeksleme

Verinin diskte hangi sayfada bulunduğunu hızlıca saptamak, Buffer Pool verimliliği açısından hayati önem taşır. PostgreSQL bu problemi, src/backend/access/nbtree/nbtsearch.c dosyasındaki _bt_search fonksiyonu ile çözer.

Bu fonksiyon, disk blokları için optimize edilmiş ve yüksek dallanma katsayısına sahip olan B+ Tree veri yapısını kullanır. Klasik ikili arama ağaçlarının aksine, B+ Tree her düğümde yüzlerce anahtar tutabilir. Bu sayede ağacın yüksekliği minimumda tutulur.

_bt_search fonksiyonu kök sayfadan başlayarak yaprak sayfalara kadar iner ve hedef verinin fiziksel adresini bulur. Bu yapı sayesinde milyonlarca satırlık bir tabloda bile yalnızca 3–4 sayfa okuması ile hedef veriye ulaşılabilir. Bu, rastgele disk taraması yerine hedef odaklı ve sınırlı I/O ile sonuç alma başarısıdır.

4. Yazma Performansı ve Güvenlik: WAL (Write-Ahead Logging)

Performans yalnızca okuma operasyonlarıyla değil, aynı zamanda yazma hızıyla da ölçülür. Rastgele disk yazma işlemleri, özellikle mekanik disklerde oldukça maliyetlidir.

PostgreSQL, src/backend/access/transam/xlog.c dosyasında yer alan XLogInsertRecord fonksiyonu ile Write-Ahead Log (WAL) ilkesini uygular. Bu ilkeye göre, bir veri değişikliği yapıldığında asıl tablo dosyası hemen güncellenmez bunun yerine yapılan değişikliğin özeti, sıralı (sequential) bir günlük dosyasına yazılır. Sıralı yazma işlemi, disk kafasının sürekli hareket etmesini gerektirmediği için rastgele yazmaya kıyasla çok daha hızlıdır.

Ayrıca kilit mekanizmalarıyla korunan bu süreç, sistem çökse bile WAL kayıtları sayesinde veritabanının kendini toparlayabilmesini ve veri kaybının önlenmesini sağlar. Böylece hem yazma performansı korunur hem de veri güvenliği en üst seviyeye çıkarılır.

Sonuç

Veritabanı performansı; donanım kısıtlarını yazılımsal katmanlarla (Buffer Pool, WAL) ve disk dostu veri yapılarıyla (B+ Tree, Slotted Page) aşma sanatıdır. PostgreSQL kaynak kodları, bu mimarinin pratikte nasıl uygulandığını açık ve öğretici bir biçimde ortaya koymaktadır.
---

## VT Üzerinde Gösterilen Kaynak Kodları

Sayfa Yönetimi [Linki]([https://...](https://github.com/postgres/postgres/blob/master/src/include/storage/bufpage.h)) \
Buffer Pool Kontrolü [Linki]([https://...](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/bufmgr.c#L1268)) \
B+ Algoritması [Linki]([https://...](https://github.com/postgres/postgres/blob/master/src/backend/access/nbtree/nbtsearch.c)) \
WAL [Linki](https://github.com/postgres/postgres/blob/master/src/backend/access/transam/xlog.c) \
...
