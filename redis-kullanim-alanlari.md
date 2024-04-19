# Redis Kullanım Alanları

**1. Caching**

Redis, sık erişilen verilerin hızlı bir şekilde saklanması ve erişilmesi için kullanılır. Bu, veritabanı yükünü azaltır ve uygulama yanıt sürelerini iyileştirir.

Örnek Kod (Task Detaylarını Cacheleme):

```bash
func getTask(c *fiber.Ctx) error {
    // Kullanıcıdan alınan ID parametresi
    id := c.Params("id")
    ctx := context.Background()

    // Redis cache'den task'ı deneyerek almayı dene
    cachedTask, err := database.RedisClient.Get(ctx, "task:"+id).Result()
    if err == nil {
        // Cache'den task bulunursa, JSON formatına çevir ve cevap olarak dön
        task := new(models.Task)
        json.Unmarshal([]byte(cachedTask), task)
        return c.JSON(task)
    }

    // Cache'de bulunamazsa, PostgreSQL veritabanından task'ı çek
    task := new(models.Task)
    err = database.PgConn.QueryRow(ctx, "SELECT id, header, description, creation_time FROM tasks WHERE id = $1", id).Scan(&task.ID, &task.Header, &task.Description, &task.CreationTime)
    if err != nil {
        // Task bulunamazsa, 404 hatası dön
        return c.Status(404).SendString("Task not found")
    }

    // Task bulunursa, sonraki talepler için Redis cache'ine kaydet
    jsonData, _ := json.Marshal(task)
    database.RedisClient.Set(ctx, "task:"+id, jsonData, 30*time.Minute) // 30 dakika boyunca geçerli olacak şekilde set et
    return c.JSON(task) // Task'ı JSON olarak dön
}
```

**<u>Test:</u>**

**Redis Araçlarını Yükleme**

- Redis-CLI Yükleyin

Ubuntu veya Debian tabanlı bir sistemde redis-cli yüklemek için aşağıdaki komutu kullanın:

```
sudo apt update
sudo apt install redis-tools
```
- Yükleme Sonrası Redis-CLI Kullanımı

Yükleme tamamlandıktan sonra, redis-cli komutunu kullanarak Redis sunucunuza bağlanabilirsiniz. Eğer Redis sunucunuz varsayılan ayarlarla (localhost ve 6379 portu) çalışıyorsa, doğrudan **redis-cli** yazarak Redis komut satırına erişebilirsiniz.

**Redis-CLI ile Veri Kontrolü**

Redis'e bağlandıktan sonra, önceden oluşturduğunuz task için kayıtlı veriyi kontrol etmek için aşağıdaki komutu kullanabilirsiniz:

```
GET task:1
```
Bu komut, **task:1** anahtarına karşılık gelen değeri getirecektir. Eğer bu anahtar mevcut ise, task bilgilerini gösterir.

**Örnek Redis Kullanımı**

Redis komut satırına girdikten sonra yapabileceğiniz bazı işlemler şunlardır:

- Anahtarların Listelenmesi:

```
KEYS *
```
Bu komut, Redis veritabanındaki tüm anahtarları listeler.

- Bir Anahtarın Değerini Alma:

```
GET task:1
```
Bu komut, task:1 anahtarına ait değeri döner. Eğer bu anahtar mevcut değilse, nil yanıtını alırsınız.

- Bir Anahtarın Süresini Ayarlama:

```
EXPIRE task:1 60
```
Bu komut, task:1 anahtarının süresini 60 saniye olarak ayarlar. 60 saniye sonra bu anahtar otomatik olarak silinir.

**2. Message Queue**

Redis listeleri veya pub/sub mekanizması, asenkron mesaj sıralaması için kullanılabilir.

Örnek Kod (Task Sıralama ve İşleme):

```bash
func enqueueTask(c *fiber.Ctx) error {
    // Yeni bir Task nesnesi oluştur
    task := new(models.Task)
    // HTTP isteğinin body'sinden task verilerini çözümle
    if err := c.BodyParser(task); err != nil {
        // Eğer çözümleme sırasında hata oluşursa, 400 durum kodu ile hata mesajını gönder
        return c.Status(400).SendString(err.Error())
    }

    // Task nesnesini JSON formatına dönüştür
    taskData, _ := json.Marshal(task)
    ctx := context.Background()

    // Dönüştürülen task verisini Redis listesine sol taraftan (başlangıç) ekleyerek kuyruğa al
    database.RedisClient.LPush(ctx, "task_queue", taskData)

    // İşlem başarılı olduğunda HTTP 201 durum kodu ile yanıt dön
    return c.Status(201).SendString("Task enqueued")
}

func ProcessTasks() {
    ctx := context.Background()
    for {
        // Sürekli olarak Redis listesinin sağ tarafından (sonundan) task çek
        result, err := database.RedisClient.RPop(ctx, "task_queue").Result()

        if err == nil {
            // Eğer task başarıyla çekilirse, JSON verisini Task nesnesine dönüştür
            task := new(models.Task)
            json.Unmarshal([]byte(result), task)

            // İşlenen task'ın ID'sini konsola yazdır
            fmt.Println("Processed task ID:", task.ID)
        }

        // Her döngüde 1 saniye bekleme süresi ekle (basit bir oran sınırlama)
        time.Sleep(1 * time.Second)
    }
}
```

