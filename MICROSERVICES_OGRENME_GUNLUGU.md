# 🚀 Mikroservis Öğrenme Günlüğü

> Bu dosya **dinamik bir öğrenme günlüğüdür**. Go ile mikroservis kursunu takip ederken
> öğrendiklerimi, takıldığım yerleri, sorduğum soruların cevaplarını ve karşılaştığım
> teknolojileri (PostgreSQL, MongoDB, RabbitMQ, Docker...) burada biriktiriyorum.
>
> Kurs ilerledikçe yeni bölümler eklenecek. Her zaman bu dosyayı açıp "mikroservis
> neymiş, ben nerede takılmışım, neler öğrenmişim" diye bakabilirim.

**Proje:** `go-micro` — Go ile yazılmış bir mikroservis sistemi
**Başlangıç:** Sıfır mikroservis bilgisiyle başladım
**Son güncelleme:** 2026-06-26 (Docker Swarm ile deploy eklendi)

---

## 📑 İçindekiler

1. [Mikroservis Nedir? (Büyük Resim)](#1-mikroservis-nedir-büyük-resim)
2. [Sistem Mimarisi — Şu Anki Hali](#2-sistem-mimarisi--şu-anki-hali)
3. [Servisler Tek Tek](#3-servisler-tek-tek)
4. [gRPC ile Servis İletişimi (Detay)](#35-grpc-ile-servis-iletişimi-detay)
5. [Docker Swarm ile Deployment (Detay)](#36-docker-swarm-ile-deployment-detay)
6. [Kullandığımız Teknolojiler](#4-kullandığımız-teknolojiler)
7. [Öğrenme Zaman Çizelgesi (Git Geçmişinden)](#5-öğrenme-zaman-çizelgesi-git-geçmişinden)
8. [Kavram Sözlüğü (Sorduğum Sorular + Cevapları)](#6-kavram-sözlüğü-sorduğum-sorular--cevapları)
9. [Takıldığım Hatalar ve Çözümleri](#7-takıldığım-hatalar-ve-çözümleri)
10. [Tekrar Eden Pattern'ler (Her Serviste Aynı)](#8-tekrar-eden-patternler-her-serviste-aynı)
11. [Sırada Ne Var?](#9-sırada-ne-var)

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

Logger servisine log yazmanın artık **dört yolu** var (hepsi aynı MongoDB kaydına çıkar):
HTTP (`/log`, :80), RabbitMQ (listener üzerinden), RPC (:5001), **gRPC (:50001)**. Bu kasıtlı —
kurs aynı işi farklı protokollerle göstererek hangisinin ne zaman uygun olduğunu öğretiyor.

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
- **gRPC:** `grpc.go` içinde `LogServer.WriteLog(...)` — broker gRPC üzerinden de log yazabilir. `go app.gRPCListen()` ile TCP 50001'de dinler. → **Logger artık aynı anda 3 protokol dinliyor: HTTP (80) + RPC (5001) + gRPC (50001).** Üçü de sonunda aynı `LogEntry.Insert()` ile MongoDB'ye yazıyor; sadece "kapıdan giriş şekli" farklı.

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

## 3.5. gRPC ile Servis İletişimi (Detay)

> Commit: `add communication between services using gRPC` (2026-06-25). Logger servisine
> **4. çağrı yöntemi** olarak gRPC eklendi (HTTP, RPC'den sonra). Broker → Logger arasında
> binary, hızlı, dil-bağımsız bir iletişim kuruldu.

### gRPC nedir? (net/rpc'den farkı)

Günlükte zaten `net/rpc` (Go'nun yerleşik RPC'si) vardı. **gRPC**, onun "büyümüş,
endüstri standardı" hali — Google'ın geliştirdiği, dil-bağımsız, yüksek performanslı bir
RPC framework'ü.

```
HTTP/JSON:   POST /log  { "name": "...", "data": "..." }   ← metin, yavaş, dışarıya açık
net/rpc:     LogInfo(payload)                              ← Go-only, binary, TCP
gRPC:        c.WriteLog(ctx, &LogRequest{...})             ← dil-bağımsız, binary, HTTP/2
```

| | HTTP/JSON | net/rpc | gRPC |
|---|-----------|---------|------|
| Veri formatı | JSON (metin) | Go gob (binary) | Protobuf (binary) |
| Taşıma | HTTP/1.1 | TCP | **HTTP/2** (multiplexing, streaming) |
| Dil bağımsız | ✅ | ❌ (sadece Go) | ✅ (her dil) |
| Hız | Yavaş | Hızlı | En hızlı |
| Sözleşme | Yok (elle) | Yok (Go struct) | `.proto` dosyası |
| Port (logger) | 80 | 5001 | 50001 |

### `.proto` — sözleşme (contract)

Her şeyin kaynağı `logs.proto` dosyası. Servisin ne kabul edip ne döndüğünü **bir kez**
burada tanımlarsın, sonra `protoc` her dil için kod üretir:

```proto
message Log         { string name = 1; string data = 2; }
message LogRequest  { Log LogEntry = 1; }
message LogResponse { string result = 1; }

service LogService {
    rpc WriteLog(LogRequest) returns (LogResponse);   // ← uzaktan çağrılacak fonksiyon
}
```

`= 1`, `= 2` → **field number'lar** (alan sırası değil, binary'de hangi alan olduğunu
işaretleyen etiket). Bir kez verdiysen değiştirme — eski veriyi bozar.

### `protoc` ile kod üretimi

```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       logs.proto
```

İki dosya üretir:
- **`logs.pb.go`** → mesaj tipleri (`Log`, `LogRequest`, `LogResponse` struct'ları + getter'lar).
- **`logs_grpc.pb.go`** → gRPC client/server iskeleti: `LogServiceClient`, `LogServiceServer`,
  `RegisterLogServiceServer`, `NewLogServiceClient`, `UnimplementedLogServiceServer`.

> Bu dosyalara **elle dokunulmaz** — `.proto` değişince yeniden üretilir. `protoc` tek başına
> Go üretmez; `protoc-gen-go` ve `protoc-gen-go-grpc` plugin'leri gerekir (`go install` ile kurulur,
> `$GOPATH/bin` PATH'te olmalı).

### Server tarafı — Logger (`grpc.go`)

Logger gRPC'de **server** (çağrıyı karşılayan):

```go
type LogServer struct {
    logs.UnimplementedLogServiceServer   // ← üretilen base, zorunlu (ileri uyumluluk için)
    Models data.Models
}

func (l *LogServer) WriteLog(ctx context.Context, req *logs.LogRequest) (*logs.LogResponse, error) {
    input := req.GetLogEntry()
    logEntry := data.LogEntry{Name: input.Name, Data: input.Data}
    err := l.Models.LogEntry.Insert(logEntry)   // ← RPC ve HTTP ile AYNI iş: MongoDB'ye yaz
    ...
    return &logs.LogResponse{Result: "logged!"}, nil
}

func (app *Config) gRPCListen() {
    lis, _ := net.Listen("tcp", fmt.Sprintf(":%s", gRpcPort))   // 50001
    s := grpc.NewServer()
    logs.RegisterLogServiceServer(s, &LogServer{Models: app.Models})
    s.Serve(lis)   // bloklayıcı → o yüzden main'de "go app.gRPCListen()" goroutine ile
}
```

`main.go`: `go app.gRPCListen()` → RPC (`go app.rpcListen()`) gibi goroutine'de başlar,
HTTP sunucusu main'i bloklar. **3 dinleyici aynı anda canlı.**

### Client tarafı — Broker (`handlers.go` → `LogViaGRPC`)

Broker gRPC'de **client** (çağrıyı yapan):

```go
func (app *Config) LogViaGRPC(w http.ResponseWriter, r *http.Request) {
    // 1. Bağlan (insecure = TLS yok, iç ağda güvenli)
    conn, _ := grpc.Dial("logger-service:50001",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock())
    defer conn.Close()

    // 2. Client oluştur (üretilen koddan)
    c := logs.NewLogServiceClient(conn)

    // 3. Timeout'lu context
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()

    // 4. Uzaktaki fonksiyonu LOCAL'deymiş gibi çağır ← RPC'nin özü budur
    _, err = c.WriteLog(ctx, &logs.LogRequest{
        LogEntry: &logs.Log{Name: ..., Data: ...},
    })
}
```

`routes.go`: `mux.Post("/log-grpc", app.LogViaGRPC)` → front-end test sayfası bu endpoint'i tetikler.

### Akış (uçtan uca)

```
Front-end (test sayfası)
    │  POST /log-grpc  (HTTP/JSON)
    ▼
Broker  ── LogViaGRPC ──→  grpc.Dial("logger-service:50001")
    │                          c.WriteLog(ctx, LogRequest{...})   ← gRPC (HTTP/2, binary)
    ▼
Logger  ── WriteLog ──→  Models.LogEntry.Insert()
    │
    ▼
MongoDB  (log kaydedildi)
```

> **Aynı `logs.proto` hem broker'da hem logger'da var** — RabbitMQ'daki `event.go` gibi.
> İki taraf da **birebir aynı sözleşmeyi** kullanmalı, yoksa binary uyuşmaz. Üretilen
> `logs.pb.go`/`logs_grpc.pb.go` her iki serviste de mevcut.

---

## 3.6. Docker Swarm ile Deployment (Detay)

> 2026-06-26. Şimdiye kadar her şeyi `docker-compose` ile **tek makinede** çalıştırıyorduk.
> Artık servisleri gerçek bir şekilde **deploy** etme aşamasına geçtik: image'ları Docker
> Hub'a push'ladık ve `docker stack deploy` ile Swarm kümesine dağıttık. Dosya: `project/swarm.yml`.

> 🧭 **ÖNEMLİ ÇERÇEVE — buradan sonrası "mikroservis" değil, "deployment/DevOps":**
> Bu bölümdeki her şey (Docker Hub, Swarm, node/manager/worker, reverse proxy, SSL, **server
> kiralama**) artık **bu projeye özel değil** — Docker'a koyabildiğin **herhangi bir uygulamayı**
> (mikroservis, monolith, tek dosyalık script, WordPress...) yayına almak için **aynıdır**.
> Yani bu kısımda öğrendiğin bilgi **transfer edilebilir**: yarın bambaşka bir proje yapsan
> bu adımları aynen kullanırsın. Mikroservis mimarisi (servis bölme, broker, gRPC, RabbitMQ...)
> büyük ölçüde bitti; şu an öğrendiğimiz **deployment**.
>
> ```
> 1. Uygulamanı yaz            → dile/mimariye özel (mikroservis mi monolith mi)
> 2. Docker image yap          → genel
> 3. Docker Hub'a push          → genel
> 4. Sunucu kirala (Linode...)  → genel   ← Linode adımı buraya düşer
> 5. Sunucuda Docker+Swarm kur  → genel
> 6. stack deploy               → genel
> 7. reverse proxy + domain + SSL → genel
> ```
> Adım 1 hariç hepsi uygulamadan bağımsız. Swarm = "kendi sunucunu yönet" tarafındaki **en
> basit orkestrasyon**; alternatifleri: Kubernetes (büyük ölçek), PaaS (Render/Railway/Fly.io
> — "sunucuyla uğraşma"), managed containers (Fargate/Cloud Run). Kavramlar (replica,
> self-healing, proxy) bir kez öğrenilince hepsine transfer olur.

### Docker Hub nedir?

Docker image'larının saklandığı **bulut tabanlı kayıt deposu (registry)**. GitHub'ın kod için
olduğu şey, Docker Hub'ın image'lar için olduğu şeydir.

- `docker build` ile lokal makinende bir image üretirsin.
- `docker push omerkoc0/broker-service:1.0.0` → o image lokalden çıkıp **Docker Hub sunucularına** yüklenir.
- İsimlendirme: `omerkoc0` = Docker Hub kullanıcı adı, `broker-service` = image adı, `1.0.0` = **tag** (versiyon).

> **"Image'lar nereye taşındı?"** → Lokal makinenden Docker Hub'a (internetteki merkezi depoya).

### Neden image'ları Docker Hub'a push'luyoruz?

Çünkü Swarm birden fazla makineye (node) dağıtım yapabilir. Bir image'ı çalıştıracak **her
makinenin** o image'a erişebilmesi gerekir. Lokal makinendeki image'ı diğer makineler göremez
— ama herkes Docker Hub'a erişebilir.

```
Senin makinen                Docker Hub                 Swarm node'ları
─────────────                ──────────                 ───────────────
docker build      ──push──►  omerkoc0/broker  ──pull──► node1, node2, node3...
                             omerkoc0/listener           (image'ı buradan çeker)
                             omerkoc0/auth...
```

`swarm.yml` içindeki `image: omerkoc0/broker-service:1.0.0` satırı tam bunu der: "Bu servisi
çalıştırırken Docker Hub'daki bu image'ı çek ve kullan."

> **Dikkat:** `rabbitmq`, `mongo`, `postgres`, `mailhog` bizim image'larımız değil — bunlar
> zaten Docker Hub'da herkese açık **resmi (official)** image'lar, onları push'lamaya gerek yok.

### Docker Swarm nedir, ne işe yarar?

Docker'ın yerleşik **container orchestration (orkestrasyon)** aracı. Orkestrasyon = birden fazla
makineyi tek mantıksal küme gibi yönetip container'ları bu makinelere otomatik dağıtmak.

`docker-compose` **tek makinede** çalışır. Swarm bunun üstüne şunları ekler:

| Özellik | Açıklama |
|---------|----------|
| **Multi-node** | Birkaç sunucuyu birleştirip tek havuz gibi kullanırsın; Swarm dağıtımı kendi yapar |
| **Replica yönetimi** | `replicas: 3` dersen 3 kopya ayağa kalkar, yük aralarında dağılır (ölçekleme) |
| **Self-healing** | Bir container çökerse Swarm fark eder ve istenen replica sayısını korumak için otomatik yenisini başlatır |
| **Load balancing** | Aynı servisin çok replica'sı varsa gelen istekleri Swarm otomatik paylaştırır |

### `replicated` vs `global` mode

`swarm.yml`'de iki tür deploy modu var:

```yaml
deploy:
  mode: replicated     # broker, auth, logger, mailer, postgres...
  replicas: 1          # toplam kaç kopya çalışacak
```
- **`replicated` + `replicas: N`** → kümede toplam N kopya çalışır.
- **`global`** → kümedeki **her node'da birer tane** çalışır (`rabbitmq`, `mailhog`, `mongo` böyle).

### Node, Manager, Worker

Swarm'ın temel üç kavramı:

```
        ┌─────────────── SWARM KÜMESİ ───────────────┐
        │   [Manager Node]   ◄──── komuta merkezi     │
        │        │                                     │
        │        ├──────────┬──────────┐              │
        │        ▼          ▼          ▼              │
        │   [Worker 1]  [Worker 2]  [Worker 3]        │
        │   container'lar burada çalışır               │
        └─────────────────────────────────────────────┘
```

- **Node** = Swarm kümesine katılmış bir makine (sunucu, VM ya da senin laptop'un). Manager ya da worker olur.
- **Manager** = kümenin **beyni**. Küme durumunu tutar, orkestrasyon kararlarını verir (kim nerede çalışacak, çökeni kim başlatacak), **yönetim komutlarını yalnızca o kabul eder** (`docker service`, `docker stack deploy`). İstersen container da çalıştırır.
- **Worker** = işi yapan kas. Sadece manager'ın atadığı container'ları (task) çalıştırır, karar vermez, yönetim komutu çalıştıramaz.

> **Tek sayı manager (3, 5...):** Production'da manager'lar arasında **Raft** consensus
> (oylama) çalışır; kararların geçerli olması için çoğunluğun (quorum) hayatta olması gerekir.
> O yüzden tek sayı seçilir (1 çökse diğer 2 devam eder). Lokal/öğrenme için **tek manager yeter**.

> **Bizim durumumuz (tek makine):** Laptop'ta `docker swarm init` deyince tek node olur ve o
> node hem manager hem worker rolünü üstlenir. Ayrı worker makineler kurmaya gerek yok — tüm
> `replicas` ve `global` servisler bu tek node üzerinde çalışır.

### Deploy akışı (komutlar)

```bash
docker swarm init                              # 1. Bu makineyi manager yap, kümeyi başlat
docker stack deploy -c swarm.yml swarm         # 2. swarm.yml'i kümeye dağıt ("swarm" = stack adı)
docker stack services swarm                     # servisleri ve replica durumunu gör
docker stack rm swarm                           # stack'i kaldır
```

Başka bir makineyi worker olarak katmak için manager'ın verdiği token kullanılır:
```bash
docker swarm join --token <token> <manager-ip>:2377
```

### Deploy edilmiş servisi güncelleme (rolling update, sıfır kesinti)

> **Önemli ayrım:** "scale etmek" (kopya sayısını artırmak) ile "update etmek" (yeni versiyona
> geçmek) **ayrı işlemlerdir**. Scale, update'in parçası değil — update'in **kesintisiz** olmasını
> sağlayan ön koşuldur.

**1. Yeni versiyonu build + yeni tag + push:**
```bash
docker build -t omerkoc0/broker-service:1.0.1 .
docker push omerkoc0/broker-service:1.0.1
```
**Yeni tag şart** (`1.0.0` → `1.0.1`): Swarm "image değişti mi?" kararını tag'e bakarak verir.
Aynı tag'i ezmek güvenilir değil — Swarm aynı tag'i çoğu zaman yeniden çekmez.

**2. Downtime'ı önleyen asıl şey → birden fazla replica:**
Tek replica (`replicas: 1`) varsa, update sırasında o tek container durup yenisi açılana kadar
kısa bir **downtime** olur (servisi karşılayan başka kopya yok). O yüzden önce **scale** edilir:
```bash
docker service scale swarm_broker-service=3
```
```
replicas: 1 →  [v1] durdur ─► boşluk (DOWNTIME!) ─► [v2] başlat

replicas: 3 →  [v1][v1][v1]   Swarm teker teker günceller, diğerleri ayakta:
               [v2][v1][v1]   ← kullanıcı hep cevap alır
               [v2][v2][v1]
               [v2][v2][v2]   ← bitti, KESİNTİ YOK
```

**3. Rolling update'i tetikle:**
```bash
docker service update --image omerkoc0/broker-service:1.0.1 swarm_broker-service
```
Swarm replica'ları **birer birer** yeni image'la değiştirir. Davranış `swarm.yml`'den ayarlanır:
```yaml
deploy:
  replicas: 3
  update_config:
    parallelism: 1          # aynı anda kaç replica güncellensin (1 = teker teker)
    delay: 10s              # her biri arasında bekle
    order: start-first      # önce yeniyi başlat (sağlıklı olunca) sonra eskiyi durdur → boşluk olmaz
    failure_action: rollback # yeni versiyon patlarsa otomatik eski haline dön (güvenlik ağı)
```

**Doğru sıralama özeti:**
```
1. build + yeni tag (1.0.1) + push       ← yeni versiyon Docker Hub'da
2. replicas > 1 olsun (gerekirse scale)  ← downtime'ı önlemek için
3. docker service update --image ...1.0.1 ← rolling update, teker teker değişir
4. başarısızsa otomatik rollback          ← güvenlik ağı
```

> **Kafa karışıklığı düzeltmesi:** "Önce scale → sonra değiştir" sezgisi doğru ama scale,
> update için *zorunlu* değil — sadece update'i **kesintisiz** yapar. Zaten `replicas` 1'den
> fazlaysa ayrıca scale etmene gerek yok, doğrudan `service update` yeterli.

### Reverse Proxy — servislerin önündeki tek kapı (sıradaki adım)

> Front-end'i Swarm'a ekledikten sonra hocanın işaret ettiği problem: Swarm'ı ayağa
> kaldırmak kolay ama front-end'e **düzgün erişmenin bir yolu yok**. Çözüm: en öne bir
> **reverse proxy (proxy web server)** koymak.

**Problem:** Şu an her servise ayrı port mapping'le erişiyoruz (broker `:8080`, mailhog `:8025`...).
Gerçek dünyada kullanıcı `http://site.com:8080/handle` yazmaz — sadece `http://site.com` yazar.
Dağınık portları **tek temiz adrese** indirip "hangi servise gidecek?" kararını birinin vermesi gerekir.

**Çözüm:** Tüm servislerin önüne bir **kapıcı** koyarsın. Bütün istekler önce ona gelir, o da
isteğe (genelde URL yoluna) bakıp doğru servise **yönlendirir (route)**.

```
                          ┌─────────────────────┐
   Kullanıcı (tarayıcı)   │   REVERSE PROXY     │
        │  http://localhost│  (Caddy/Nginx/      │
        └────────────────►│   Traefik)  :80     │
                          └──────────┬──────────┘
                          ┌──────────┴──────────┐
                          ▼                     ▼
                  ┌───────────────┐    ┌───────────────┐
                  │  Front-End    │    │    Broker     │
                  └───────────────┘    └───────────────┘
```
- `http://localhost/`        → **front-end** (arayüzü göster)
- `http://localhost/handle`  → **broker** (API isteği)

> **Neden sadece bu iki servis?** Dışarıdan trafik alan yalnızca **front-end** ve **broker** var.
> Geri kalanlar (auth, logger, mailer, listener, mongo, postgres) zaten iç ağda, dışarı kapalı —
> proxy onlara yönlendirme yapmaz.

**Reverse proxy ≠ broker (iki ayrı katman!):**

| | Reverse Proxy | Broker (API Gateway) |
|---|---|---|
| Nerede | Tüm Swarm'ın **en önünde** | Mikroservislerin önünde |
| Neye bakar | URL yoluna (`/` mı `/handle` mı) | İstek body'sindeki `action` alanına |
| Yönlendirir | front-end **veya** broker'a | auth / logger / mailer'a |
| Katman | Altyapı / network | Uygulama mantığı |

Yani akış uzuyor:
```
Tarayıcı → Reverse Proxy → Broker → (auth / logger / mailer)
                        ↘ Front-End
```

> Sonraki adım: `swarm.yml`'e proxy'yi (muhtemelen **Caddy**) yeni bir servis olarak eklemek
> + bir config dosyası (`Caddyfile` gibi) yazmak.

#### Caddy'yi `swarm.yml`'e ekleme (yapıldı)

Reverse proxy olarak **Caddy** seçtik ve `swarm.yml`'e ekledik:

```yaml
caddy:
  image: omerkoc0/micro-caddy:1.0.0   # kendi image'ımız (Caddyfile içine gömülü)
  deploy:
    mode: replicated
    replicas: 1
  ports:
    - "80:80"                         # HTTP  → dışarı bakan tek kapı
    - "443:443"                       # HTTPS → otomatik TLS için
  volumes:
    - caddy_data:/data                # SSL sertifikaları burada KALICI saklanır
    - caddy_config:/config

volumes:
  caddy_data:
    external: true                    # "bu volume'u ben önceden oluşturdum, sen yaratma"
  caddy_config:
```

**Kritik noktalar:**
- **`micro-caddy` = kendi image'ımız.** Resmi `caddy` image'ını doğrudan kullanmıyoruz çünkü
  **Caddyfile'ı içine gömmemiz** gerekiyor: Dockerfile resmi caddy'yi alır + `Caddyfile`'ı
  kopyalar → `micro-caddy` adıyla build + Docker Hub'a push.
- **Artık dışarı bakan tek kapı Caddy** (`:80`/`:443`). Dikkat: **broker'dan `ports` kaldırıldı**
  (eskiden `"8080:80"` vardı) — artık broker'a Caddy üzerinden erişiliyor. İstediğimiz tek temiz kapı.
- **`:443` = otomatik HTTPS.** Caddy'nin süper gücü: gerçek domain'de Let's Encrypt'ten SSL
  sertifikasını **otomatik alır ve yeniler**, elle uğraşmazsın (Nginx'te bu manuel).
- **`caddy_data` volume'u kritik:** Aldığı SSL sertifikaları burada saklanır. Olmasaydı her
  restart'ta sertifikayı sıfırdan alır → Let's Encrypt rate limit'ine takılırdın. Volume = kalıcılık.
- **`external: true`:** "Bu volume zaten var (`docker volume create caddy_data` ile elle oluşturdum),
  Swarm yenisini yaratma." → Deploy'dan önce `docker volume create caddy_data` çalıştırmak gerekir.

Akış artık:
```
Tarayıcı ──:80/:443──► Caddy ──┬──► front-end   (/ isteği)
                               └──► broker       (/handle isteği)
                                       │
                                       ▼
                              auth / logger / mailer
```

> ⚠️ **Takıldığım YAML hatası:** `front-end` servisinde `mode: replicated:` (sondaki fazladan
> `:`) yazmışım, `replicas` da yanlış girintiye kaymıştı. Doğrusu diğer servislerle aynı:
> `mode: replicated` + altına aynı hizada `replicas: 1`.

#### Caddyfile'ın içi — virtual host'lar + neden `:80`

`Caddyfile` Caddy'ye "hangi adrese gelen isteği nereye yönlendireceğini" söyler. Bizim dosyamız
(`project/Caddyfile`):

```caddy
{
    email   you@gmail.com           # Let's Encrypt sertifikası için iletişim maili
}

(static) {                          # tekrar kullanılabilir blok (snippet): statik dosya cache'i
	@static { file; path *.ico *.css *.js *.png *.svg *.woff *.json ... }
	header @static Cache-Control max-age=5184000
}

(security) { ... }                  # güvenlik header'ları (HSTS, nosniff, referrer-policy)

localhost:80 {                      # 1. VIRTUAL HOST → front-end
	encode zstd gzip                # yanıtları sıkıştır (hızlı transfer)
	import static                   # yukarıdaki (static) snippet'ini buraya kat
	reverse_proxy http://front-end:8081
}

backend:80 {                        # 2. VIRTUAL HOST → broker
	reverse_proxy http://broker-service:8080
}
```

**Virtual host (sanal sunucu) nedir?** Caddyfile'da her `isim:port { ... }` bloğu bir virtual
host. Caddy gelen isteğin **host adına** bakıp doğru bloğa yönlendirir:
- İstek `localhost`'a → **front-end:8081**
- İstek `backend`'e → **broker-service:8080**

> Bu, reverse proxy'nin "hangi servise?" kararı — ama URL path'ine değil, **host adına** göre.
> İçerideki `front-end:8081` / `broker-service:8080` Docker iç DNS isimleri (servis adıyla erişim).

**⚠️ Neden `localhost:80` — `:80` neden ŞART (hocanın uyarısı):**
Caddy varsayılan olarak her site bloğu için **otomatik HTTPS** dener → Let's Encrypt'ten SSL
sertifikası almaya çalışır. Ama `localhost`/`backend` **gerçek domain değil**, Let's Encrypt
bunlara sertifika **veremez** (internetten doğrulayamaz) → **patlar**.
```
localhost    { ... }   → Caddy ":443 HTTPS + Let's Encrypt" dener → HATA
localhost:80 { ... }   → ":80" = "düz HTTP yeter, sertifika deneme" → çalışır ✓
```
Yani `:80` yazarak Caddy'ye "burada HTTPS deneme" demiş oluyoruz. Gerçek domain (`site.com`)
olunca `:80`'i kaldırırsın, Caddy otomatik HTTPS'i devreye sokar.

#### `backend` ismini tanıtma (`/etc/hosts`)

`localhost` her makinede otomatik tanımlı ama `backend` **uydurma bir isim** — makine "backend
nedir?" diye bilmez. Tarayıcıdan `http://backend` çalışsın diye bu ismi makineye tanıttık.
`/etc/hosts` (makinenin mini telefon rehberi: "isim → IP") dosyasına ekledik:

```
127.0.0.1   localhost backend       # backend artık 127.0.0.1'i (kendi makineyi) işaret eder
::1         localhost backend       # aynısı IPv6 için
```
`127.0.0.1` = "bu makinenin kendisi". Düzenleme `sudo vim /etc/hosts` ile yapılır, `:wq` ile kaydedilir.

Tam zincir:
```
Tarayıcı "http://backend"
   │  /etc/hosts: backend = 127.0.0.1   ← bu satırı ekledik
   ▼
Caddy :80  (backend:80 bloğu yakalar)
   ▼
broker-service:8080
```

> **Not:** `/etc/hosts` ayarı **sadece bu makine** için geçerli — lokal test çözümü. Gerçek
> sunucuda `backend` yerine gerçek domain (`api.site.com`) olur, isim çözümü DNS ile yapılır.

#### SSL/TLS sertifikası nedir? (Caddy'nin otomatik HTTPS'inin temeli)

- **HTTP** = şifresiz hat, yolda (wifi/ISP/router) okunabilir. **HTTPS (SSL/TLS)** = şifreli hat → kilit ikonu, `https://`. (Doğru terim **TLS**, "SSL" eski adı; ikisi de aynı şey kastedilir.)
- **SSL sertifikası** = sitenin dijital kimliği, iki iş yapar: (1) **kimlik doğrulama** ("ben gerçekten o siteyim, sahte değil"), (2) **şifreleme** (içindeki public key ile veri şifrelenir, sadece sunucudaki private key çözer).
- **CA (Certificate Authority)** = sertifikayı imzalayan güvenilir kuruluş (pasaportu veren devlet gibi). Tarayıcılar güvenilir CA listesini önceden bilir. Bizim CA: **Let's Encrypt** (ücretsiz, otomatik).
- **Caddy'nin süper gücü:** Gerçek domain'de Let's Encrypt'ten sertifikayı otomatik **alır, kurar, 90 günde bir yeniler** — elle uğraşmazsın (Nginx'te bu manuel).

> **`caddy_data` volume'u burada anlam kazanır:** Caddy aldığı sertifikaları `/data`'ya yazar.
> Volume olmasaydı her container restart'ında sertifika kaybolur, Caddy yeniden ister ve
> Let's Encrypt **rate limit**'ine takılırdın. Volume = sertifika kalıcılığı.

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

### 🐝 Docker Hub & Docker Swarm
- **Docker Hub:** Image'ları sakladığın merkezi bulut deposu (registry). `docker push` ile yükle, makineler `docker pull` ile çeker. Detay: bkz. [Bölüm 3.6](#36-docker-swarm-ile-deployment-detay).
- **Docker Swarm:** Birden çok makineyi (node) tek küme yapıp container'ları yöneten orkestrasyon aracı. `docker-compose`'un çok-makineli, üretime yakın hali.
- **Node / Manager / Worker:** Node = kümedeki makine. Manager = karar veren beyin (komutları o kabul eder). Worker = sadece atanan container'ları çalıştıran işçi.
- **replica & self-healing:** Bir servisin kaç kopya çalışacağını sen söylersin (`replicas`), çöken kopyayı Swarm otomatik yeniden başlatır.
- **`replicated` vs `global`:** replicated = toplam N kopya; global = her node'da birer tane.
- **compose vs swarm farkı:** Aynı YAML formatı ama Swarm'da `deploy:` bloğu (mode, replicas) devreye girer; `docker stack deploy` ile dağıtılır. Bu projede: `project/swarm.yml`.

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

### 🔌 gRPC + Protocol Buffers
- **gRPC:** Google'ın RPC framework'ü. Servis-servis iletişiminde çok yaygın. HTTP/2 + binary → hızlı.
- **Protocol Buffers (protobuf):** Verinin binary serileştirme formatı + sözleşme dili (`.proto`). JSON'un dil-bağımsız, hızlı, tip-güvenli alternatifi.
- **`protoc`:** `.proto` → kaynak kod derleyicisi. macOS'ta `brew install protobuf` ile kurulur (standart `.proto` import'larıyla birlikte gelir). Tek başına Go üretmez.
- **Plugin'ler:** `protoc-gen-go` (mesajlar) + `protoc-gen-go-grpc` (servis). `go install ...@latest` ile kurulur, `$GOPATH/bin` PATH'te olmalı.
- Bu projede: **Broker (client) → Logger (server)** arası `LogService.WriteLog`. Detay: bkz. [Bölüm 3.5](#35-grpc-ile-servis-iletişimi-detay).

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
| 16 | `add communication between services using gRPC` | **gRPC + Protocol Buffers**. `.proto` sözleşmesi, `protoc` ile kod üretimi (`logs.pb.go` + `logs_grpc.pb.go`). Logger = server (`grpc.go`, port 50001), Broker = client (`LogViaGRPC`). Logger artık 3 protokol dinliyor. |
| 17 | Docker Swarm ile deploy (`project/swarm.yml`) | **Docker Hub** (image registry) + **Docker Swarm** (orkestrasyon). Image'ları `docker push` ile Hub'a yükledim. Node/manager/worker, `replicated` vs `global` mode, replica, self-healing kavramları. `docker swarm init` + `docker stack deploy` akışı. |

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

### "gRPC nedir, net/rpc'den farkı ne? Madem RPC var, niye bir de gRPC?"
İkisi de "uzaktaki fonksiyonu local'deymiş gibi çağırma" fikri. Farklar:
- **net/rpc** sadece Go ↔ Go çalışır (Go'nun gob formatı). gRPC **dil-bağımsız** — Go servisi
  Python/Java servisini çağırabilir, çünkü ortak dil `.proto`.
- **net/rpc** TCP üzerinde, gRPC **HTTP/2** üzerinde (multiplexing, streaming).
- gRPC **sözleşme önce (contract-first)** çalışır: `.proto`'da tanımlarsın, kod üretilir → iki taraf
  da kesin aynı tipi kullanır, tip-güvenlik garanti. net/rpc'de elle struct uydurursun.
Kısaca gRPC, RPC fikrinin endüstri-standardı, dil-bağımsız, daha hızlı hali.

### "protoc ne işe yarar? protoc-gen-go neden ayrı?"
`protoc` = protobuf derleyicisi. `.proto` dosyasını okur ama **tek başına Go üretmez** —
hangi dile çevireceğini plugin'ler söyler. `protoc-gen-go` mesaj tiplerini, `protoc-gen-go-grpc`
servis/client iskeletini üretir. protoc bu plugin'leri PATH'te `protoc-gen-*` ismiyle arar,
o yüzden `$GOPATH/bin` PATH'te olmalı. (macOS'ta `protoc`'u elle kopyalamak yerine
`brew install protobuf` — yanında standart `.proto` import'larını da getirir.)

### "logs.pb.go ve logs_grpc.pb.go dosyalarını elle düzenler miyim?"
**Hayır.** Bunlar `protoc` çıktısı (generated code). `.proto` değişince yeniden üretilir,
elle yaptığın değişiklik silinir. Sadece `.proto`'yu düzenle, sonra `protoc`'u tekrar çalıştır.
İçindeki `Unimplemented...Server` da senin değil, ileri-uyumluluk için üretilen base struct.

### "Aynı .proto neden hem broker'da hem logger'da var?"
RabbitMQ'daki `event.go` ile aynı mantık: gRPC'nin iki ucu (client + server) **birebir aynı
sözleşmeyi** kullanmak zorunda, yoksa binary uyuşmaz. Logger server'ı implemente eder,
broker client'ı kullanır — ama ikisi de aynı `logs.proto`'dan üretilmiş koda ihtiyaç duyar.
İdeal çözüm ortak bir paket/repo olurdu; kursta basitlik için kopyalandı.

### "grpc.Dial'daki insecure ve WithBlock ne?"
- `insecure.NewCredentials()` → **TLS yok**. Üretimde gRPC TLS ister ama bu servisler Docker
  iç ağında konuşuyor (dışarı kapalı), o yüzden şifrelemesiz. Üretimde mTLS eklenir.
- `grpc.WithBlock()` → bağlantı kurulana kadar `Dial` **bekler** (yoksa bağlantı arka planda
  kurulur, ilk çağrı "not ready" diye patlayabilir).

### "gRPC neden tarayıcıdan (web browser) doğrudan kullanılamıyor?" (kurs uyarısı)
Kurs şunu vurguluyor: **gRPC backend'de (servis-servis) harika, ama tarayıcıdan doğrudan
çalışmaz.** Sebep teknik: gRPC, **HTTP/2'nin düşük seviye kontrolüne** (trailer'lar, binary
framing) ihtiyaç duyar; tarayıcıların `fetch`/`XMLHttpRequest` API'leri bu kadar derine
erişim vermez. Bu yüzden tarayıcıdaki JavaScript doğrudan gRPC isteği atamaz.

Bu yüzden bizim mimaride iletişim ikiye bölünmüş — ve doğru yapılmış:
```
Front-end (tarayıcı) ──HTTP/JSON──→ Broker ──gRPC──→ Logger
   (gRPC YOK, çünkü                 (gRPC client)    (gRPC server)
    tarayıcı yapamaz)
```
- **Tarayıcı → Broker:** HTTP/JSON (`POST /log-grpc`). gRPC **değil**.
- **Broker → Logger:** gRPC. İki taraf da sunucu (Docker iç ağı) → çalışır ve hızlıdır.

> ⚠️ **İsim tuzağı:** `/log-grpc` endpoint'inin **kendisi HTTP**. "grpc" eki, broker'ın o
> isteği aldıktan *sonra* logger'a gRPC ile ileteceğini anlatır. Broker burada bir
> **çevirmen** gibi: dışarıdan HTTP alır, içeride gRPC konuşur.
>
> (Tarayıcıdan gRPC için **gRPC-Web** + proxy diye bir çözüm var ama kurs şimdilik oraya
> girmiyor. "Eventually that'll probably change" derken bunu kastediyor.)

### "Docker Hub nedir? Image'lar nereye taşındı?"
Docker Hub = image'ların saklandığı **bulut tabanlı kayıt deposu (registry)**. GitHub'ın kod
için olduğu şey. `docker push omerkoc0/broker-service:1.0.0` deyince image lokal makinenden
çıkıp **Docker Hub sunucularına** yüklenir. Yani image'lar lokalden internetteki merkezi depoya
taşındı. İsim formatı: `kullanıcı/image-adı:tag` → `omerkoc0/broker-service:1.0.0`. Detay: bkz.
[Bölüm 3.6](#36-docker-swarm-ile-deployment-detay).

### "Image'ları neden Docker Hub'a push'luyoruz?"
Çünkü Swarm container'ları birden çok makineye dağıtabilir ve o makinelerin hepsinin image'a
erişmesi gerekir. Lokal image'ı diğer makineler göremez; ama herkes Docker Hub'a erişip
`pull` edebilir. `push` = lokal image'ı Hub'a yükle, `pull` = Hub'dan makineye indir (Swarm bunu
otomatik yapar). `rabbitmq`, `mongo`, `postgres` gibi resmi image'lar zaten Hub'da, onları
push'lamaya gerek yok.

### "Docker Swarm nedir, docker-compose'dan farkı ne?"
Swarm = Docker'ın yerleşik **orkestrasyon** aracı: birden çok makineyi (node) tek küme yapıp
container'ları otomatik dağıtır. `docker-compose` **tek makinede** çalışır; Swarm bunun üstüne
multi-node, replica yönetimi, self-healing (çökeni otomatik başlatma) ve load balancing ekler.
Aynı YAML formatı ama Swarm'da `deploy:` bloğu (mode, replicas) anlam kazanır ve `docker stack
deploy` ile dağıtılır.

### "Node, manager, worker nedir?"
- **Node** = Swarm kümesine katılmış bir makine (sunucu/VM/laptop). Manager ya da worker olur.
- **Manager** = kümenin beyni. Durumu tutar, "kim nerede çalışacak / çökeni kim başlatacak" kararını verir, **yönetim komutlarını yalnızca o kabul eder** (`docker stack deploy` vb.). İstersen container da çalıştırır.
- **Worker** = sadece manager'ın atadığı container'ları çalıştıran işçi; karar vermez.
- Production'da **tek sayı manager** (3/5) seçilir çünkü Raft consensus'ta çoğunluk (quorum) gerekir. Lokalde tek node hem manager hem worker olur, yeterli.

### "`replicated` ve `global` mode farkı ne?"
- **`replicated` + `replicas: N`** → kümede **toplam N kopya** çalışır (örn. `replicas: 1`).
- **`global`** → kümedeki **her node'da birer tane** çalışır (bizim `rabbitmq`, `mongo`, `mailhog`).
Bir servisi ölçeklemek için `replicas` sayısını artırırsın; Swarm yeni kopyaları başlatır ve yükü dağıtır.

### "Self-healing nedir?"
Swarm'ın, senin müdahalen olmadan **bozulan durumu otomatik düzeltme** yeteneği. Temel mantık
**istenen durum (desired state)** vs **gerçek durum** karşılaştırması: `replicas: 1` dediğinde
Swarm'a "bundan hep 1 tane çalışsın" demiş olursun. Manager sürekli kontrol eder, ikisi
uyuşmazsa farkı kapatır.

```
İstenen: 1 replica          İstenen: 1 replica
Gerçek:  1 çalışıyor   ──►   Gerçek:  0 (çöktü!)
                                  │  Swarm fark eder
                                  ▼
                            Otomatik yeni container başlatır → Gerçek: 1 ✓
```

- Container çökerse → Swarm yenisini başlatır.
- Bir **node** komple çökerse → o makinenin container'larını başka sağlıklı node'lara taşır.
- Biri yanlışlıkla container silerse → eksik kopyayı geri getirir.

Faydası: dayanıklılık (resilience), az manuel müdahale, yüksek erişilebilirlik (replica >1 ise
biri çökerken diğerleri trafiği karşılar, kullanıcı kesinti hissetmez).

> ⚠️ **Sınır:** Self-healing "çalışması gerekirken durmuş olanı" geri getirir — **kötü kodu
> düzeltmez**. Uygulama bug yüzünden sürekli çöküyorsa Swarm sürekli yeniden başlatır
> (**crash loop**). Container'ın içindeki uygulama gerçekten sağlıklı mı diye bakmak için ayrı
> bir **healthcheck** mekanizması var.

### "Deploy edilmiş servisi nasıl güncellerim? Önce scale, sonra değiştir mi?"
Sezgi doğru ama **scale ile update ayrı işlemler** — scale, update'in *parçası* değil, onu
**kesintisiz** yapan ön koşul. Doğru akış:
1. Yeni versiyonu **yeni tag**'le build + push (`1.0.0` → `1.0.1`). Swarm değişikliği tag'den anlar, aynı tag'i ezmek güvenilir değil.
2. **Downtime olmaması için replica > 1 olmalı.** Tek replica varsa update'te o tek container durunca boşluk (downtime) olur → önce `docker service scale ...=3`.
3. `docker service update --image ...:1.0.1 <servis>` → Swarm replica'ları **teker teker** değiştirir (**rolling update**); biri güncellenirken diğerleri trafiği karşılar.
4. `update_config` ile davranış ayarlanır: `order: start-first` (önce yeniyi başlat) + `failure_action: rollback` (patlarsa otomatik geri dön).

Zaten `replicas` 1'den fazlaysa ayrıca scale etmene gerek yok, doğrudan `service update` yeterli.
Detay + diyagram: bkz. [Bölüm 3.6](#36-docker-swarm-ile-deployment-detay).

### "Reverse proxy nedir, neden lazım? Broker'dan farkı ne?"
**Problem:** Swarm'ı ayağa kaldırmak kolay ama front-end'e/broker'a düzgün erişim yok — her
servis ayrı portta (`:8080`, `:8025`...). Kullanıcı `http://site.com:8080/handle` yazmaz.
**Çözüm:** Tüm servislerin en önüne bir **reverse proxy (Caddy/Nginx/Traefik)** koyarsın; bütün
istekler ona gelir, o **URL yoluna** bakıp doğru servise yönlendirir: `/` → front-end, `/handle`
→ broker. Dışarı bakan **sadece bu iki servis** var; gerisi iç ağda kapalı.

**Broker'la karıştırma — iki ayrı katman:** Reverse proxy *tüm Swarm'ın en önünde*, **URL yoluna**
bakar (front-end mi broker mı). Broker (API Gateway) *mikroservislerin önünde*, istek
body'sindeki **`action` alanına** bakar (auth/log/mail). Akış: `Tarayıcı → Proxy → Broker →
auth/logger/mailer`. Detay: bkz. [Bölüm 3.6](#36-docker-swarm-ile-deployment-detay).

### "Caddyfile'da `localhost:80` neden `:80` ile? Olmazsa ne olur?"
Caddy varsayılan olarak her site bloğu için **otomatik HTTPS** dener → Let's Encrypt'ten SSL
sertifikası ister. Ama `localhost`/`backend` gerçek domain değil, Let's Encrypt bunlara sertifika
**veremez** → patlar. `:80` yazmak Caddy'ye **"düz HTTP yeter, HTTPS deneme"** der. Gerçek
domain'de `:80`'i kaldırırsın, otomatik HTTPS devreye girer. Her `isim:port { }` bloğu bir
**virtual host** (Caddy host adına göre yönlendirir: `localhost`→front-end, `backend`→broker).

### "`backend`'i neden `/etc/hosts`'a ekledik?"
`localhost` her makinede otomatik tanımlı ama `backend` **uydurma bir isim** — makine onu
çözemez. `/etc/hosts`'a `127.0.0.1 localhost backend` ekleyince `backend` de kendi makineyi
işaret eder, böylece tarayıcıdan `http://backend` Caddy'ye ulaşır. Sadece **bu makine** için
geçerli lokal çözüm; gerçek sunucuda yerine gerçek domain + DNS olur.

### "SSL/TLS sertifikası nedir?"
Sitenin **dijital kimlik belgesi**. İki iş yapar: (1) **kimlik doğrulama** — "ben gerçekten o
siteyim, sahte değil" (güvenilir bir **CA**/otorite imzalar), (2) **şifreleme** — içindeki public
key ile veri şifrelenir, sadece sunucudaki private key çözer (yolda kimse okuyamaz). `https://`
ve kilit ikonu bu demek. **CA** = sertifikayı imzalayan güvenilir kuruluş (pasaport veren devlet
gibi); bizimki **Let's Encrypt** (ücretsiz, otomatik). **Caddy** bu sertifikayı otomatik alır,
kurar ve yeniler — `caddy_data` volume'u da sertifikaları kalıcı saklar (yoksa rate limit'e
takılırsın). Terim notu: doğrusu **TLS**, "SSL" eski adı, ikisi de aynı şey.

### "Linode'dan server kiralama — bu artık mikroservise özel değil mi?"
Doğru. Buradan sonrası **mikroservis mimarisi değil, deployment/DevOps**. Server kiralamak
(Linode = bulut VPS sağlayıcısı, AWS/DigitalOcean gibi), Docker'a koyabildiğin **herhangi bir
projeyi** yayına almanın bir yolu — mikroservis, monolith, tek script fark etmez, adımlar aynı.
Bu yüzden bu bilgi **transfer edilebilir**: image yap → Docker Hub'a push → sunucu kirala →
Swarm kur → deploy → proxy+SSL. Hocanın "2 server al" demesinin sebebi: multi-node Swarm'ı
gerçekten görmek (Server 1 = manager/`swarm init`, Server 2 = worker/`swarm join`; bir node
çökünce container'ların diğerine kaymasını izlemek). **Ücretli** olduğu için alternatif:
Play with Docker (ücretsiz, tarayıcıda) ya da lokal VM'ler. Detay: bkz. [Bölüm 3.6](#36-docker-swarm-ile-deployment-detay).

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
| `zsh: command not found: protoc` | `protoc`'u elle `go/bin`'e kopyalamıştım ama (1) o klasör PATH'te değildi (2) elle kopya eksikti (yanında `include/` `.proto` dosyaları yok) | `brew install protobuf` (PATH'teki `/opt/homebrew/bin`'e + standart `.proto`'larla kurar). Ayrı olarak `protoc-gen-go` + `protoc-gen-go-grpc` plugin'leri `go install` ile, `$GOPATH/bin` `~/.zshrc`'ye eklendi |
| `swarm.yml` front-end servisi bozuk | `mode: replicated:` (sonda fazladan `:`) yazılmış, `replicas` yanlış girintiye kaymıştı | `mode: replicated` + altına aynı hizada `replicas: 1` (diğer servislerle tutarlı) |

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
- [x] gRPC ile servis-servis iletişimi — `logs.proto`, `protoc` kod üretimi, logger server (`grpc.go`, :50001), broker client (`LogViaGRPC`). Logger artık 3 protokol dinliyor. Bkz. [Bölüm 3.5](#35-grpc-ile-servis-iletişimi-detay).
- [x] RPC (logger'daki `rpcPort = 5001`) — `rpc.go` + `rpcListen()` tamamlandı
- [ ] gRPC'yi `make`/Docker'a entegre et + front-end test sayfasına "Test gRPC" butonu (henüz endpoint `/log-grpc` elle test ediliyor)
- [ ] Servislerin daha fazla asenkron mesajlaşmaya geçmesi
- [ ] Production güvenliği: mail servisini iç ağa kapatma, broker'ın doğrudan mail çağırmaması
- [x] Deployment / orchestration — **Docker Swarm**: image'lar Docker Hub'a push'landı, `swarm.yml` ile `docker stack deploy`. Bkz. [Bölüm 3.6](#36-docker-swarm-ile-deployment-detay).
- [ ] Swarm'da servisleri ölçekleme (`docker service scale`) + rolling update (yeni image versiyonuna sıfır kesintiyle geçiş)
- [ ] (İleride) Kubernetes ile aynı sistemi deploy etme
- [ ] Monitoring / tracing

---

> 📝 **Bu dosya nasıl güncellenecek:** Kursta yeni bir bölüm bittiğinde / yeni kod
> eklediğimde, ilgili bölümlere (servisler, teknolojiler, zaman çizelgesi, kavram sözlüğü)
> yeni satırlar ekleyeceğim. Sorduğum her yeni soru "Kavram Sözlüğü"ne, her yeni hata
> "Takıldığım Hatalar"a girecek.
