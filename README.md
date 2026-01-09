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
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

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

# Açıklama (Ort. 600 kelime)

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Suspendisse lacinia luctus urna, vel aliquet lacus facilisis ac. Donec quis placerat orci, efficitur consectetur lacus. Sed rhoncus erat ex, at sagittis velit mollis et. Aliquam enim orci, sollicitudin sit amet libero quis, mollis ultricies risus. Fusce tempor, felis a consequat tristique, dolor magna convallis nulla, vel ullamcorper magna mauris non ipsum. Pellentesque habitant morbi tristique senectus et netus et malesuada fames ac turpis egestas. Nam quis imperdiet ex, at blandit sapien. Aliquam lacinia erat ac ipsum fringilla, quis vestibulum augue posuere. Nulla in enim nulla. Nunc euismod odio mauris, sed sollicitudin ex condimentum non. In efficitur egestas enim. Fusce tempus erat quis placerat convallis.

Nam sit amet tincidunt ante. Pellentesque sit amet quam interdum, pellentesque dui vel, iaculis elit. Donec sed dui sodales nulla dignissim tincidunt. Maecenas semper metus id fermentum vulputate. Pellentesque lobortis hendrerit venenatis. Nullam imperdiet, ex eget ultricies egestas, mauris nunc aliquam ante, sed consectetur tellus ex vel leo. Nunc ut erat dapibus, auctor dolor eu, pretium sem. In lacinia congue eros et finibus. Aenean auctor, leo a feugiat placerat, urna felis lacinia purus, laoreet volutpat mi nisl eget dui. Ut vitae condimentum leo.

Maecenas ex diam, vehicula et nulla vel, mattis viverra metus. Nam at ex scelerisque, semper augue lobortis, semper est. Etiam id pretium odio, eget rutrum neque. Pellentesque blandit magna vel aliquam gravida. Nullam massa nisl, imperdiet at dapibus non, cursus vehicula turpis. Vestibulum rutrum hendrerit augue. Aliquam id nisi id arcu tempor venenatis vel nec erat. Morbi sed posuere erat. Morbi et sollicitudin urna. Suspendisse ullamcorper vitae purus sit amet sodales. Nam ut tincidunt ipsum, ut varius erat. Duis congue magna nec euismod condimentum. In hac habitasse platea dictumst. Nunc mattis odio sed enim laoreet imperdiet. In hac habitasse platea dictumst. Nullam tincidunt quis.

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Suspendisse lacinia luctus urna, vel aliquet lacus facilisis ac. Donec quis placerat orci, efficitur consectetur lacus. Sed rhoncus erat ex, at sagittis velit mollis et. Aliquam enim orci, sollicitudin sit amet libero quis, mollis ultricies risus. Fusce tempor, felis a consequat tristique, dolor magna convallis nulla, vel ullamcorper magna mauris non ipsum. Pellentesque habitant morbi tristique senectus et netus et malesuada fames ac turpis egestas. Nam quis imperdiet ex, at blandit sapien. Aliquam lacinia erat ac ipsum fringilla, quis vestibulum augue posuere. Nulla in enim nulla. Nunc euismod odio mauris, sed sollicitudin ex condimentum non. In efficitur egestas enim. Fusce tempus erat quis placerat convallis.

Nam sit amet tincidunt ante. Pellentesque sit amet quam interdum, pellentesque dui vel, iaculis elit. Donec sed dui sodales nulla dignissim tincidunt. Maecenas semper metus id fermentum vulputate. Pellentesque lobortis hendrerit venenatis. Nullam imperdiet, ex eget ultricies egestas, mauris nunc aliquam ante, sed consectetur tellus ex vel leo. Nunc ut erat dapibus, auctor dolor eu, pretium sem. In lacinia congue eros et finibus. Aenean auctor, leo a feugiat placerat, urna felis lacinia purus, laoreet volutpat mi nisl eget dui. Ut vitae condimentum leo.

Maecenas ex diam, vehicula et nulla vel, mattis viverra metus. Nam at ex scelerisque, semper augue lobortis, semper est. Etiam id pretium odio, eget rutrum neque. Pellentesque blandit magna vel aliquam gravida. Nullam massa nisl, imperdiet at dapibus non, cursus vehicula turpis. Vestibulum rutrum hendrerit augue. Aliquam id nisi id arcu tempor venenatis vel nec erat. Morbi sed posuere erat. Morbi et sollicitudin urna. Suspendisse ullamcorper vitae purus sit amet sodales. Nam ut tincidunt ipsum, ut varius erat. Duis congue magna nec euismod condimentum. In hac habitasse platea dictumst. Nunc mattis odio sed enim laoreet imperdiet. In hac habitasse platea dictumst. Nullam tincidunt quis.

## VT Üzerinde Gösterilen Kaynak Kodları

Açıklama [Linki](https://...) \
Açıklama [Linki](https://...) \
Açıklama [Linki](https://...) \
... \
...