**<u>Test:</u>**

- Test Etme:

**ProcessTasks** fonksiyonu doğrudan bir HTTP isteği ile tetiklenmediği için, fonksiyonun doğru çalıştığını test etmek için enqueued task'ların işlenip işlenmediğini gözlemlemeniz gerekir. Bunun için **enqueueTask** fonksiyonu ile birkaç task ekleyip, console çıktılarını gözlemleyebilirsiniz.


- Request türü için "POST" seçeneğini belirleyin.
- URL alanına http://localhost:3000/enqueue yazın.
- "Headers" sekmesine geçin.
- Key olarak Content-Type ve Value olarak application/json ekleyin.
- "Body" sekmesine geçin ve "raw" seçeneğini belirleyin.
Aşağıdaki JSON verisini girin:

```
{
  "header": "Sample Task",
  "description": "This is a test task."
}
```
- Yanıt, Postman'in alt kısmındaki "Body" sekmesinde görüntülenecektir. Başarılı bir request sonucunda HTTP 201 durum kodu ve "Task enqueued" mesajını görmelisiniz.

- Logların Gözlemlenmesi:
Uygulamanız çalışırken, console üzerinde "Processed task ID: [TaskID]" şeklinde loglar görülmelidir. Bu loglar, ProcessTasks fonksiyonunun arka planda başarıyla çalıştığını ve task'ları işlediğini gösterir.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k7p9fur27b2h52b7vtp9.png)


**3. Real-Time Applications**

Redis pub/sub kanalları, gerçek zamanlı uygulamalar için bir iletişim köprüsü olarak kullanılabilir.

Örnek Kod (Gerçek Zamanlı Task Güncellemeleri):

```bash
func publishTask(task *models.Task) {
    // Yeni bir bağlam (context) oluştur
    ctx := context.Background()
    
    // Task nesnesini JSON formatına dönüştür
    taskData, err := json.Marshal(task)
    if err != nil {
        // Eğer JSON dönüşümü sırasında hata oluşursa, hatayı logla
        log.Printf("Error marshaling task data: %v", err)
        return
    }
    
    // Dönüştürülen task verisini Redis pub/sub kanalına yayınla
    err = database.RedisClient.Publish(ctx, "tasks", taskData).Err()
    if err != nil {
        // Eğer yayınlama sırasında hata oluşursa, hatayı logla
        log.Printf("Error publishing task to Redis: %v", err)
    } else {
        // Eğer yayınlama başarılı ise, hangi task'ın yayınlandığını logla
        log.Printf("Task published: %v", task.ID)
    }
}

func subscribeToTasks(c *fiber.Ctx) error {
    // Yeni bir bağlam (context) oluştur
    ctx := context.Background()
    
    // Redis pub/sub kanalına abone ol
    pubsub := database.RedisClient.Subscribe(ctx, "tasks")
    // Fonksiyon bitiminde pubsub bağlantısını kapat
    defer pubsub.Close()

    // Mesaj kanalını al
    ch := pubsub.Channel()

    // Mesaj kanalından mesaj gelmesini bekle
    select {
    case msg := <-ch:
        // Eğer mesaj alınırsa, alınan task'ı konsola yazdır ve HTTP yanıtı olarak döndür
        fmt.Println("Received task:", msg.Payload)
        return c.SendString("Received message: " + msg.Payload)
    case <-time.After(10 * time.Second): // Eğer 10 saniye içinde mesaj gelmezse timeout ol
        // Timeout durumunda 408 Request Timeout HTTP durum kodunu döndür
        return c.SendStatus(fiber.StatusRequestTimeout)
    }
}
```

**publishTask Fonksiyonu**

Bu fonksiyon, bir task nesnesini JSON formatına dönüştürür ve bu veriyi Redis'in pub/sub kanalına "tasks" kanalı üzerinden yayınlar.

Kodun Anlatımı:

- Fonksiyon, verilen task nesnesini JSON formatına serileştirir.
- Serileştirme işlemi başarısız olursa, hata loglanır ve fonksiyon sonlandırılır.
- Eğer serileştirme başarılı olursa, bu veri Redis üzerinde "tasks" kanalına yayınlanır.
- Yayın sırasında bir hata oluşursa, bu hata loglanır. 
- Başarılı bir yayın durumunda, hangi task'ın yayınlandığı loglanır.

