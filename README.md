# Deney Sonu Teslimatı

## Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Veritabanı performansı temel olarak **disk ve bellek arasındaki hız farkını** yönetme problemidir. Disk erişimi milisaniye, bellek erişimi nanosaniye mertebesindedir; bu yaklaşık **milyon kat farktır**.

### Disk Erişim Stratejisi

Diskler **blok bazlı** çalışır; satır bazlı okuma fiziksel olarak mümkün değildir. Veritabanları bu gerçeği kabul ederek veriyi sabit boyutlu **sayfalara (page)** böler. Bir sayfaya erişmek zorunda kalındığında, o sayfanın tamamı okunur. SQLite'da `sqlite3PagerGet` fonksiyonu **Pgno (Page Number)** parametresi ile çalışır ve bu yaklaşımı uygular. Sonraki satırlara erişim ücretsiz hale gelir ve **rastgele erişim** yerine **sıralı erişim** teşvik edilir.

### Bellek Hiyerarşisi ve Önbellekleme

Sık kullanılan sayfalar RAM'de tutulur; buna **Buffer Pool** denir. `pcache1Fetch` fonksiyonu bir sayfa istendiğinde önce cache'de arar. Bellek dolduğunda **sayfa değiştirme algoritmaları** devreye girer. **LRU (Least Recently Used)** en bilinen algoritmadır; `pcache1Unpin` fonksiyonu bu mekanizmayı `struct PgHdr1` yapısındaki `pLruNext` ve `pLruPrev` pointer'ları ile gerçekleştirir. **CLOCK** gibi daha verimli varyasyonlar da mevcuttur. Bu algoritmaların amacı **cache hit oranını** maksimize etmektir; her **cache miss** bir disk I/O anlamına gelir.

### Veri Organizasyonu ve Index Yapıları

**B+ Tree** dengeli bir ağaç yapısıdır ve veritabanlarında standart haline gelmiştir. Yüksek **branching factor** sayesinde ağaç kısa kalır; bu da **logaritmik aramayı** pratikte 3-4 disk erişimine indirger. `sqlite3BtreeTableMoveto` fonksiyonu **rowid** ile arama yaparken bu yapıyı kullanır. B+ Tree'nin yapraklarının bağlı olması, **aralık sorgularını (range scan)** verimli hale getirir. 

Farklı sistemler farklı varyasyonlar kullanır: **InnoDB** clustered index tercih eder, **PostgreSQL** heap-based yaklaşım kullanır, **LSM-tree** tabanlı sistemler (LevelDB, RocksDB) yazma ağırlıklı işler için optimize edilmiştir.

### Index Türleri

**Clustered index** verinin kendisini içerir ve primary key'e göre fiziksel sıralamayı belirler. **Non-clustered index** ayrı bir yapıdır ve sadece key-pointer çiftleri tutar. `sqlite3BtreeIndexMoveto` fonksiyonu çok kolonlu anahtarlarla arama yaparken **UnpackedRecord** yapısını kullanır. Her ikisinin de avantaj ve dezavantajları vardır; seçim, okuma/yazma oranına ve sorgu tiplerine göre yapılır.

### Kalıcılık ve Sistem Çağrıları

**Write-Ahead Logging (WAL)** prensibi, değişikliklerin önce log'a yazılmasını zorunlu kılar. `sqlite3WalFrames` fonksiyonu bu işlemi gerçekleştirir. Ancak `write()` sistem çağrısı yeterli değildir; veri işletim sistemi buffer'ında kalabilir. `fsync()` çağrısı fiziksel diske yazılmayı garanti eder. İki çağrı arasındaki **trade-off** performans ve güvenlik dengesini belirler; `write()` hızlı ancak crash'te veri kaybı riski taşır, `fsync()` güvenli ancak yavaştır. `sqlite3WalFrames` fonksiyonundaki **sync_flags** parametresi bu ayrımı kontrol eder.

### Sonuç

Veritabanı performansı; disk fiziksel sınırlamalarını aşmak için **sayfa bazlı erişim**, sık kullanılan verileri bellekte tutmak için **önbellekleme**, hızlı arama için **dengeli ağaç yapıları** ve veri bütünlüğü için **doğru sistem çağrılarının** kombinasyonudur.
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

# RAM ve Disk Üzerinde Veri Erişimi: Genel Sistem vs SQLite

