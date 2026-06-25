# 🚀 Mikroservis Öğrenme Günlüğü

> Bu dosya **dinamik bir öğrenme günlüğüdür**. Go ile mikroservis kursunu takip ederken
> öğrendiklerimi, takıldığım yerleri, sorduğum soruların cevaplarını ve karşılaştığım
> teknolojileri (PostgreSQL, MongoDB, RabbitMQ, Docker...) burada biriktiriyorum.
>
> Kurs ilerledikçe yeni bölümler eklenecek. Her zaman bu dosyayı açıp "mikroservis
> neymiş, ben nerede takılmışım, neler öğrenmişim" diye bakabilirim.

**Proje:** `go-micro` — Go ile yazılmış bir mikroservis sistemi
**Başlangıç:** Sıfır mikroservis bilgisiyle başladım
**Son güncelleme:** 2026-06-25

---

## 📑 İçindekiler

1. [Mikroservis Nedir? (Büyük Resim)](#1-mikroservis-nedir-büyük-resim)
2. [Sistem Mimarisi — Şu Anki Hali](#2-sistem-mimarisi--şu-anki-hali)
3. [Servisler Tek Tek](#3-servisler-tek-tek)
4. [Kullandığımız Teknolojiler](#4-kullandığımız-teknolojiler)
5. [Öğrenme Zaman Çizelgesi (Git Geçmişinden)](#5-öğrenme-zaman-çizelgesi-git-geçmişinden)
6. [Kavram Sözlüğü (Sorduğum Sorular + Cevapları)](#6-kavram-sözlüğü-sorduğum-sorular--cevapları)
7. [Takıldığım Hatalar ve Çözümleri](#7-takıldığım-hatalar-ve-çözümleri)
8. [Tekrar Eden Pattern'ler (Her Serviste Aynı)](#8-tekrar-eden-patternler-her-serviste-aynı)
9. [Sırada Ne Var?](#9-sırada-ne-var)

---

## 1. Mikroservis Nedir? (Büyük Resim)

**Monolitik mimari:** Her şey tek bir programın içinde. Kullanıcı arayüzü, iş mantığı,
veritabanı erişimi — hepsi aynı kod tabanında, tek process olarak çalışır.

**Mikroservis mimarisi:** Her sorumluluk **ayrı, bağımsız bir program** (servis).
Birbirlerinden bağımsız çalışırlar.

```
Monolith:                      Mikroservis:
┌──────────────┐               ┌─────────┐ ┌─────────┐ ┌─────────┐
│  Tek Program │               │ Auth    │ │ Logger  │ │ Mail    │
│  - UI        │               │ Service │ │ Service │ │ Service │
│  - Auth      │      vs       └─────────┘ └─────────┘ └─────────┘
│  - Logging   │                    ↑           ↑           ↑
│  - Mail      │                    └───────────┴───────────┘
└──────────────┘                       (birbirinden bağımsız)
```

### Neden mikroservis?

| Avantaj | Açıklama |
|---------|----------|
| **İzolasyon** | Biri çökse diğerleri çalışmaya devam eder |
| **Bağımsız ölçeklendirme** | Sadece yoğun olan servisin kopyasını artırırsın |
| **Bağımsız deploy** | Bir servisi güncellerken diğerlerine dokunmazsın |
| **Teknoloji çeşitliliği** | Her servis kendi diline/DB'sine sahip olabilir |

### Bedeli (her şey güzel değil)

- Ağ üzerinden haberleşme karmaşıklığı (servisler birbirine HTTP/mesaj ile konuşur)
- Dağıtık sistem zorlukları (bir servis hazır değilken diğeri başlayabilir)
- Daha fazla operasyon yükü (Docker, orchestration, monitoring...)

> **Kişisel not:** Bu projede her şeyi standart kütüphane + `chi` router ile sıfırdan
> yazıyoruz. Framework (Gin, Fiber, Echo) kullanmamamızın sebebi "işin arkasını
> görmek" — gerçek dünyada mikroservislerde de framework kullanılabilir.

---

## 2. Sistem Mimarisi — Şu Anki Hali

```
                                  ┌──────────────────┐
                                  │    Front-End     │  (tarayıcı arayüzü, :80)
                                  │  test sayfası    │
                                  └────────┬─────────┘
                                           │ fetch (HTTP/JSON)
                                           ▼
                                  ┌──────────────────┐
            ┌─────────────────────│  Broker Service  │  (API Gateway, :8080)
            │                     │  tek giriş kapısı│
            │                     └────────┬─────────┘
            │ SENKRON (HTTP)               │ ASENKRON (RabbitMQ)
            ▼                              ▼
   ┌──────────────┐              ┌──────────────────┐
   │     Auth     │              │     RabbitMQ     │  (mesaj kuyruğu)
   │   Service    │              │   "logs_topic"   │
   └──────┬───────┘              └────────┬─────────┘
          │                               │
          ▼                               ▼
   ┌──────────────┐              ┌──────────────────┐
   │  PostgreSQL  │              │ Listener Service │  (consumer)
   │  (users DB)  │              └────────┬─────────┘
   └──────────────┘                       │ HTTP
                                          ▼
   ┌──────────────┐              ┌──────────────────┐
   │     Mail     │              │  Logger Service  │
   │   Service    │              │   + MongoDB      │
   │  (SMTP)      │              └──────────────────┘
   └──────────────┘
```

**Her şey Docker container'ı olarak çalışıyor** — sadece kullanıcı (tarayıcı) Docker dışında.

İki farklı haberleşme şekli var:
- **Senkron:** Broker → Auth servisine direkt HTTP isteği atar, cevap bekler.
- **Asenkron:** Broker → RabbitMQ kuyruğuna mesaj bırakır, unutur. Listener kuyruktan alır.

---

## 3. Servisler Tek Tek

### 🖥️ Front-End Service
- **Görev:** Kullanıcıya web arayüzü gösterir.
- **Port:** 80
- **Teknoloji:** Go `html/template` paketi, `.gohtml` şablonları.
- **Yapı:** Sayfa üç parçadan birleşir → `base.layout` + `header.partial` + `footer.partial`.
- **Test sayfası:** `Output`, `Sent`, `Received` kutuları + test butonları (Test Broker, Test Auth).
- Butona tıklayınca JavaScript `fetch` ile broker'a POST isteği atar, gelen JSON'ı ekrana yazar.

### 🚪 Broker Service (API Gateway)
- **Görev:** Tüm mikroservislerin önünde duran **tek giriş noktası**. İstekleri doğru servise yönlendirir.
- **Port:** 8080 (dışarıdan) → 80 (container içinde)
- **Router:** `chi` + CORS + `/ping` heartbeat
- **Endpoint'ler:**
  - `POST /` → "Hit the broker" döner (basit test)
  - `POST /handle` → `HandleSubmission`: gelen `action` alanına göre `switch` ile doğru servise yönlendirir
- **`action` switch'i:** `"auth"` → authenticate(), `"log"` → logEvent(), `"mail"` → sendMail()...
- **Önemli:** Broker auth servisinin ham yanıtını iletmez — kendi temiz yanıtını oluşturup front-end'e döner.

> **Kavram düzeltmesi:** Bu projedeki "broker" aslında bir **API Gateway**. Gerçek bir
> "message broker" (RabbitMQ/Kafka) farklı bir kavramdır — ve onu da listener bölümünde ekledik.

### 🔐 Authentication Service
- **Görev:** Kullanıcı kimlik doğrulama (login).
- **Port:** 8081 → 80
- **DB:** PostgreSQL
- **Config:** `DB *sql.DB` + `Models data.Models`
- **Endpoint:** `POST /authenticate`
- **Akış:** Email+şifre al → DB'de kullanıcıyı bul (`GetByEmail`) → bcrypt ile şifre karşılaştır (`PasswordMatches`) → yanıt dön.
- **Güvenlik:**
  - Şifreler **bcrypt** ile hash'lenir (cost factor = 12).
  - `Password string \`json:"-"\`` → şifre asla JSON yanıtına dahil edilmez.
  - "Kullanıcı yok" ve "şifre yanlış" için **aynı** hata mesajı → saldırgan hangi email'lerin kayıtlı olduğunu anlayamaz.

### 📝 Logger Service
- **Görev:** Tüm servislerden gelen logları kaydeder.
- **DB:** MongoDB (NoSQL — log gibi esnek/değişken veri için ideal)
- **Portlar:** HTTP (80), RPC (5001), gRPC (50001) — 3 farklı çağrı yöntemine hazır.
- **Endpoint:** `POST /log` → `WriteLog`: gelen JSON'ı (name + data) MongoDB'ye yazar.
- **RPC:** `rpc.go` içinde `RPCServer.LogInfo(payload, *resp)` — broker RPC üzerinden log yazabilir.
- **`rpcListen()`:** `go app.rpcListen()` ile goroutine'de başlar, TCP 5001'i dinler. HTTP sunucusu (`ListenAndServe`) main'i bloklar, RPC arka planda çalışır.

### ✉️ Mail Service
- **Görev:** Email gönderir (SMTP).
- **Endpoint:** `POST /send`
- **İki ana struct:**
  - `Mail` → SMTP sunucu ayarları (host, port, kullanıcı, şifreleme), uygulama başında bir kez kurulur.
  - `Message` → her gönderimde doldurulan email paketi (from, to, subject, data).
- **Özellikler:**
  - Hem HTML hem düz metin (plain text) formatı gönderir (bazı istemciler HTML göstermez).
  - **inline CSS** (`premailer`): email istemcileri harici CSS yüklemediği için CSS her elementin `style=""` attribute'una gömülür.
  - `.gohtml` şablonları kullanır (`mail.html.gohtml`, `mail.plain.gohtml`).

> **Güvenlik notu (kurstan):** Production'da broker mail servisini **asla doğrudan**
> çağırmamalı. Bunun yerine ilgili servis (auth login başarısız olunca, vb.) kendisi mail
> servisini tetikler. Şu anki kurulum sadece development kolaylığı için.

### 👂 Listener Service
- **Görev:** RabbitMQ kuyruğunu dinler, gelen mesajları ilgili servise yönlendirir.
- **RabbitMQ açısından:** Bir **consumer**.
- **Akış:** RabbitMQ'ya bağlan → `consumer.Listen([]string{"log.INFO", "log.WARNING", "log.ERROR"})` → mesaj gelince `handlePayload` ile `switch` yapıp logger'a yönlendirir.
- **Bağlantı:** Exponential backoff ile bekler (1s, 4s, 9s... `counts²`).

---

## 4. Kullandığımız Teknolojiler

### 🐳 Docker & Docker Compose
- **Docker Image:** Programın + tüm bağımlılıklarının (OS, kütüphaneler, config) dondurulmuş paketi. "Bende çalışıyordu" sorununu çözer.
- **Container:** Image'ın çalışan hali.
- **Docker Compose:** Tüm servisleri tek `docker-compose up` komutuyla ayağa kaldırır.
- **Multi-stage build** (eski yöntem): Aşama 1'de Go ile derle, Aşama 2'de sadece binary'yi küçük alpine image'a kopyala (~300MB → ~10-15MB).
- **Bizim yöntem (yeni):** Derlemeyi Makefile'a taşıdık, Docker sadece hazır binary'yi alpine'e kopyalıyor → çok hızlı build.
- **Port mapping** `"8080:80"`: Mac'teki 8080 → container içindeki 80.
- **Volume'lar:** `./db-data/...:/var/lib/...` → container silinse bile veri kalıcı kalır.
- **`depends_on`:** Servis başlama sırasını zorlar (önce postgres, sonra auth).
- **Önemli:** Logger gibi iç servislerde `ports` yok → sadece Docker iç ağındaki servisler erişebilir, dışarıdan erişilemez (güvenlik).

### 🐘 PostgreSQL (İlişkisel / SQL)
- Veriler önceden tanımlı **tablolarda**, her satır aynı yapıda.
- Tablolar arası ilişki (JOIN), güçlü tutarlılık garantisi.
- **Ne zaman:** Yapısal veri, ilişkiler, kritik tutarlılık (banka, kullanıcı, sipariş).
- Bağlantı: **DSN string** + `pgx` driver (`database/sql` ile).
- Bu projede: **Auth service** kullanıcıları burada tutar.

### 🍃 MongoDB (Döküman tabanlı / NoSQL)
- **Şema zorunlu değil** — her kayıt farklı yapıda olabilir. JSON benzeri dökümanlar (BSON formatında saklanır).
- **Ne zaman:** Esnek/değişken yapı, log/event/analitik, çok fazla yazma.
- **Yapı eşlemesi:** `Database` ≈ PostgreSQL veritabanı, `Collection` ≈ tablo. İlk kayıtta otomatik oluşur (CREATE TABLE gerekmez).
- Bağlantı: URI + credential (`options.Client().ApplyURI(...)` → `mongo.Connect(...)`).
- Bu projede: **Logger service** logları burada tutar.

> **Polyglot Persistence:** Mikroservislerde her servis kendi ihtiyacına en uygun DB'yi
> seçer (auth→Postgres, logger→Mongo). Monolith'te de teknik olarak mümkün ama
> ekstra karmaşıklık genellikle buna değmez.

### 🐰 RabbitMQ (Mesaj Kuyruğu)
- Servisler arası mesaj taşıyan **ara katman**. Kendisi iş yapmaz — mesajı alır, saklar, dağıtır. (Kargo şirketi analojisi.)
- **Protokol:** AMQP (Advanced Message Queuing Protocol) — HTTP gibi ama mesaj kuyrukları için tasarlanmış.
- **Akış:** `Producer → Exchange → Queue → Consumer`
  - **Producer/Emitter:** Mesajı gönderen (bizim projede broker).
  - **Exchange:** "Bu mesaj hangi queue'ya gitmeli?" kararını verir. Tipleri: **Direct** (belirli kuyruk), **Fanout** (tüm kuyruklar/broadcast), **Topic** (pattern'e göre).
  - **Queue:** Mesajların beklediği kuyruk.
  - **Consumer:** Kuyruktan mesaj alan (bizim projede listener).
- **Routing key (severity):** `"log.INFO"`, `"log.ERROR"` → exchange hangi kuyruğa göndereceğine bununla karar verir.
- **Avantajları:**
  1. **Bağımsızlık:** Auth çökse bile broker mesajı kuyruğa bırakır, işlem kaybolmaz.
  2. **Yük dengeleme:** 1000 istek gelse kuyruğa yığılır, sırayla işlenir.
  3. **Fire and forget:** Broker "gönder ve unut" yapar, kullanıcıya anında cevap döner.
- **Sadece mikroservis değil:** Monolith'te de kullanılır (sipariş→fatura+kargo+email paralel işleme).
- **Neden Kafka değil:** RabbitMQ basit, olgun, kurulumu kolay, öğrenmek için ideal. Kafka milyonlarca mesaj/saniye gereken devasa sistemler için.

### 🛠️ Makefile
- Uzun komutlara kısa isimler verir. `make up_build` gibi.
- Komutlar: `up` (build'siz başlat), `up_build` (derle + build + başlat), `down` (durdur), `build_broker`/`build_auth`/`build_logger` (Linux binary derle), `start`/`stop` (front-end).
- **`GOOS=linux`:** Mac'te çalışsan bile Docker (Linux) için derle. Olmazsa binary container'da çalışmaz.

---

## 5. Öğrenme Zaman Çizelgesi (Git Geçmişinden)

> En eskiden en yeniye doğru. Her commit bir öğrenme adımı.

| # | Commit | Ne Öğrendim / Yaptım |
|---|--------|----------------------|
| 1 | `Initial commit` | İlk iki servis: front-end + broker iskeletleri. chi router, template'ler. |
| 2 | `Add .gitignore` | `.DS_Store` temizliği, build artifact'lerini ignore etme. |
| 3 | `build a docker image for the broker service` | İlk Dockerfile + docker-compose. **Docker image/container** kavramı. |
| 4 | `test the connection of fronted and broker` | Front-end JS ile broker'a ilk `fetch` isteği. Servisleri ilk kez bağladım. |
| 5 | `add some helper function to deal with json` | `readJSON`, `writeJSON`, `errorJSON` helper'ları. **Makefile** eklendi. HTTP header vs body ayrımı. |
| 6 | `set up a stub authentication service` | 3. servis: Auth iskeleti. `User` modeli, CRUD fonksiyonları, **bcrypt**. |
| 7 | `ignore build artifacts and local db data` | `.gitignore` genişletildi. |
| 8 | `connect to Postgres and add authenticate endpoint` | **PostgreSQL** bağlantısı, `connectToDB` retry döngüsü, `pgx` driver, DSN. |
| 9 | `add Dockerfile and Makefile build target` (auth) | Auth için tek-aşamalı Dockerfile, `build_auth`. |
| 10 | `add authentication and postgres services to docker-compose` | Compose'a auth + postgres, environment, volumes. |
| 11 | `forward authentication requests to auth service` | Broker `HandleSubmission` + `authenticate` → **servisler arası HTTP** (`http.Client`, `client.Do`). |
| 12 | `add Test Auth button` | Front-end'e Test Auth butonu. |
| 13 | `add the logger service` | 4. servis: Logger + **MongoDB**. `bson.D`/`bson.M`, cursor, `ObjectIDFromHex`, `context`. `serve()` + goroutine. |
| 14 | `add mail service` | 5. servis: Mail + **SMTP**. HTML/plain template, **inline CSS** (premailer). |
| 15 | `add listener service and RabbitMQ` | 6. servis: Listener + **RabbitMQ/AMQP**. Emitter (broker), Consumer (listener), exchange, channel, routing key. |

---

## 6. Kavram Sözlüğü (Sorduğum Sorular + Cevapları)

> Sohbet sırasında merak edip sorduğum şeyler. İleride "şu neydi?" deyince buraya bakacağım.

### "monolith'te de JSON kullanılıyor ama `readJSON` yazdığımı hatırlamıyorum, niye?"
Monolith'te genelde **framework** (Gin/Fiber/Echo) kullanırsın, o `c.Bind(&data)` ile JSON
okumayı senin için yapar. Bu kursta sadece standart kütüphane + chi var (chi sadece
routing yapar), o yüzden 1MB sınırı, tek-obje kontrolü, JSON yanıt formatını **kendin
yazarsın**. Framework'ün içinde de aynı kod var, sadece görmüyorsun.

### "İstek/yanıtta header kısmı + data kısmı mı var?"
Evet. **Header** = veri hakkında meta bilgi (`Content-Type`, status kodu, `Authorization`).
**Body/Data** = asıl taşınan veri (JSON içeriği). Go'da yazma sırası önemli:
`w.Header().Set(...)` → `w.WriteHeader(status)` → `w.Write(body)`. Body yazmaya başlayınca
header'lar kilitlenir.

### "payload ne demek?"
**Payload = taşınan veri paketi.** Bir istek/yanıtta asıl içerik olan kısım (= body).
Projede tutarlı olarak "şu an göndermek/almak istediğim veri" anlamında kullanılıyor.

### "Broker ne işe yarar? Mikroservislerde broker bunun için mi kullanılır?"
Bu kursta yapılan teknik olarak **API Gateway** pattern'i — tüm istekler tek noktadan
girer, yönlendirilir. Gerçek dünyada servisler şu şekillerde haberleşir:
1. **Direkt HTTP (REST)** — servisler birbirini direkt çağırır.
2. **API Gateway** — Nginx, Kong, AWS API Gateway (kursta elle yazdık).
3. **Message Broker (gerçek broker)** — RabbitMQ, Kafka — kuyruk üzerinden asenkron.
4. **gRPC** — HTTP'den hızlı binary protokol, servis-servis iletişiminde çok yaygın.

### "`client := &http.Client{}` / `client.Do()` nedir?"
`http.Client` Go'nun HTTP isteği gönderme yapısı — bir **tarayıcı** gibi düşün.
`http.NewRequest(...)` mektubu yazmak, `client.Do(request)` ise postaneye götürüp göndermek.
Asıl network bağlantısı `Do`'da kurulur. Boş `&http.Client{}` varsayılan ayarlarla çalışır
(timeout yok — production'da `Timeout` eklenmeli).

### "mongo.Client de var, http.Client de — aynı mı?"
Hayır, isim aynı ama farklı kütüphaneler. `http.Client` (`net/http`) → HTTP isteği atmak için.
`mongo.Client` (`mongo-driver`) → MongoDB'ye bağlanmak için. Konsept benzer (bir şeye
bağlan/istek at) ama tamamen farklı API'ler.

### "context nedir?"
Bir işleme verilen **"kimlik kartı + zaman sınırı + iptal butonu"**. Bir HTTP isteği
geldiğinde o istek için otomatik bir context oluşturulur. Bu context:
- Ne kadar süre bekleyeceğini bilir (timeout)
- İptal edilip edilmediğini bilir (cancel)
- İstek boyunca taşınan değerleri tutar (request ID, user ID vs.)

Kullanıcı isteği yarıda iptal ederse (tarayıcıyı kapattı), context iptal sinyali yayar,
devam eden DB sorgusu da iptal olur. Boşa kaynak harcanmaz.

Context zincirleme çalışır:
```
HTTP isteği → context oluşur
    ↓
Handler context'i alır
    ↓
DB sorgusuna geçer
    ↓
İstek iptal olursa → DB sorgusu da iptal olur
```

### "`context.WithTimeout(context.Background(), 15*time.Second)` nedir?"
**context** = bir işlemin ne kadar süre çalışacağını kontrol eder. `Background()` boş
başlangıç context'i. `WithTimeout(...)` → "bu işlem en fazla 15 saniye sürebilir". İki şey
döner: `ctx` (izin kağıdı) + `cancel` (erken iptal fonksiyonu). `defer cancel()` → işlem
bitince timer'ı temizler (bellek sızıntısını önler).

### "`context.TODO()` nedir?"
`context.Background()` ile çalışma zamanında **aynı**. Fark sadece niyet:
`Background()` = "kasıtlı boş context". `TODO()` = "buraya sonra gerçek context koyacağım"
(teknik borç işareti).

### "Context normalde nasıl doldurulur?"
Üç yol var:

**1. HTTP isteğinden al (en doğal yol):**
```go
func (app *Config) WriteLog(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    collection.InsertOne(ctx, LogEntry{...})
}
```

**2. Elle timeout ile oluştur:**
```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
collection.InsertOne(ctx, LogEntry{...})
```

**3. HTTP context + timeout — en iyi yol:**
```go
ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
defer cancel()
collection.InsertOne(ctx, LogEntry{...})
```
Hem HTTP isteğine bağlı (istek iptal olursa DB işlemi de durur) hem de timeout var.

### "MongoDB'de `collection := client.Database("logs").Collection("logs")` ne yapıyor?"
MongoDB yapısı: `Client` (≈ sunucu) → `Database("logs")` (≈ veritabanı) →
`Collection("logs")` (≈ tablo). PostgreSQL'de tabloyu `CREATE TABLE` ile önceden açman
gerekir; MongoDB'de ilk kayıtta otomatik oluşur. `InsertOne` ≈ SQL'deki `INSERT INTO`,
ama SQL yazmak yerine Go struct'ı gönderiyorsun.

### "`bson.D` ve `bson.M` nedir?"
MongoDB JSON yerine **BSON** (Binary JSON) kullanır.
- `bson.D{{"created_at", -1}}` → **sıralı** döküman, sıra önemliyse (sort gibi). `-1` = azalan, `1` = artan.
- `bson.M{"_id": docID}` → **sırasız map**, basit filtreler için. (`WHERE _id = docID` gibi.)
- `bson.D{}` → boş filtre = hepsini getir (WHERE yok).

### "`primitive.ObjectIDFromHex` nedir?"
MongoDB her kayda otomatik bir **ObjectID** verir (hex string: `"507f1f77bcf86cd799439011"`).
İçeride binary saklanır. `ObjectIDFromHex` bu hex string'i MongoDB'nin anlayacağı binary
tipe çevirir. (PostgreSQL'de ID sadece int — dönüştürme gerekmez.)

### "cursor nedir? `opts` ne işe yarar?"
- **cursor** = iterator, kayıtlar üzerinde tek tek gezinir. PostgreSQL'deki `rows.Next()` ile aynı. `cursor.Decode(&item)` ≈ `rows.Scan(...)`. `defer cursor.Close(ctx)` ile kaynak serbest bırakılır.
- **opts** (`options.Find()`) = sorgu seçenekleri. `opts.SetSort(bson.D{{"created_at", -1}})` = sıralama (`ORDER BY created_at DESC`).

### "`var client *mongo.Client` + `New(...)` pattern'i ne yapıyor?"
Auth'taki `New(dbPool *sql.DB)` ile **birebir aynı pattern** (Postgres yerine Mongo).
`main.go`'da bağlantı kurulur → `data.New(client)` ile bu bağlantı paket-seviyesi global
değişkene kaydedilir → artık `Insert`, `GetAll` gibi fonksiyonlar bu global `client`
üzerinden DB'ye erişir.

### "PostgreSQL vs MongoDB — farkı ne, ne zaman hangisi?"
| Durum | Tercih |
|-------|--------|
| Veriler arası ilişki var | PostgreSQL |
| Şema sabit | PostgreSQL |
| Tutarlılık kritik (para, sipariş) | PostgreSQL |
| Esnek/değişken yapı | MongoDB |
| Çok fazla yazma | MongoDB |
| Log, event, analitik | MongoDB |

Kısaca: Postgres "veri nasıl görünmeli" konusunda katı ama güvenli; MongoDB "ne gelirse
kaydet" konusunda esnek ama daha az garantili. Rakip değiller, farklı problemler için
doğru araçlar.

### "RabbitMQ nedir, neden bu iş için onu kullanıyoruz?"
Mesaj kuyruğu sunucusu — servisler arası mesaj taşıyan ara katman (kargo şirketi gibi).
AMQP protokolü kullanır (mesaj kuyrukları için tasarlanmış, HTTP'den hafif). Exchange+Queue
sistemi mesaj garantisi sağlar (listener çökse mesaj kuyrukta bekler). Kafka'ya göre basit
ve olgun → öğrenmek için ideal.

### "channel ve exchange nedir? (AMQP)"
- **Channel (AMQP):** TCP Connection kurmak pahalı. Channel bu bağlantı üzerinde açılan **hafif sanal yol**. Birden çok channel aynı connection'ı paylaşır. Her işlem (publish/consume) bir channel üzerinden yapılır.
- **Exchange:** Producer mesajı direkt kuyruğa göndermez — önce exchange'e gönderir. Exchange "bu mesaj nereye gitmeli?" kararını verir (Direct/Fanout/Topic).

### "Buradaki channel'lar Go channel mı, AMQP channel mı?"
**AMQP channel** — Go channel (`chan`) ile alakasız, sadece isim aynı.
| | Go Channel | AMQP Channel |
|--|-----------|--------------|
| Nerede | Program içi | Network üzerinde |
| Ne taşır | Goroutine → Goroutine | Servis → RabbitMQ |
| Paket | Built-in (`chan`) | `amqp091-go` |

### "Queue'lar eş zamanlı mı çalışır?"
Evet. Her queue ayrı goroutine ile dinlenir, hepsi aynı anda çalışır. Ayrıca **bir queue'yu
birden çok consumer** da tüketebilir (yük dengeleme) — RabbitMQ mesajları aralarında
dağıtır, aynı mesajı iki listener almaz.

### "consumer ile listener arasındaki ilişki ne?"
Aynı şeyin farklı isimleri. **Listener** = mimari terim ("dinleyen servis"). **Consumer** =
RabbitMQ terminolojisi ("kuyruktan mesaj alan taraf"). `listener-service`, RabbitMQ açısından
bir consumer'dır.

### "Emitter nedir? event.go neden iki serviste de var?"
- **Emitter** = "yayıncı", RabbitMQ'ya mesaj gönderen taraf (broker). `emitter.Push(event, severity)` → exchange'e publish eder. `severity` = routing key (`"log.INFO"`, `"log.ERROR"`).
- **event.go** (`declareExchange`, `declareRandomQueue`) hem broker hem listener'da var çünkü exchange tanımı **birebir aynı** olmalı (yoksa RabbitMQ hata verir). İdeal çözüm ortak bir shared paket olurdu ama Go'da servisler arası kod paylaşımı karmaşık → kopyalandı.

> ⚠️ **Not:** `consumer.go` broker'a **yanlışlıkla** kopyalanmış — broker sadece emitter
> (gönderen), consumer'a ihtiyacı yok. Broker'daki `consumer.go` hiçbir yerde kullanılmıyor,
> silinebilir.

### "Local'de çalıştırmak vs Docker'da çalıştırmak farkı ne?"
Docker'da servisler birbirini **isimle** bulur (Docker iç DNS'i): `mongodb://mongo:27017`,
`host=postgres`. Local'de Docker ağı yok, `localhost` gerekir: `mongodb://localhost:27017`.
Ayrıca local'de port 80 Docker tarafından tutulduğu için test ederken farklı port (8082,
8083) kullandık. **Docker'a deploy ederken bunlar eski haline dönmeli.**

### "`serve()` ayrı yapılınca program durdu, sürekli çalışması gerekmiyor mu?"
`go app.serve()` goroutine'i arka planda başlatır ama `main()` hemen biter → main bitince
tüm goroutine'ler ölür. İki çözüm: (1) `select {}` ile main'i sonsuza kadar beklet, ya da
(2) `serve()`'ün gövdesini main'e taşı → `ListenAndServe` zaten bloklayıcı, main canlı kalır.
(Biz 2. yolu seçtik.)

### "Makefile'daki build sırası fark eder mi?"
Hayır. `build_broker`, `build_auth`, `build_logger` birbirinden bağımsız, paralel bile
çalışabilir. Sıra **Docker Compose'da** önemli → `depends_on` ile (önce postgres, sonra auth).

### "`make up` ile `make up_build` farkı ne? Container/image silinirse?"
- `make up` = mevcut image'ları olduğu gibi başlatır (hızlı, kod değişmediyse).
- `make up_build` = önce binary derler, Docker image'larını yeniden build edip başlatır (kod değiştiyse).
- Image var + container yok → `make up` yeter (container'ı yeniden oluşturur).
- Image de silindiyse → `make up_build` gerekir.

### "RPC nedir? HTTP'den farkı ne?"
**RPC (Remote Procedure Call)** = uzaktaki bir fonksiyonu sanki local'deymiş gibi çağırmak.
HTTP'de "şu endpoint'e şu JSON'ı gönder" dersin. RPC'de ise "şu fonksiyonu şu argümanlarla
çağır" dersin — network üzerinde olduğunu bile hissetmezsin.

```
HTTP:  POST /log  { "name": "event", "data": "..." }
RPC:   LogInfo(payload)  ← direkt fonksiyon çağrısı gibi
```

Go'nun `net/rpc` paketi şu imzayı zorunlu kılar:
```go
func (t *T) MethodName(args ArgsType, reply *ReplyType) error
// args   → gelen veri
// reply  → pointer ile dönen yanıt
// error  → hata varsa
```

| | HTTP | RPC |
|---|---|---|
| Protokol | HTTP/JSON | TCP/binary |
| Kullanım | Dışarıdan (tarayıcı, broker) | Servisler arası (iç ağ) |
| Hız | Biraz daha yavaş | Daha hızlı |
| Port (logger) | 80 | 5001 |

### "rpcListen nasıl çalışıyor? Neden goroutine?"
```go
rpc.Register(new(RPCServer))  // "bu struct'ın metodları RPC ile çağrılabilir"
go app.rpcListen()            // goroutine'de başlat
srv.ListenAndServe()          // main'i bloklar — RPC arka planda çalışır
```

`rpcListen` içinde sonsuz döngü:
```go
listen, _ := net.Listen("tcp", "0.0.0.0:5001")
for {
    rpcConn, _ := listen.Accept()   // bağlantı gelince al
    go rpc.ServeConn(rpcConn)       // her bağlantı için ayrı goroutine
}
```
`go app.rpcListen()` goroutine olarak başlatılır çünkü `ListenAndServe` (HTTP) main'i
bloklar. Böylece logger servisi aynı anda **hem HTTP (80) hem RPC (5001)** dinler.

### "Monolith'te de iki farklı DB (Postgres + Mongo) kullanılır mı?"
Teknik olarak mümkün ama pratikte çok nadir — iki bağlantı havuzu, iki migration sistemi,
transaction karmaşıklığı. Mikroservislerde **doğal** çünkü her servis kendi DB'sine sahip,
biri diğerini etkilemez (Polyglot Persistence).

---

## 7. Takıldığım Hatalar ve Çözümleri

| Hata | Sebep | Çözüm |
|------|-------|-------|
| `TypeError: Failed to fetch` | Front-end `https://localhost:8080` kullanıyordu ama broker düz HTTP (SSL yok) | `https` → `http` |
| `cd: ../broker-service: No such file` (make) | Makefile `project/` klasöründen çalışacak şekilde yazılmış ama kök dizindeyiz | `cd ../broker-service` → `cd broker-service`, `docker-compose` → `docker-compose -f project/docker-compose.yml` |
| `Bind for 0.0.0.0:5432 failed: port is already allocated` | Mac'te yerel PostgreSQL + eski `quiz-postgres` container 5432'yi tutuyordu | `brew services stop postgresql` + eski container'ı durdur |
| `SyntaxError: Unexpected non-whitespace character after JSON` | Broker `/handle` endpoint'i hiç eklenmemişti → 404 HTML dönüyordu, JSON parse patlıyordu | `routes.go`'ya `mux.Post("/handle", app.HandleSubmission)` eklendi |
| `listen tcp :80: bind: address already in use` (logger/front-end) | Local'de port 80 Docker tarafından tutuluyordu | Local test için farklı port (8082 logger, 8083 front-end) |
| `"/templates": not found` (mail Dockerfile) | `templates/` klasörü hiç oluşturulmamıştı, mailer.go `./templates/...` arıyordu | Template dosyaları oluşturuldu (`mail.html.gohtml`, `mail.plain.gohtml`) |

---

## 8. Tekrar Eden Pattern'ler (Her Serviste Aynı)

Mikroservislerin güzelliği: bir kez öğrendiğin pattern her serviste tekrar ediyor.

### Servis iskeleti (her serviste)
```
servis-adı/
├── cmd/api/
│   ├── main.go      ← Config struct, DB bağlantısı, server başlatma
│   ├── routes.go    ← chi router, endpoint tanımları
│   ├── handlers.go  ← iş mantığı (handler fonksiyonları)
│   └── helpers.go   ← readJSON / writeJSON / errorJSON
├── data/models.go   ← (DB kullanan servislerde) veri modelleri
├── go.mod
└── servis.dockerfile
```

### Config struct pattern
```go
type Config struct {
    // servise göre değişir: DB *sql.DB, Models data.Models, Rabbit *amqp.Connection...
}
```
Tüm bağımlılıklar (DB, client'lar) burada toplanır, handler'lar `app.X` ile erişir.

### DB bağlantı retry döngüsü
DB/RabbitMQ container'lar aynı anda başlar ama hazır olması zaman alır. O yüzden:
- **Auth (Postgres):** sabit 2sn bekle, 10 deneme.
- **Listener (RabbitMQ):** exponential backoff (`counts²` saniye), 5 deneme.

### `New()` ile model paketi başlatma
`main.go` bağlantıyı kurar → `data.New(connection)` paket-global değişkene yazar →
model fonksiyonları o global bağlantıyı kullanır. (Hem Postgres hem Mongo'da aynı.)

### JSON helper'ları
`readJSON` (1MB sınır + tek-obje kontrolü), `writeJSON`, `errorJSON` — her serviste
kopyalanıyor (Go'da kolay kod paylaşımı yok).

---

## 9. Sırada Ne Var?

Kursun gidişatına göre muhtemel sonraki adımlar:

- [ ] Listener'da `auth` case'ini doldurup RabbitMQ üzerinden authentication
- [ ] gRPC ile servis-servis iletişimi (logger zaten gRPC portu tanımlamış)
- [x] RPC (logger'daki `rpcPort = 5001`) — `rpc.go` + `rpcListen()` tamamlandı
- [ ] Servislerin daha fazla asenkron mesajlaşmaya geçmesi
- [ ] Production güvenliği: mail servisini iç ağa kapatma, broker'ın doğrudan mail çağırmaması
- [ ] Deployment / orchestration (Docker Swarm veya Kubernetes?)
- [ ] Monitoring / tracing

---

> 📝 **Bu dosya nasıl güncellenecek:** Kursta yeni bir bölüm bittiğinde / yeni kod
> eklediğimde, ilgili bölümlere (servisler, teknolojiler, zaman çizelgesi, kavram sözlüğü)
> yeni satırlar ekleyeceğim. Sorduğum her yeni soru "Kavram Sözlüğü"ne, her yeni hata
> "Takıldığım Hatalar"a girecek.
