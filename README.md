# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [X]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [X]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [X]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [X]  LRU / CLOCK gibi algoritmaları
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
- [X]  Log disk (fsync vs write) sistem çağrıları farkı

---

# SQLite Performans Analizi: Bellek-Disk Karşılaştırması

| Kavram | Bellek (C / Sistem) | Veri Tabanı (SQLite / Disk) | Teknik Açıklama |
|--------|---------------------|----------------------------|-----------------|
| **Adresleme** | RAM Adresi (Pointer) | Pgno (Page Number) + Offset | Bellekte veriye doğrudan pointer ile erişilir; diskte ise sayfa numarası (Pgno) ve sayfa içi konum (offset) ile dolaylı erişim yapılır. SQLite iki seviyeli adresleme kullanır: önce sayfa bulunur, sonra sayfa içinde konum hesaplanır. |
| **Erişim Hızı** | O(1) (Doğrudan) | Page I/O (Milisaniye seviyesi) | Bellek erişimi nanosaniye mertebesinde sabit zamanlıyken, disk erişimi fiziksel kafa hareketi ve dönel gecikme (rotational delay) nedeniyle milisaniye mertebesindedir. Yaklaşık 1 milyon kat hız farkı vardır. |
| **Kimlik (Key)** | Değişken Adı / Adres | rowid (INTEGER, B+ Tree Key) | RAM'de veriye değişken ismi veya adres ile ulaşılır, Primary Key kavramı yoktur. SQLite'da her satırın otomatik atanan benzersiz bir rowid'si vardır ve bu B+ Tree yapısının anahtarı olarak kullanılır. |
| **Veri Yapısı** | Array / Struct / Linked List | B+ Tree / Page Layout | RAM'de basit array ve liste yapıları yeterlidir çünkü tüm veri bellektedir. Diskte ise arama maliyetini düşürmek için dengeli B+ Tree index yapısı, veriyi kompakt tutmak için Page Layout kullanılır. |
| **Ön Bellek** | CPU L1/L2/L3 Cache | Buffer Pool (LRU - PgHdr1) | CPU cache'i donanımsal ve otomatik olarak çalışır. SQLite ise yazılımsal Buffer Pool kullanır; pcache1.c dosyasındaki LRU algoritması ile sık kullanılan sayfalar (PgHdr1 yapısı) RAM'de tutularak disk I/O minimize edilir. |
| **Güvenlik** | Yok (Uçucu veri) | WAL (Write-Ahead Logging) + fsync | RAM'deki veri elektrik kesildiğinde tamamen kaybolur (volatile). SQLite ise önce değişiklikleri WAL dosyasına yazar, ardından fsync sistem çağrısı ile diske zorlar (force flush). Böylece crash durumunda bile veri kurtarılabilir ve tutarlılık garanti edilir. |

---

# Video [Linki](https://www.youtube.com/watch?v=Nw1OvCtKPII&t=2635s) 
Ekran kaydı. 2-3 dk. açık kaynak V.T. kodu üzerinde konunun gösterimi. Video kendini tanıtma ile başlamalıdır (Numara, İsim, Soyisim, Teknik İlgi Alanları). 

---

# Açıklama

Bu çalışmada, açık kaynaklı ilişkisel veritabanı yönetim sistemi olan SQLite'ın kaynak kodları incelenerek; sistem programlama (işletim sistemi, disk I/O, bellek yönetimi) ve veri yapıları (B+Tree, Page Layout, LRU) perspektifinden performansı optimize etmek amacıyla hazırlanan mekanizmalar analiz edilmiştir. Çalışmanın amacı genel amaçlı dosya sistemlerinin üstüne geçerli, disk erişim maliyetlerini minimize etmek ve veri bütünlüğünü sağlamak için tasarlanmış mimarileri kavramaktır.

### Sistem Perspektifi: Disk ve Sayfa Bazlı Erişim