| Kavram | Bellek (C / Sistem) | Veri Tabanı (SQLite / Disk) | Teknik Açıklama |
|--------|---------------------|----------------------------|-----------------|
| **Adresleme** | RAM Adresi (Pointer) | Pgno (Page Number) + Offset | Bellekte veriye doğrudan pointer ile erişilir; diskte ise sayfa numarası (Pgno) ve sayfa içi konum (offset) ile dolaylı erişim yapılır. SQLite iki seviyeli adresleme kullanır: önce sayfa bulunur, sonra sayfa içinde konum hesaplanır. |
| **Erişim Hızı** | O(1) (Doğrudan) | Page I/O (Milisaniye seviyesi) | Bellek erişimi nanosaniye mertebesinde sabit zamanlıyken, disk erişimi fiziksel kafa hareketi ve dönel gecikme (rotational delay) nedeniyle milisaniye mertebesindedir. Yaklaşık 1 milyon kat hız farkı vardır. |
| **Kimlik (Key)** | Değişken Adı / Adres | rowid (INTEGER, B+ Tree Key) | RAM'de veriye değişken ismi veya adres ile ulaşılır, Primary Key kavramı yoktur. SQLite'da her satırın otomatik atanan benzersiz bir rowid'si vardır ve bu B+ Tree yapısının anahtarı olarak kullanılır. |
| **Veri Yapısı** | Array / Struct / Linked List | B+ Tree / Page Layout | RAM'de basit array ve liste yapıları yeterlidir çünkü tüm veri bellektedir. Diskte ise arama maliyetini düşürmek için dengeli B+ Tree index yapısı, veriyi kompakt tutmak için Page Layout kullanılır. |
| **Ön Bellek** | CPU L1/L2/L3 Cache | Buffer Pool (LRU - PgHdr1) | CPU cache'i donanımsal ve otomatik olarak çalışır. SQLite ise yazılımsal Buffer Pool kullanır; LRU algoritması ile sık kullanılan sayfalar  RAM'de tutularak disk I/O minimize edilir. |
| **Güvenlik** | Yok (Uçucu veri) | WAL (Write-Ahead Logging) + fsync | RAM'deki veri elektrik kesildiğinde tamamen kaybolur (volatile). SQLite ise önce değişiklikleri WAL dosyasına yazar, ardından fsync sistem çağrısı ile diske zorlar (force flush). Böylece crash durumunda bile veri kurtarılabilir ve tutarlılık garanti edilir. |

---

# Video [Linki](https://www.youtube.com/watch?v=Nw1OvCtKPII&t=2635s) 

---

# Açıklama

Bu çalışmada, açık kaynaklı ilişkisel veritabanı yönetim sistemi olan SQLite'ın kaynak kodları incelenerek; sistem programlama (işletim sistemi, disk I/O, bellek yönetimi) ve veri yapıları (B+Tree, Page Layout, LRU) perspektifinden performansı optimize etmek amacıyla hazırlanan mekanizmalar analiz edilmiştir. Çalışmanın amacı genel amaçlı dosya sistemlerinin üstüne geçerli, disk erişim maliyetlerini minimize etmek ve veri bütünlüğünü sağlamak için tasarlanmış mimarileri kavramaktır.

### Sistem Perspektifi: Disk ve Sayfa Bazlı Erişim

Disk, işletim sistemi ve veritabanı yönetim sistemleri için en temel veri depolama alanıdır ancak fiziksel olarak yavaş bir donanımdır. SQLite, veriyi satır satır değil, sayfa (page) bazında organize eder. Her sayfa varsayılan olarak 4 kilobyte boyutundadır ve bir sayfa içinde birden fazla satır bulunabilir. `src/pager.c ` dosyasındaki `sqlite3PagerGet` fonksiyonu bir sayfa numarası (`Pgno pgno`) alır ve o sayfanın tamamını diskten RAM'e yükler. `sqlite3PagerWrite` fonksiyonu ise bir sayfaya yazma yapılmadan önce (`PgHdr *pPg` parametresi ile) o sayfayı yazılabilir hale getirir ve gerekli logging mekanizmasını (journal veya WAL) tetikler. Bu sayede crash durumunda rollback işlemi gerçekleştirilebilir. Disk erişimi blok bazlıdır; tek bir satırı okumak için bile tüm sayfa okunmalıdır. Bu nedenle SQLite, sık erişilen sayfaları RAM'de tutarak (Buffer Pool) gereksiz disk I/O işlemlerini önler.

### Buffer Pool ve LRU Algoritması ile Bellek Yönetimi

