# 💣 SQL Injection (OR '1'='1') - Android Security Demo

## 📱 Proje Hakkında

Bu proje, bir Android uygulamasında **SQLite Veritabanı** üzerinde gerçekleştirilen klasik **SQL Injection (SQLi)** zafiyetini uygulamalı olarak göstermek amacıyla oluşturulmuştur. Bu demo, kullanıcı girdisinin veritabanı sorgularına doğrudan eklenmesinin yol açtığı kritik güvenlik açığını sergiler.

**AMAÇ:** Geliştiricilerin, kullanıcı doğrulama (login) işlemlerinde parametreleştirilmiş sorgu (Bound Arguments) kullanmanın önemini kavraması ve SQL Injection'dan korunma yöntemlerini öğrenmesi.

---

## 🔓 Zafiyetler

### 1. **Düz Metin Birleştirme (String Concatenation)**
- Kullanıcı adı ve şifre girdileri, sorgu metnine doğrudan **Kotlin String Interpolation (`$`)** ile ekleniyor.
- Bu, saldırganın tırnak işaretleri (`'`) ve SQL komutları (`OR 1=1`, `--`) enjekte etmesine olanak tanır.

![DB login Logic ](https://i.hizliresim.com/ejeeutq.png)


### 2. **Yetkisiz Giriş (Bypass Authentication)**
- Saldırgan, şifre alanına `' OR '1'='1` gibi bir yük (payload) girerek, geçerli bir şifreye sahip olmadan sisteme giriş yapabilir.
- Saldırı sonucunda, SQL sorgusu mantıksal olarak **her zaman doğru (TRUE)** döner ve yetkilendirme mekanizması atlanmış olur.

![OR 1=1  ](https://i.hizliresim.com/ipjxbnw.png)

### 3. **Zafiyetli SQL Sorgusu İncelemesi**

Kullanıcı, Şifre alanına **`' OR '1'='1`** girdiğinde, uygulama katmanında oluşan zafiyetli sorgu örneği:

```sql
SELECT * FROM users WHERE username = '...' AND password = '' OR '1'='1'

```

ve Saldırgan password'u bypass ederek ilgili username ile giriş yapar

![UI detail ](https://i.hizliresim.com/dn6gejq.png)

## 🛡️ Nasıl Korunulur? (The Secure Way)
SQL Injection'dan korunmanın temel yolu, kullanıcı girdisini SQL kodu olarak değil, sadece bir veri değeri olarak ele almaktır. Bu, Bağlı Değişkenler (Bound Arguments) kullanılarak gerçekleştirilir.

### 1. Bağlı Değişkenler (? Yer Tutucular) Kullanın
Bu yöntem, kullanıcı girdisinin tırnak işaretleri ile çevrelenmesini ve sadece bir değer olarak işlenmesini sağlar. SQLiteOpenHelper sınıfındaki db.rawQuery() metodunu kullanırken, parametreleri ayırmak için soru işareti (?) yer tutucuları kullanırız.

### Güvenli (Soru İşareti Yer Tutucular):

```kotlin
// Güvenli: Girdiler selectionArgs dizisi aracılığıyla ayrı bir kanal üzerinden gönderilir.
fun loginSecure(username: String, password: String): Boolean {
    val db = readableDatabase
    
    // 1. Sorgu metninde değerler yerine ? yer tutucuları kullanılır.
    val selection = "username = ? AND password = ?"
    
    // 2. Kullanıcı girdileri, sorgu metninden ayrı olarak bir diziye konur.
    val selectionArgs = arrayOf(username, password) 

    val cursor = db.rawQuery(
        "SELECT * FROM users WHERE $selection",
        selectionArgs // Bu dizi, sırayla ? işaretlerinin yerine geçer.
    )
    return cursor.moveToFirst()
}
```

### Neden Güvenli?
Saldırgan aynı yükü (' OR '1'='1) girdiğinde, sistem bu ifadeyi bir kod parçası olarak değil, basitçe bir tek dize değeri olarak ele alır:

```sql
-- Çalıştırılan Sorgu:
SELECT * FROM users WHERE username = '...' AND password = ' ' OR '1'='1' ' 
-- Bu dize, hiçbir kullanıcıya ait olmadığı için giriş başarısız olur.
```

### 2. Room Persistence Library Kullanımı
Modern Android geliştirmede, manuel SQLiteOpenHelper kullanımı yerine Room Persistence Library kullanılması şiddetle önerilir. Room, yer tutucular kullanarak güvenli sorgular oluşturmayı zorunlu kılar ve geliştiricinin yanlışlıkla SQL Injection zafiyeti oluşturma riskini büyük ölçüde azaltır.

```kotlin
// Room DAO (Veri Erişim Nesnesi) Metodu
@Query("SELECT * FROM User WHERE username = :username AND password = :password LIMIT 1")
suspend fun login(username: String, password: String): User?
```
### Room, bu yapıyı otomatik olarak güvenli, parametreleştirilmiş bir sorguya çevirir.