**subscribeToTasks Fonksiyonu**

Bu fonksiyon, "tasks" kanalına abone olur ve bu kanaldan mesajları dinler. Her alınan mesaj konsola yazdırılır.

Kodun Anlatımı:

- Redis'e "tasks" kanalına abone olunur.
- Abonelik başarılı olursa, mesajlar için bir kanal (ch) alınır.
- Bu kanaldan gelen her mesaj konsola yazdırılır.
- Fonksiyon sürekli çalıştığı için bir HTTP isteği sonucunda bu fonksiyonu çağırmak uygun olmayabilir, bu nedenle daha ziyade arka plan servisi olarak çalıştırılması önerilir.

**<u>Test:</u>**

Bu iki fonksiyonun test edilmesi için iki aşamalı bir yaklaşım izleyebiliriz:

1) Abonelik Testi

İlk olarak, subscribeToTasks fonksiyonunu ayrı bir terminal veya log izleme aracında çalıştırarak Redis kanalından mesajları dinlemeye başlayabilirsiniz.

Eğer bu fonksiyonu bir HTTP endpoint olarak tetiklemek istiyorsanız:

```
curl http://localhost:3000/subscribe
```
Bir mesaj yayınlandığında, subscribeToTasks fonksiyonu bu mesajı alacak ve konsola "Received task: [Message Detail]" şeklinde çıktı verecektir.

2) Yayın Testi

Farklı bir terminal açıp, publishTask fonksiyonunu çağırarak bir task oluşturun ve bu task'ı Redis'e yayınlayın.

```
curl -X POST http://localhost:3000/task -H "Content-Type: application/json" -d '{"header":"Another Dynamic Task", "description":"Another test for subscription"}'
```
Bu komut, yeni bir task oluşturur ve bu task Redis kanalına yayınlanır.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/oy3ny4fi6g9jo0qvqv3f.png)


**4. Session Store**

Redis, hızlı erişilebilir ve süresi dolabilir oturum bilgilerini saklamak için idealdir.

Örnek Kod (Oturum Yönetimi):

```bash
func storeSession(c *fiber.Ctx) error {
    // Kullanıcı ID'sini URL parametresinden al
	userID := c.Params("user_id")
	// Oturum verisini oluştur, burada son giriş zamanı saklanıyor
	sessionData := map[string]interface{}{
		"last_login": time.Now(), // Şu anki zamanı kaydet
	}
	// Oturum verisini JSON formatına dönüştür
	sessionJSON, _ := json.Marshal(sessionData)
	// Yeni bir bağlam (context) oluştur
	ctx := context.Background()

	// Oluşturulan oturum verisini Redis'e 24 saat süreyle kaydet
	database.RedisClient.Set(ctx, "session:"+userID, sessionJSON, 24*time.Hour)
	// İşlem başarılıysa HTTP 200 durum kodu döndür
	return c.SendStatus(fiber.StatusOK)
}

func checkSession(c *fiber.Ctx) error {
	// Kullanıcı ID'sini URL parametresinden al
	userID := c.Params("user_id")
	// Yeni bir bağlam (context) oluştur
	ctx := context.Background()

	// Redis'den oturum verisini çek
	sessionData, err := database.RedisClient.Get(ctx, "session:"+userID).Result()
	if err != nil {
		// Eğer oturum verisi bulunamazsa 401 Unauthorized hatası döndür
		return c.Status(401).SendString("Unauthorized")
	}

	// Oturum geçerliyse, oturum verisini HTTP yanıtı olarak döndür
	return c.Status(200).SendString("Session valid: " + sessionData)
}
```

**<u>Test:</u>**

**1. storeSession Fonksiyonunu Test Etmek**

Bu fonksiyon, kullanıcı oturum bilgisini Redis'e kaydeder. Test etmek için, öncelikle bir POST isteği göndermelisiniz.

Ubuntu Terminalinde Curl ile Test:

```
curl -X POST http://localhost:3000/session/123
```
Bu komut, kullanıcı ID'si 123 olan bir kullanıcı için oturum bilgisini kaydeder. Komut başarılı olduğunda, HTTP 200 durum kodu döner.

**2. checkSession Fonksiyonunu Test Etmek**

Bu fonksiyon, kaydedilmiş bir kullanıcı oturumunu kontrol eder.

Ubuntu Terminalinde Curl ile Test:

```
curl http://localhost:3000/session/123
```
Bu komut, kullanıcı ID'si 123 olan bir kullanıcının oturum bilgisini kontrol eder. Oturum bilgisi doğruysa, "Session valid: [session bilgisi]" şeklinde bir yanıt döner. Oturum bilgisi yoksa veya hatalıysa, HTTP 401 Unauthorized hatası alırsınız.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e3jukkltnlgmj6r1mgdp.png)