Disk, işletim sistemi ve veritabanı yönetim sistemleri için en temel veri depolama alanıdır ancak fiziksel olarak yavaş bir donanımdır. SQLite, veriyi satır satır değil, sayfa (page) bazında organize eder. Her sayfa varsayılan olarak 4 kilobyte boyutundadır ve bir sayfa içinde birden fazla satır bulunabilir. src/pager.c dosyasındaki `sqlite3PagerGet` fonksiyonu bir sayfa numarası (`Pgno pgno`) alır ve o sayfanın tamamını diskten RAM'e yükler. `sqlite3PagerWrite` fonksiyonu ise bir sayfaya yazma yapılmadan önce (`PgHdr *pPg` parametresi ile) o sayfayı yazılabilir hale getirir ve gerekli logging mekanizmasını (journal veya WAL) tetikler. Bu sayede crash durumunda rollback işlemi gerçekleştirilebilir.

Disk erişimi blok bazlıdır; tek bir satırı okumak için bile tüm sayfa okunmalıdır. Bu nedenle SQLite, sık erişilen sayfaları RAM'de tutarak (Buffer Pool) gereksiz disk I/O işlemlerini önler.

### Buffer Pool ve LRU Algoritması ile Bellek Yönetimi

SQLite, disk I/O maliyetini azaltmak için sık kullanılan sayfaları bellekte önbelleğe alır. Bu mekanizmaya Buffer Pool (Page Cache) denir. src/pcache1.c dosyasında tanımlanan `struct PgHdr1` yapısı her sayfayı temsil eder ve `pLruNext` ile `pLruPrev` pointer'ları aracılığıyla dairesel bir LRU (Least Recently Used) listesi oluşturulur.

`pcache1Fetch` fonksiyonu bir sayfa istendiğinde önce cache'de arar. Eğer sayfa bellekteyse disk'e gitmeden doğrudan döndürülür. `pcache1Unpin` fonksiyonu ise kullanımı biten bir sayfayı LRU listesine ekler. Bellek dolduğunda, en az kullanılan sayfa (liste başındaki) çıkarılır ve yerine yeni sayfa yerleştirilir. Bu algoritma sayesinde sık erişilen sayfalar RAM'de kalır, eski sayfalar ise disk I/O gerektiğinde evict edilir.

LRU listesi dairesel (circular) yapıdadır; en yeni kullanılan sayfa listenin sonuna, en eski kullanılan sayfa başına eklenir. Bu tasarım, sık kullanılan root node gibi kritik sayfaların sürekli bellekte kalmasını sağlar.

### Veri Yapıları: B+ Tree ve Sayfa Organizasyonu

SQLite, verileri disk üzerinde rastgele değil, B+ Tree (B Artı Ağacı) veri yapısı ile organize eder. B+ Tree dengeli bir ağaç yapısıdır ve her düğüm bir disk sayfasına karşılık gelir. Gerçek satır verileri yalnızca yaprak (leaf) düğümlerde bulunur; iç düğümler (internal nodes) sadece yönlendirme bilgisi tutar.

src/btree.c dosyasındaki `sqlite3BtreeTableMoveto` fonksiyonu, rowid ile arama yaparken (`i64 intKey` parametresi) B+ Tree'nin kök sayfasından başlayarak aşağı iner. Her seviyede karşılaştırma yapılır ve doğru alt düğüme geçilir. Milyonlarca satır içeren bir tabloda bile logaritmik zaman karmaşıklığı sayesinde sadece 3-4 sayfa okunarak hedef satıra ulaşılır.

`sqlite3BtreeIndexMoveto` fonksiyonu ise indeks B+ Tree'lerinde çok kolonlu anahtarlarla arama yapar. Anahtar `UnpackedRecord` yapısı ile temsil edilir; bu yapı birden fazla sütunun değerini, tipini ve sıralama kuralını içerir. Bu sayede WHERE, JOIN ve ORDER BY gibi sorgular tam tablo taraması yapmadan hızlıca sonuçlanabilir.

B+ Tree'nin yaprak düğümleri birbirine bağlıdır (linked list). Bu özellik, aralık sorguları (range scan) için kritik önem taşır; bir anahtar bulunduktan sonra ağacın köküne dönmeden yan taraftaki sayfalara geçilerek sıralı veri okunabilir.

