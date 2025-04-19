# Brute Force Saldırılarına Karşı SQL Server'da Başarısız Giriş Denemelerini Yakalama

SQL Server ortamlarında brute force saldırılarına karşı en önemli savunmalardan biri, başarısız giriş denemelerini tespit etmek ve bu verileri analiz etmektir. Bu yazıda, SQL Server Management Studio (SSMS) kullanarak nasıl başarısız girişleri yakalayabileceğimizi anlatacağım.

---

## İçindekiler
1. [Brute Force Saldırıları Nedir?](#brute-force-saldırıları-nedir)
2. [SQL Server'da Giriş Denemeleri Nereden Takip Edilir?](#sql-serverda-giriş-denemeleri-nereden-takip-edilir)
3. [Saldırıyı Tespit Etme](#saldırıyı-tespit-etme)


---

## Brute Force Saldırıları Nedir?

Brute force saldırıları, kullanıcı adı ve parola kombinasyonlarını sistematik şekilde deneyerek doğru giriş bilgilerini bulmaya çalışan bir saldırı türüdür. SQL Server'da da bu tür denemeler, `SQL Server Error Log` veya `sys.fn_get_audit_file` gibi yollarla takip edilebilir.

---

## SQL Server'da Giriş Denemeleri Nereden Takip Edilir?

### 1. SQL Server Error Log
SQL Server, başarısız oturum açma denemelerini varsayılan olarak error log dosyalarında saklar. Bu error log dosyalarına "Management" klasörünün alt klasörü "Sql Server Logs" klasöründen erişilebilir.

```sql
EXEC xp_readerrorlog;
```
Bu procedure giriş denemelerinin loglarını tutan dosyayı okur. Bu kod ile dönen sonucu bir TEMP tabloda saklarız çünkü işimiz bittiğinde bu tabloyu yok etmemiz gerekir böylece veritabınımıza yük bindirmemiş oluruz.

## Saldırıyı Tespit Etme.
```sql
CREATE TABLE #LOG(LOGDATE DATETIME,PROCESSINFO VARCHAR(100),TEXT VARCHAR(MAX))
INSERT INTO #LOG
EXEC xp_readerrorlog 0, 1, N'Login failed';
SELECT * FROM #LOG WHERE TEXT LIKE 'login failed%' AND LOGDATE>=DATEADD(MINUTE,-3,GETDATE())
TRUNCATE TABLE #LOG
```
Yukarıdaki kod bloğu ile 'xp_readerrorlog' tablo yapısına uygun "#LOG" adında bir geçici(TEMP) tablo oluşturduk. Error Log tablosundan kayıtları alıp geçici tablomuza ekledik.
'SELECT' sorgusu ile son 3dk içinde yapılan başarısız giriş denemelerini listeledik.
'TRUNCATE' komutu ile tablomuzdaki verileri temizledik.

## IP'ye Göre Gruplama
```sql
SELECT 
    SUBSTRING(Text, CHARINDEX('[CLIENT:', Text) + 8, LEN(Text) - CHARINDEX('[CLIENT:', Text) - 8) AS IP_Address,
    COUNT(*) AS FailedAttempts
FROM #LOG
WHERE LogDate >= DATEADD(MINUTE, -3, GETDATE())
GROUP BY 
    SUBSTRING(Text, CHARINDEX('[CLIENT:', Text) + 8, LEN(Text) - CHARINDEX('[CLIENT:', Text) - 8)
HAVING COUNT(*) > 3 -- 3’ten fazla deneme varsa
```
SUBSTRING fonksiyonu ile text içindeki 'CLIENT' bilgisinde yer alan ip adresini çekiyoruz. 'COUNT(*) AS FAILEDATTEMPS' ile aynı ip adresinden
kaç kez hatalı giriş denemesi yapılmış bunu sayıyoruz.
'GROUP BY' ile ip'leri grupluyor, 'HAVING COUNT(*) > 3' kodu ile aynı ip 3'ün üzerinde yanlış giriş yaptıysa listeliyoruz. Bu deneme sayısı eşiğini kendiniz de belirleyebilirsiniz.

### NOT
Eğer yukarıda yazılan komutları bir arada kullanmak isterseniz ilk örnekteki verileri temizleme komutunu (TRUNCATE) kod bloğunuzun en son satırına taşımayı unutmayın.