**5. Rate Limiting**

Redis, API sınırlamalarını uygulamak için kullanılabilir. Kullanıcıların veya IP adreslerinin belirli bir süre içinde yapabileceği istek sayısını sınırlayarak hizmetin kötüye kullanılmasını önler.

Örnek Kod (API Rate Limiting):

```bash
func rateLimiter(c *fiber.Ctx) error {
    // Kullanıcı ID'sini URL parametresinden alarak unique bir key oluştur
	userID := c.Params("user_id")
	ctx := context.Background()  // Yeni bir context oluştur
	key := "rate_limit:" + userID

	// Redis'te bu kullanıcı için kaydedilen istek sayısını kontrol et
	resp, err := database.RedisClient.Get(ctx, key).Result()
	if err == redis.Nil {
		// Eğer bu anahtar yoksa, bu kullanıcının ilk isteği demektir, counter'ı 1 olarak ayarla
		database.RedisClient.Set(ctx, key, 1, time.Minute)
	} else if err != nil {
		// Redis'den veri çekerken hata oluşursa, 500 Internal Server Error dön
		return c.Status(500).SendString("Rate limit check failed")
	} else {
		// Redis'deki counter değerini integer'a çevir
		count, _ := strconv.Atoi(resp)
		if count >= 100 {  // Eğer kullanıcı belirlenen limiti (örneğin 100 istek) aşmışsa
			return c.Status(429).SendString("Rate limit exceeded")
		}
		// Aksi takdirde, istek sayısını bir arttır
		database.RedisClient.Incr(ctx, key)
	}

	// Eğer limit aşılmamışsa, bir sonraki middleware fonksiyonuna geç
	return c.Next()
}
```

**<u>Test:</u>**

**Adım 1: Endpoint Tanımlama**

İlk olarak, rateLimiter middleware'ini kullanacak bir endpoint tanımlayın. Bu örnekte, basit bir GET isteğine yanıt veren bir endpoint kullanacağım:

```
// RegisterRoutes fonksiyonuna bu yeni route'u ekleyin
func RegisterRoutes(app *fiber.App) {
    // Diğer route tanımlamaları...

    app.Get("/test-rate-limit", rateLimiter, func(c *fiber.Ctx) error {
        return c.SendString("Rate limit test başarılı!")
    })
}
```

**Adım 2: Test Scripti Oluşturma**

Bu endpoint'i test etmek için, belirlenen süre içinde birden fazla istek göndererek rateLimiter'in istek sayısını doğru bir şekilde sınırlayıp sınırlamadığını kontrol edebilirsiniz. Bu test için bir bash scripti kullanabilirsiniz. Script, belirtilen endpoint'e 105 kez istek gönderecek ve çıktıları ekrana basacaktır:

```
for i in {1..105}; do
  curl -X GET http://localhost:3000/test-rate-limit -H "Content-Type: application/json"
done
```
Script çalıştırıldığında, çıktıları dikkatlice inceleyin. İlk 100 istekten sonra 429 durum kodu (Too Many Requests) ve "Hız limiti aşıldı" mesajının döndüğünü görmelisiniz. Eğer bu sonuçları alıyorsanız, rateLimiter fonksiyonunuz beklenen şekilde çalışıyor demektir.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5bdznqfw6cl8te6xyv91.png)


**6. Geospatial Data Handling**

Redis'in geospatial desteği, konum tabanlı sorguları destekler, böylece kullanıcılar yakındaki nesneleri veya yerleri sorgulayabilir.

Örnek Kod (Konum Tabanlı Sorgular):

```bash
func storeLocation(c *fiber.Ctx) error {
	// URL'den alınan kullanıcı ID'si ve sorgu parametrelerinden enlem ve boylamı al
	userID := c.Params("user_id")
	lat, _ := strconv.ParseFloat(c.Query("lat"), 64)
	lon, _ := strconv.ParseFloat(c.Query("lon"), 64)
	ctx := context.Background()

	// Kullanıcının konumunu Redis geospatial set'e kaydet
	// Burada kullanıcı adı, boylam ve enlem ile yeni bir geo lokasyon oluşturuluyor
	database.RedisClient.GeoAdd(ctx, "user_locations", &redis.GeoLocation{
		Name:      userID, // Redis'teki isim olarak kullanıcı ID'si kullanılıyor
		Longitude: lon,    // Kullanıcının boylamı
		Latitude:  lat,    // Kullanıcının enlemi
	})
	return c.SendStatus(fiber.StatusOK) // Başarılı kayıt sonrası OK durum kodu gönder
}

func findNearby(c *fiber.Ctx) error {
	// Kullanıcı ID'sini URL'den al
	userID := c.Params("user_id")
	ctx := context.Background()

	// Redis Geo komutunu kullanarak belirli bir yarıçap içindeki kullanıcıları bul
	// Bu örnekte, 5000 metre (5 kilometre) yarıçap içinde en fazla 10 kullanıcı listelenir ve
	// sonuçlar mesafeye göre artan sırada döndürülür.
	res, err := database.RedisClient.GeoRadiusByMember(ctx, "user_locations", userID, &redis.GeoRadiusQuery{
		Radius:    5000, // Metre cinsinden yarıçap
		Unit:      "m",  // Yarıçabın birimi (metre)
		Count:     10,   // Döndürülecek maksimum kullanıcı sayısı
		Sort:      "ASC",// Artan sıralama (en yakından en uzağa)
	}).Result()
	if err != nil {
		return c.Status(500).SendString("Failed to query nearby users") // Sorgu hatası durumunda hata mesajı dön
	}

	return c.JSON(res) // Bulunan kullanıcıların listesini JSON formatında dön
}
```