SQLite, disk I/O maliyetini azaltmak için sık kullanılan sayfaları bellekte önbelleğe alır. Bu mekanizmaya Buffer Pool (Page Cache) denir. `src/pcache1.c ` dosyasında tanımlanan `struct PgHdr1` yapısı her sayfayı temsil eder ve `pLruNext` ile `pLruPrev` pointer'ları aracılığıyla dairesel bir LRU (Least Recently Used) listesi oluşturulur. `pcache1Fetch` fonksiyonu bir sayfa istendiğinde önce cache'de arar. Eğer sayfa bellekteyse disk'e gitmeden doğrudan döndürülür. `pcache1Unpin` fonksiyonu ise kullanımı biten bir sayfayı LRU listesine ekler. Bellek dolduğunda, en az kullanılan sayfa (liste başındaki) çıkarılır ve yerine yeni sayfa yerleştirilir. Bu algoritma sayesinde sık erişilen sayfalar RAM'de kalır, eski sayfalar ise disk I/O gerektiğinde evict edilir. LRU listesi dairesel yapıdadır; en yeni kullanılan sayfa listenin sonuna, en eski kullanılan sayfa başına eklenir. Bu tasarım, sık kullanılan kritik sayfaların sürekli bellekte kalmasını sağlar.

### Veri Yapıları: B+ Tree ve Sayfa Organizasyonu

SQLite, verileri disk üzerinde rastgele değil, B+ Tree (B Artı Ağacı) veri yapısı ile organize eder. B+ Tree dengeli bir ağaç yapısıdır ve her düğüm bir disk sayfasına karşılık gelir. Gerçek satır verileri yalnızca yaprak (leaf) düğümlerde bulunur; iç düğümler (internal nodes) sadece yönlendirme bilgisi tutar. `src/btree.c` dosyasındaki `sqlite3BtreeTableMoveto` fonksiyonu, rowid ile arama yaparken (`i64 intKey` parametresi) B+ Tree'nin kök sayfasından başlayarak aşağı iner. Her seviyede karşılaştırma yapılır ve doğru alt düğüme geçilir. Milyonlarca satır içeren bir tabloda bile logaritmik zaman karmaşıklığı sayesinde sadece 3-4 sayfa okunarak hedef satıra ulaşılır. `sqlite3BtreeIndexMoveto` fonksiyonu ise indeks B+ Tree'lerinde çok kolonlu anahtarlarla arama yapar. Anahtar `UnpackedRecord` yapısı ile temsil edilir; bu yapı birden fazla sütunun değerini, tipini ve sıralama kuralını içerir. Bu sayede WHERE, JOIN ve ORDER BY gibi sorgular tam tablo taraması yapmadan hızlıca sonuçlanabilir. B+ Tree'nin yaprak düğümleri birbirine bağlıdır (linked list). Bu özellik, aralık sorguları (range scan) için kritik önem taşır; bir anahtar bulunduktan sonra ağacın köküne dönmeden yan taraftaki sayfalara geçilerek sıralı veri okunabilir.

### Veri Bütünlüğü ve WAL (Write-Ahead Logging)

SQLite WAL modunda çalışırken, veritabanındaki değişiklikler doğrudan ana dosyaya yazılmaz. Bunun yerine tüm güncellemeler önce `src/wal.c` dosyasındaki `sqlite3WalFrames` fonksiyonu aracılığıyla WAL dosyasına kaydedilir. Her değiştirilen sayfa bir "frame" olarak WAL'a eklenir ve commit işlemi özel bir commit frame ile işaretlenir. Bu yaklaşımın en büyük avantajı, okuyucuların hiçbir zaman engellenmemesidir. Ana veritabanı dosyası değişmediği için okuyucular tutarlı bir snapshot üzerinden okumaya devam edebilir. Yazıcı ise sadece WAL dosyasına yazar ve bu işlem çok hızlıdır. `sqlite3WalFrames` fonksiyonundaki `sync_flags` parametresi, sistem seviyesinde kritik bir ayrımı temsil eder: verinin sadece işletim sistemi buffer'ına (`write` sistem çağrısı) yazılması mı, yoksa `fsync` sistem çağrısı ile fiziksel diske zorlanması mı gerektiğini belirler. `write()` çağrısı hızlı ancak crash durumunda veri kaybolabilir çünkü veri henüz OS buffer'ında bekliyor olabilir. `fsync()` ise daha yavaş olmakla birlikte verinin kalıcı depolama alanına yazılmasını garanti eder ve atomik işlem özelliği sağlar. Bu mekanizma, SQLite'ın ACID (Atomicity, Consistency, Isolation, Durability) özelliklerinden Durability'yi sağlamasının temelini oluşturur. Zamanla büyüyen WAL dosyası `sqlite3WalCheckpoint` fonksiyonu ile temizlenir. Bu işlem, WAL'daki değişiklikleri ana veritabanı dosyasına kopyalar (backfill). Ancak hâla okuma yapan transaction'lar varsa, checkpoint onları bekler ve veri tutarlılığını korur.

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