### Veri Bütünlüğü ve WAL (Write-Ahead Logging)

SQLite WAL modunda çalışırken, veritabanındaki değişiklikler doğrudan ana dosyaya yazılmaz. Bunun yerine tüm güncellemeler önce src/wal.c dosyasındaki `sqlite3WalFrames` fonksiyonu aracılığıyla WAL dosyasına kaydedilir. Her değiştirilen sayfa bir "frame" olarak WAL'a eklenir ve commit işlemi özel bir commit frame ile işaretlenir.

Bu yaklaşımın en büyük avantajı, okuyucuların (reader) hiçbir zaman engellenmemesidir. Ana veritabanı dosyası değişmediği için okuyucular tutarlı bir snapshot üzerinden okumaya devam edebilir. Yazıcı (writer) ise sadece WAL dosyasına yazar ve bu işlem çok hızlıdır.

`sqlite3WalFrames` fonksiyonundaki `sync_flags` parametresi, sistem seviyesinde kritik bir ayrımı temsil eder: verinin sadece işletim sistemi buffer'ına (`write` sistem çağrısı) yazılması mı, yoksa `fsync` sistem çağrısı ile fiziksel diske zorlanması mı gerektiğini belirler. `write()` çağrısı hızlı ancak crash durumunda veri kaybolabilir çünkü veri henüz OS buffer'ında bekliyor olabilir. `fsync()` ise daha yavaş olmakla birlikte verinin kalıcı depolama alanına yazılmasını garanti eder ve atomik işlem özelliği sağlar. Bu mekanizma, SQLite'ın ACID (Atomicity, Consistency, Isolation, Durability) özelliklerinden Durability'yi sağlamasının temelini oluşturur.

Zamanla büyüyen WAL dosyası `sqlite3WalCheckpoint` fonksiyonu ile temizlenir. Bu işlem, WAL'daki değişiklikleri ana veritabanı dosyasına kopyalar (backfill). Ancak hâlâ okuma yapan transaction'lar varsa, checkpoint onları bekler ve veri tutarlılığını korur.

### Sonuç

SQLite, sistem programlama ilkeleri ve veri yapılarını etkin kullanarak yüksek performans elde eder. Sayfa bazlı disk erişimi, LRU algoritması ile bellek yönetimi, B+ Tree indeksleme ve WAL protokolü birlikte çalışarak disk I/O'yu minimize eder. Bu tasarım kararları sayesinde SQLite, verimli bir veritabanı altyapısı sunar.

## VT Üzerinde Gösterilen Kaynak Kodları

**1. Blok Bazlı Disk Erişimi (block_id + offset):** [pager.c - sqlite3PagerGet](https://github.com/sqlite/sqlite/blob/master/src/pager.c)

**2. VT Sayfa Okuması (Satır/Sayfa):** [pager.c - sqlite3PagerWrite](https://github.com/sqlite/sqlite/blob/master/src/pager.c)

**3. Sık Kullanılan Sayfaları RAM'de Kopyalama (Caching):** [pcache1.c - pcache1Fetch](https://github.com/sqlite/sqlite/blob/master/src/pcache1.c)

**4. LRU Algoritması:** [pcache1.c - struct PgHdr1 ve pcache1Unpin](https://github.com/sqlite/sqlite/blob/master/src/pcache1.c)

**5. Disk I/O Minimizasyonu:** [pcache1.c - Buffer Pool Mekanizması](https://github.com/sqlite/sqlite/blob/master/src/pcache1.c)

**6. B+ Tree Veri Yapısı Kullanımı:** [btree.c - sqlite3BtreeTableMoveto ve sqlite3BtreeIndexMoveto](https://github.com/sqlite/sqlite/blob/master/src/btree.c)

**7. WAL (Write Ahead Log) İlkesi:** [wal.c - sqlite3WalFrames](https://github.com/sqlite/sqlite/blob/master/src/wal.c)

**8. fsync vs write Sistem Çağrıları Farkı:** [wal.c - sqlite3WalFrames (sync_flags parametresi)](https://github.com/sqlite/sqlite/blob/master/src/wal.c)