**<u>Test:</u>**

**Adım 1: Endpoint'leri Tanımlama**

Öncelikle, storeLocation ve findNearby fonksiyonları için endpoint'lerinizi tanımlayın:

```
func RegisterRoutes(app *fiber.App) {
    app.Post("/location/:user_id", storeLocation)  // Kullanıcı konumu kaydet
    app.Get("/nearby/:user_id", findNearby)       // Yakındaki kullanıcıları bul
}
```
**Adım 2: Test Script'lerinin Hazırlanması**

- Kullanıcı Konumu Kaydetme

Bir kullanıcının konumunu kaydetmek için aşağıdaki gibi bir curl komutu kullanabilirsiniz. Örnek olarak, user1 ID'li kullanıcının enlem (lat) ve boylam (lon) bilgilerini gönderelim:

```
curl -X POST "http://localhost:3000/location/user1?lat=40.7128&lon=-74.0060" -H "Content-Type: application/json"
```
- Yakındaki Kullanıcıları Bulma

Bir kullanıcının yakınındaki kullanıcıları bulmak için aşağıdaki gibi bir curl komutu kullanabilirsiniz. Örnek olarak, user1 ID'li kullanıcı için yakındaki kullanıcıları sorgulayalım:

```
curl -X GET "http://localhost:3000/nearby/user1" -H "Content-Type: application/json"
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/znbv8lsudjda1woprb7g.png)

**7. Session Replay**

Redis, kullanıcı etkinliklerini kaydetmek için kullanılabilir, bu kayıtlar daha sonra oturum tekrarı için kullanılabilir.

Örnek Kod (Session Replay):

```bash
func logUserAction(c *fiber.Ctx) error {
    userID := c.Params("user_id") // Kullanıcı ID'sini URL parametresinden al
    action := string(c.Body())    // Kullanıcının yaptığı eylemi request body'den al
    timestamp := time.Now().Unix() // Şu anki zamanı Unix zaman damgası olarak al
    ctx := context.Background()

    // Kullanıcının eylemini ve zaman damgasını Redis sorted set'e kaydet
    // Eylemler, zaman damgasına göre sıralanacaktır
    database.RedisClient.ZAdd(ctx, "session:"+userID, &redis.Z{
        Score:  float64(timestamp), // Zaman damgası, sıralama için kullanılır
        Member: action,             // Eylem metni
    })
    fmt.Printf("Action logged: %s\n", action) // Eylemi debug amaçlı logla
    return c.SendStatus(fiber.StatusOK) // Başarılı bir şekilde loglandığını belirten HTTP durum kodu dön
}

func replaySession(c *fiber.Ctx) error {
    userID := c.Params("user_id") // Kullanıcı ID'sini URL parametresinden al
    ctx := context.Background()

    // Redis'teki sorted set'ten, kullanıcının tüm eylemlerini zaman sırasına göre al
    // ZRange fonksiyonu, belirtilen aralıktaki tüm elemanları döndürür
    actions, err := database.RedisClient.ZRange(ctx, "session:"+userID, 0, -1).Result()
    if err != nil {
        return c.Status(500).SendString("Failed to replay session") // Eylemleri alırken bir hata oluştuğunda hata mesajı dön
    }

    return c.JSON(actions) // Kullanıcının eylem listesini JSON formatında dön
}
```

**<u>Test:</u>**

**Adım 1: Endpoint'leri Tanımlama**

```
func RegisterRoutes(app *fiber.App) {
    // Diğer route tanımlamaları...
    app.Post("/log-action/:user_id", logUserAction)  // Kullanıcı aksiyonunu kaydeder
    app.Get("/replay-session/:user_id", replaySession)  // Kullanıcının oturum aksiyonlarını geri döndürür
}
```

**Adım 2: curl Komutu ile Test Et**

- Kullanıcı Eylemini Kaydetme:

Terminalden curl komutu ile API'nize bir POST isteği göndererek kullanıcı eylemini kaydedebilirsiniz. Örneğin, kullanıcı ID'si user123 ve eylem olarak basit bir metin mesajı gönderelim:

```
curl -X POST http://localhost:3000/log-action/user123 -d 'User logged in'
```
Burada -d parametresi ile gönderilen veri kullanıcının yaptığı eylemi temsil ediyor.

- Kullanıcı Oturumunu Yeniden Oynatma:

Kullanıcı eylemlerini yeniden oynatmak için bir GET isteği yapın:

```
curl -X GET http://localhost:3000/replay-session/user123
```
Bu komut, kullanıcı user123 için kaydedilmiş tüm eylemleri listeler.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0r1bg8mq8zepvmewujo3.png)


**8. Lightweight Transactions**

Redis'in MULTI/EXEC komutları, birden çok işlemin atomik olarak yürütülmesini sağlar, bu da veri bütünlüğünü korumaya yardımcı olur.

Örnek Kod (Transaction Kullanımı):

```bash
func updateProfile(c *fiber.Ctx) error {
	userID := c.Params("user_id") // Kullanıcı ID'sini URL parametresinden al
	newEmail := c.FormValue("email") // Kullanıcının yeni email adresini form verisinden al
	ctx := context.Background()

	// Redis'de kullanıcının email adresini güncellemek için bir transaction başlat
	tx := database.RedisClient.TxPipeline()
	tx.HSet(ctx, "user:"+userID, "email", newEmail) // Hash set kullanarak kullanıcının email adresini güncelle
	tx.Publish(ctx, "profile_updates", userID+" updated their email") // Güncelleme bilgisini pub/sub kanalında yayınla
	_, err := tx.Exec(ctx) // Transaction'ı çalıştır
	if err != nil {
		return c.Status(500).SendString("Failed to update profile") // Eğer transaction sırasında bir hata oluşursa, hata mesajı ile birlikte 500 durum kodu dön
	}

	return c.SendStatus(fiber.StatusOK) // Başarılı güncelleme sonrası 200 durum kodu dön
}
```

**<u>Test:</u>**

**Adım 1: Endpoint Tanımlama**

İlk olarak, updateProfile fonksiyonunuzun doğru bir şekilde RegisterRoutes fonksiyonuna eklenip eklenmediğini kontrol edin. Eğer eklenmediyse, aşağıdaki şekilde ekleyebilirsiniz:

```
func RegisterRoutes(app *fiber.App) {
    // Diğer route tanımlamaları...
    app.Post("/update-profile/:user_id", updateProfile)  // Kullanıcı profili günceller
}
```

**Adım 2: curl Komutu ile Test Et**

Terminalden veya Postman'den aşağıdaki curl komutu ile updateProfile fonksiyonalitesini test edebilirsiniz:

```
curl -X POST http://localhost:3000/update-profile/user123 -F 'email=newemail@example.com'
```
Bu komut, kullanıcı "user123" için yeni bir e-posta adresi belirlemek üzere bir POST isteği gönderir.

```
redis-cli HGET user:user123 email
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/87w7na861mvmutpmnqir.png)


**handlers/handlers.go:**

```bash
package handlers

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
//    "os"
    "time"
    "strconv"

    "task/database"
    "task/models"

    "github.com/gofiber/fiber/v2"
    "github.com/go-redis/redis/v8"
)

// API endpointlerini Fiber uygulamasına kaydetmek için kullanılan fonksiyon.
func RegisterRoutes(app *fiber.App) {
    app.Post("/task", createTask)              // Yeni görev oluşturur
    app.Get("/task/:id", getTask)              // ID'ye göre görev bilgisi getirir
    app.Put("/task/:id", updateTask)           // Mevcut görevi günceller
    app.Delete("/task/:id", deleteTask)        // Görevi siler
    app.Get("/redis/keys", listRedisKeys)      // Redis'deki tüm anahtarları listeler
    app.Post("/session/:user_id", storeSession)// Kullanıcı oturum bilgisini kaydeder
    app.Get("/session/:user_id", checkSession) // Kullanıcı oturumunu kontrol eder
    app.Post("/enqueue", enqueueTask)          // Görevi message queue'ya ekler
    app.Get("/subscribe", subscribeToTasks)    // Görev güncellemelerine abone olur
    app.Get("/test-rate-limit", rateLimiter, func(c *fiber.Ctx) error {
        return c.SendString("Rate limit test başarılı!")
    })
    app.Post("/location/:user_id", storeLocation)  // Kullanıcı konumu kaydet
    app.Get("/nearby/:user_id", findNearby)       // Yakındaki kullanıcıları bul
    app.Post("/log-action/:user_id", logUserAction)  // Kullanıcı aksiyonunu kaydeder
    app.Get("/replay-session/:user_id", replaySession)  // Kullanıcının oturum aksiyonlarını geri döndürür
    app.Post("/update-profile/:user_id", updateProfile)  // Kullanıcı profili günceller


}

// HTTP POST request ile yeni görev oluşturur.
func createTask(c *fiber.Ctx) error {
    task := new(models.Task)
    if err := c.BodyParser(task); err != nil {
        return c.Status(400).SendString(err.Error())
    }
    task.CreationTime = time.Now()
    ctx := context.Background()
    _, err := database.PgConn.Exec(ctx, "INSERT INTO tasks (header, description, creation_time) VALUES ($1, $2, $3)", task.Header, task.Description, task.CreationTime)
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }
    publishTask(task) // Yeni görev oluşturulduğunda pub/sub kanalında yayınlar.
    return c.Status(201).JSON(task)
}

// ID'ye göre görev bilgisi getirir, öncelikle Redis cache kontrol edilir.
func getTask(c *fiber.Ctx) error {
    id := c.Params("id")
    ctx := context.Background()
    cachedTask, err := database.RedisClient.Get(ctx, "task:"+id).Result()
    if err == nil {
        task := new(models.Task)
        json.Unmarshal([]byte(cachedTask), task)
        return c.JSON(task)
    }
    task := new(models.Task)
    err = database.PgConn.QueryRow(ctx, "SELECT id, header, description, creation_time FROM tasks WHERE id = $1", id).Scan(&task.ID, &task.Header, &task.Description, &task.CreationTime)
    if err != nil {
        return c.Status(404).SendString("Görev bulunamadı")
    }
    jsonData, _ := json.Marshal(task)
    database.RedisClient.Set(ctx, "task:"+id, jsonData, 30*time.Minute) // Görev bilgisini Redis'e kaydeder.
    return c.JSON(task)
}

// Mevcut bir görevi günceller ve değişikliği pub/sub kanalında yayınlar.
func updateTask(c *fiber.Ctx) error {
    id := c.Params("id")
    task := new(models.Task)
    if err := c.BodyParser(task); err != nil {
        return c.Status(400).SendString(err.Error())
    }
    ctx := context.Background()
    _, err := database.PgConn.Exec(ctx, "UPDATE tasks SET header = $1, description = $2 WHERE id = $3", task.Header, task.Description, id)
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }
    publishTask(task)
    return c.SendStatus(fiber.StatusOK)
}

// Bir görevi veritabanından siler.
func deleteTask(c *fiber.Ctx) error {
    id := c.Params("id")
    ctx := context.Background()
    _, err := database.PgConn.Exec(ctx, "DELETE FROM tasks WHERE id = $1", id)
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }
    return c.SendStatus(fiber.StatusOK)
}

// Redis'te saklanan tüm anahtarları ve değerlerini listeler.
func listRedisKeys(c *fiber.Ctx) error {
    ctx := context.Background()
    keys, err := database.RedisClient.Keys(ctx, "*").Result()
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }
    var results []map[string]interface{}
    for _, key := range keys {
        val, err := database.RedisClient.Get(ctx, key).Result()
        if err != nil {
            continue
        }
        results = append(results, map[string]interface{}{key: val})
    }
    return c.JSON(results)
}

// Kullanıcı oturum bilgisini Redis'e kaydeder.
func storeSession(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    sessionData := map[string]interface{}{
        "last_login": time.Now(),
    }
    sessionJSON, _ := json.Marshal(sessionData)
    ctx := context.Background()
    database.RedisClient.Set(ctx, "session:"+userID, sessionJSON, 24*time.Hour)
    return c.SendStatus(fiber.StatusOK)
}

// Kaydedilmiş bir kullanıcı oturumunu kontrol eder.
func checkSession(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    ctx := context.Background()
    sessionData, err := database.RedisClient.Get(ctx, "session:"+userID).Result()
    if err != nil {
        return c.Status(401).SendString("Yetkisiz erişim")
    }
    return c.Status(200).SendString("Oturum geçerli: " + sessionData)
}

// Bir görevi Redis message queue'ya ekler.
func enqueueTask(c *fiber.Ctx) error {
    task := new(models.Task)
    if err := c.BodyParser(task); err != nil {
        return c.Status(400).SendString(err.Error())
    }
    taskData, _ := json.Marshal(task)
    ctx := context.Background()
    database.RedisClient.LPush(ctx, "task_queue", taskData)
    return c.Status(201).SendString("Görev kuyruğa eklendi")
}

// Message queue'dan görev çıkarıp işler, sürekli çalışan bir arka plan işlemidir.
func ProcessTasks() {
    ctx := context.Background()
    for {
        result, err := database.RedisClient.RPop(ctx, "task_queue").Result()
        if err == nil {
            task := new(models.Task)
            json.Unmarshal([]byte(result), task)
            fmt.Println("İşlenen görev ID:", task.ID) // İşlenen görevi loglar
        }
        time.Sleep(1 * time.Second)
    }
}

// Bir görevin bilgilerini Redis pub/sub kanalında yayınlar.
func publishTask(task *models.Task) {
    ctx := context.Background()
    taskData, err := json.Marshal(task)
    if err != nil {
        log.Printf("Görev verisi oluşturulurken hata: %v", err)
        return
    }
    err = database.RedisClient.Publish(ctx, "tasks", taskData).Err()
    if err != nil {
        log.Printf("Görev Redis'e yayınlanırken hata: %v", err)
    } else {
        log.Printf("Görev yayınlandı: %v", task.ID)
    }
}

// HTTP isteği ile pub/sub kanalına abone olur, ilk mesajı alınca yanıt döndürür.
func subscribeToTasks(c *fiber.Ctx) error {
    ctx := context.Background()
    pubsub := database.RedisClient.Subscribe(ctx, "tasks")
    defer pubsub.Close()

    ch := pubsub.Channel()
    select {
    case msg := <-ch:
        fmt.Println("Alınan görev:", msg.Payload)
        return c.SendString("Alınan mesaj: " + msg.Payload)
    case <-time.After(10 * time.Second): // 10 saniye sonra zaman aşımı
        return c.SendStatus(fiber.StatusRequestTimeout)
    }
}

// Kullanıcı için counter kontrolü ve artırma işlemi yaparak, belirli bir süre içinde belirli sayıda istek sınırını kontrol eder.
func rateLimiter(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    ctx := context.Background()
    key := "rate_limit:" + userID

    resp, err := database.RedisClient.Get(ctx, key).Result()
    if err == redis.Nil {
        database.RedisClient.Set(ctx, key, 1, time.Minute)
    } else if err != nil {
        return c.Status(500).SendString("Hız limiti kontrolü başarısız")
    } else {
        count, _ := strconv.Atoi(resp)
        if count >= 100 { // Örneğin, bir dakika içinde 100 istek sınırı
            return c.Status(429).SendString("Hız limiti aşıldı")
        }
        database.RedisClient.Incr(ctx, key)
    }

    return c.Next()
}

func storeLocation(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    lat, _ := strconv.ParseFloat(c.Query("lat"), 64)
    lon, _ := strconv.ParseFloat(c.Query("lon"), 64)
    ctx := context.Background()

    // Kullanıcının konumunu Redis geospatial set'e kaydet
    database.RedisClient.GeoAdd(ctx, "user_locations", &redis.GeoLocation{
        Name:      userID,
        Longitude: lon,
        Latitude:  lat,
    })
    return c.SendStatus(fiber.StatusOK)
}

func findNearby(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    ctx := context.Background()

    // Kullanıcıya en yakın 10 kullanıcıyı bul
    res, err := database.RedisClient.GeoRadiusByMember(ctx, "user_locations", userID, &redis.GeoRadiusQuery{
        Radius:    5000,
        Unit:      "m",
        Count:     10,
        Sort:      "ASC",
    }).Result()
    if err != nil {
        return c.Status(500).SendString("Failed to query nearby users")
    }

    return c.JSON(res)
}


func logUserAction(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    action := string(c.Body())  // Kullanıcının yaptığı eylem
    timestamp := time.Now().Unix()
    ctx := context.Background()

    // Eylemi Redis'e kaydet
    database.RedisClient.ZAdd(ctx, "session:"+userID, &redis.Z{
        Score:  float64(timestamp),
        Member: action,
    })
    fmt.Printf("Action logged: %s\n", action) // Log the action for debugging
    return c.SendStatus(fiber.StatusOK)
}


func replaySession(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    ctx := context.Background()

    // Kullanıcının tüm eylemlerini zaman damgası sırasına göre al
    actions, err := database.RedisClient.ZRange(ctx, "session:"+userID, 0, -1).Result()
    if err != nil {
        return c.Status(500).SendString("Failed to replay session")
    }

    return c.JSON(actions)
}

func updateProfile(c *fiber.Ctx) error {
    userID := c.Params("user_id")
    newEmail := c.FormValue("email")
    ctx := context.Background()

    // Email adresini güncellemek için transaction başlat
    tx := database.RedisClient.TxPipeline()
    tx.HSet(ctx, "user:"+userID, "email", newEmail)
    tx.Publish(ctx, "profile_updates", userID+" updated their email")
    _, err := tx.Exec(ctx)
    if err != nil {
        return c.Status(500).SendString("Failed to update profile")
    }

    return c.SendStatus(fiber.StatusOK)
}
```
