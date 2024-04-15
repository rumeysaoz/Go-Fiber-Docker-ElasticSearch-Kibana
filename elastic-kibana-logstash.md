# Go + Fiber + Docker + ElasticSearch + Logstash + Kibana



# Adım 1: Go ve Fiber ile Basit Bir API Oluşturma

**_!!_** Go kurulu olmalıdır. 

<u>Go Kurulumu</u>

- İlk olarak, Go'nun resmi web sitesinden golang.org/dl üzerinden Go'nun son sürümünü indirin. Örneğin, go1.18.linux-amd64.tar.gz dosyasını indirin.

- Terminali açın ve aşağıdaki komutlarla Go'yu çıkarın:

```
rm -rf /usr/local/go 
wget https://golang.org/dl/go1.18.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz
```
**GOROOT**, Go'nun kurulu olduğu dizindir ve bu örnekte /usr/local/go olarak belirlenmiştir. **GOPATH**, Go projelerinizin ve bağımlılıklarının saklandığı yerdir ve bu örnekte /home/ubuntu/go olarak ayarlanacaktır.

- Bu değişkenleri ayarlamak için .profile dosyanızı düzenleyin:

```
echo "export GOROOT=/usr/local/go" >> ~/.profile
echo "export GOPATH=/home/ubuntu/go" >> ~/.profile
echo "export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin" >> ~/.profile
```

- Bu ayarları uygulamak için oturumu kapatıp açın veya aşağıdaki komutu çalıştırın:

```
source ~/.profile
```

- Go'nun başarıyla kurulduğunu ve ortam değişkenlerinin doğru ayarlandığını doğrulamak için terminalde şu komutları çalıştırın:

```
go version
echo $GOROOT
echo $GOPATH
```

Bu komutlar, sırasıyla Go'nun sürümünü, GOROOT ve GOPATH değerlerini göstermelidir.

<u>Go Uygulaması Oluşturma</u>

- Proje Dizini Oluşturma ve İnitialize Etme:

```
mkdir go-fiber-task && cd go-fiber-task
go mod init go-fiber-task
```

- Fiber ve Diğer Gerekli Kütüphaneleri Ekleme:

```
go get -u github.com/gofiber/fiber/v2
```
- ElasticSearch Go İstemci Kütüphanesini Ekleyin:

Terminalde aşağıdaki komutu çalıştırarak ElasticSearch Go kütüphanesini projenize ekleyin.

```
go get github.com/elastic/go-elasticsearch/v7
```

- Go Dosyalarını Oluşturma:

**Model Katmanı:** Veri modellerini ve yapılarını tanımlar.

**Router Katmanı:** HTTP yönlendirmelerini ve endpoint tanımlarını içerir.

**Handler Katmanı:** HTTP isteklerini işler ve yanıtlarını döndürür.

**Service Katmanı:** İş mantığını içerir.

**Repository Katmanı:** Veritabanı işlemlerini gerçekleştirir.

_**1. Model Katmanı (models/task.go)**_

```bash
package models

import "time"

// Task yapısını tanımla. Bir Task, bir ID, başlık, açıklama ve oluşturulma zamanını içerir.
type Task struct {
    ID          string    `json:"id"`
    Header      string    `json:"header"`
    Description string    `json:"description"`
    CreationTime time.Time `json:"creation_time"`
}
```

_**2. Service Katmanı (services/taskService.go)**_

```bash
package services

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "github.com/elastic/go-elasticsearch/v7"
    "go-fiber-task/models" 
    "log"
//    "net/http"
//    "strconv"
    "sync/atomic"
    "time"
)

var taskID int64

// CreateTask, bir task'ı ElasticSearch'e kaydeder.
func CreateTask(es *elasticsearch.Client, task *models.Task) error {
    newID := atomic.AddInt64(&taskID, 1)
    task.ID = fmt.Sprintf("%d", newID)
    task.CreationTime = time.Now()

    jsonTask, err := json.Marshal(task)
    if err != nil {
        return err
    }

    res, err := es.Index("tasks", bytes.NewReader(jsonTask), es.Index.WithDocumentID(task.ID), es.Index.WithRefresh("true"))
    if err != nil || res.IsError() {
        return fmt.Errorf("error indexing task: %s", res.String())
    }
    return nil
}

// GetAllTasks, ElasticSearch'teki tüm task'ları listeler.
func GetAllTasks(es *elasticsearch.Client) ([]models.Task, error) {
    var r map[string]interface{}
    res, err := es.Search(
        es.Search.WithContext(context.Background()),
        es.Search.WithIndex("tasks"),
        es.Search.WithSize(1000), // Maksimum 1000 task getir
    )
    if err != nil || res.IsError() {
        return nil, fmt.Errorf("error getting tasks: %s", res.String())
    }
    defer res.Body.Close()

    if err := json.NewDecoder(res.Body).Decode(&r); err != nil {
        return nil, err
    }

    hits := r["hits"].(map[string]interface{})["hits"].([]interface{})
    tasks := make([]models.Task, len(hits))

    for i, hit := range hits {
        source := hit.(map[string]interface{})["_source"]
        sourceBytes, _ := json.Marshal(source)
        var task models.Task
        if err := json.Unmarshal(sourceBytes, &task); err != nil {
            log.Printf("Failed to unmarshal source: %v", err)
            continue
        }
        tasks[i] = task
    }

    return tasks, nil
}

// UpdateTask, belirli bir ID'ye sahip task'ı günceller.
func UpdateTask(es *elasticsearch.Client, id string, taskUpdates models.Task) error {
    // Sadece güncellenmek istenen alanları belirleyin.
    // Örneğin: 'header' ve 'description' alanlarını güncellemek istiyoruz.
    updateDoc := map[string]interface{}{
        "doc": map[string]interface{}{
            "header":      taskUpdates.Header,
            "description": taskUpdates.Description,
        },
    }

    // Güncelleme için JSON'a dönüştür
    reqBody, err := json.Marshal(updateDoc)
    if err != nil {
        return err
    }

    // Elasticsearch güncelleme API'sini kullanarak güncelleme işlemini yapın
    res, err := es.Update(
        "tasks", // Index adı
        id,      // Güncellenecek dokümanın ID'si
        bytes.NewReader(reqBody), // Güncelleme için JSON içeriği
        es.Update.WithContext(context.Background()),
        es.Update.WithRefresh("true"), // Güncelleme sonrası index'i yenile
    )
    if err != nil || res.IsError() {
        return fmt.Errorf("error updating task: %s", res.String())
    }
    return nil
}

// DeleteTask, belirli bir ID'ye sahip task'ı siler.
func DeleteTask(es *elasticsearch.Client, id string) error {
    res, err := es.Delete("tasks", id, es.Delete.WithRefresh("true"))
    if err != nil || res.IsError() {
        return fmt.Errorf("error deleting task: %s", res.String())
    }
    return nil
}
```

_**3. Handler Katmanı (handlers/taskHandler.go)**_

```bash
package handlers

import (
    "github.com/gofiber/fiber/v2"
    "go-fiber-task/models"
    "go-fiber-task/services"
    "github.com/elastic/go-elasticsearch/v7"
    "net/http"
//    "strconv"
)

func RegisterTaskRoutes(app *fiber.App, es *elasticsearch.Client) {
    app.Post("/task", func(c *fiber.Ctx) error {
        var task models.Task
        if err := c.BodyParser(&task); err != nil {
            return c.Status(fiber.StatusBadRequest).SendString(err.Error())
        }

        if err := services.CreateTask(es, &task); err != nil {
            return c.Status(http.StatusInternalServerError).SendString(err.Error())
        }

        return c.Status(fiber.StatusOK).JSON(task)
    })

    app.Get("/tasks", func(c *fiber.Ctx) error {
        tasks, err := services.GetAllTasks(es)
        if err != nil {
            return c.Status(http.StatusInternalServerError).SendString(err.Error())
        }

        return c.Status(fiber.StatusOK).JSON(tasks)
    })

    app.Put("/task/:id", func(c *fiber.Ctx) error {
        id := c.Params("id")
        var taskUpdates models.Task
        if err := c.BodyParser(&taskUpdates); err != nil {
            return c.Status(fiber.StatusBadRequest).SendString(err.Error())
        }

        if err := services.UpdateTask(es, id, taskUpdates); err != nil {
            return c.Status(http.StatusInternalServerError).SendString(err.Error())
        }

        return c.SendStatus(fiber.StatusOK)
    })

    app.Delete("/task/:id", func(c *fiber.Ctx) error {
        id := c.Params("id")

        if err := services.DeleteTask(es, id); err != nil {
            return c.Status(http.StatusInternalServerError).SendString(err.Error())
        }

        return c.SendStatus(fiber.StatusOK)
    })
}
```

_**4. Main Dosyası (main.go)**_

```bash
package main

import (
    "github.com/gofiber/fiber/v2"
    "go-fiber-task/handlers"
    "github.com/elastic/go-elasticsearch/v7"
    "log"
)

func main() {
    es, err := elasticsearch.NewDefaultClient()
    if err != nil {
        log.Fatalf("Error creating the client: %s", err)
    }

    app := fiber.New()
    handlers.RegisterTaskRoutes(app, es)

    log.Fatal(app.Listen(":3000"))
}
```

- Uygulamayı Çalıştırma:

```
go run main.go
```

Terminalde aşağıdaki komutu çalıştırarak uygulamanızı başlatın.
Artık http://localhost:3000 adresine giderek API'nize ulaşabilirsiniz.

# Adım 2: Docker ile Uygulamayı Konteynerize Etme

**_!!_** Docker ve Docker-Compose kurulu olmalıdır. [Şu link](https://dev.to/aciklab/docker-ve-docker-compose-kurulumu-ubuntu-2004-3ch7) ile kurulum gerçekleştirilebilir.

- Dockerfile Oluşturma:

Proje kök dizininde, Dockerfile adında bir dosya oluşturun ve aşağıdaki içeriği ekleyin.

```bash
# Golang 1.18 sürümünü kullanarak bir Alpine Linux tabanlı builder imajı oluştur.
FROM golang:1.18-alpine AS builder

# /app dizinini çalışma dizini olarak ayarla.
WORKDIR /app

# Go modül desteğini etkinleştir.
ENV GO111MODULE=on

# Mevcut dizindeki tüm dosyaları (bu Dockerfile'ın bulunduğu yerdeki) /app dizinine kopyala.
COPY . .

# Go ile main.go dosyasını derle ve 'main' adında bir çalıştırılabilir dosya oluştur.
# CGO'yu devre dışı bırak ve statik olarak bağla.
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -installsuffix cgo -o main .

# Yeni bir aşama başlat: Hafif bir Alpine Linux tabanlı imaj kullan.
# Alpine imajına ca-certificates eklemek, HTTPS üzerinden bağlantı kurarken SSL sertifikası doğrulama sorunlarını önler.
FROM alpine:latest  
RUN apk --no-cache add ca-certificates

# /app dizinini çalışma dizini olarak ayarla.
WORKDIR /app

# Builder aşamasından elde edilen 'main' çalıştırılabilir dosyasını şu anki aşamaya (/app dizinine) kopyala.
COPY --from=builder /app/main .

# Konteynerin 3000 numaralı portunu dış dünya ile paylaş.
EXPOSE 3000

# Konteyner çalıştırıldığında 'main' komutunu çalıştır (Bu, Go uygulamanızı başlatır).
CMD ["./main"]
```

- Docker Image'ını Oluşturma ve Çalıştırma:

```
# 'go-fiber-task' adında bir Docker image'ı oluştur. Bu işlem, yukarıdaki Dockerfile'ı kullanır.
docker build -t go-fiber-task .

# Oluşturulan 'go-fiber-task' image'ını bir konteyner olarak başlat.
# Konteynerin 3000 numaralı portunu, host makinenin 3001 numaralı portuna yönlendir.
# Bu sayede, host makineden konteynerde çalışan uygulamaya ulaşılabilir.
docker run -p 3001:3000 go-fiber-task
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fpay9m8exnz4e7prw6e5.png)

# Adım 3: ElasticSearch, Logstash ve Kibana ile Entegrasyon

- Pipeline Dizini Oluşturma:

Docker Compose dosyanızın bulunduğu dizinde, Logstash için bir pipeline dizini oluşturun. Bu dizin, Logstash konfigürasyon dosyanızı içerecek:

```
mkdir -p ./logstash/pipeline
```
Bu komut, logstash adında bir dizin ve içinde pipeline adında bir alt dizin oluşturur.

- Logstash Konfigürasyon Dosyası Oluşturma:

pipeline dizinine geçin:

```
cd ./logstash/pipeline
```
logstash/pipeline dizinindeki logstash.conf dosyasını açın ve düzenleyin:

```
nano logstash.conf
```
stdin input plugin'ini kaldırın ve dosya input plugin'ini aşağıdaki gibi ekleyin:

```
input {
  file {
    path => "/usr/share/logstash/data/test.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```
Bu konfigürasyon, Logstash'in /usr/share/logstash/data/test.log dosyasındaki yeni satırları okumasını sağlar. sincedb_path => "/dev/null" ayarı, Logstash'in dosyayı her okumasında baştan başlamasını sağlar, bu da test amaçlı kullanışlıdır.

pipeline dizininden çıkıp Docker Compose dosyanızın bulunduğu ana dizine geri dönün:

```
cd ..
cd ..
```
- docker-compose.yml Dosyası Oluşturma:
Projeye bir docker-compose.yml dosyası ekleyin ve aşağı içeriği ekleyin. Bu dosya ElasticSearch, Logstash ve Kibana'yı aynı anda çalıştırmanızı sağlayacak.

```bash
version: '3.8'  # Docker Compose dosyasının versiyonunu belirtir. Bu, kullanılan özelliklerin hangi Docker Compose sürümleriyle uyumlu olduğunu gösterir.

services:
  elasticsearch:
    container_name: elasticsearch  # Oluşturulan Docker konteyneri için belirli bir isim atar.
    image: docker.elastic.co/elasticsearch/elasticsearch:7.9.2  # ElasticSearch için kullanılacak Docker imajı. Elastic.co'nun resmi ElasticSearch 7.9.2 sürümünü kullanır.
    environment:
      - discovery.type=single-node  # ElasticSearch'in tek düğüm (node) modunda çalışmasını sağlar.
    ports:
      - "9200:9200"  # Konteynerin 9200 numaralı portunu dış dünya ile eşleştirir. Bu, ElasticSearch'e dışarıdan erişilebilir olmasını sağlar.

  kibana:
    container_name: kibana  # Oluşturulan Docker konteyneri için belirli bir isim atar.
    image: docker.elastic.co/kibana/kibana:7.9.2  # Kibana için kullanılacak Docker imajı. Elastic.co'nun resmi Kibana 7.9.2 sürümünü kullanır.
    ports:
      - "5601:5601"  # Kibana'nın bağlı olduğu port. Kibana arayüzüne tarayıcı üzerinden 5601 portu kullanılarak erişilebilir.
    depends_on:
      - elasticsearch  # Kibana servisinin, elasticsearch servisine bağımlı olduğunu belirtir. ElasticSearch konteyneri başlatıldıktan sonra Kibana konteyneri başlatılır.

  logstash:
    container_name: logstash  # Oluşturulan Docker konteyneri için belirli bir isim atar.
    image: docker.elastic.co/logstash/logstash:7.9.2  # Logstash için kullanılacak Docker imajı. Elastic.co'nun resmi Logstash 7.9.2 sürümünü kullanır.
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro  # Host sistemdeki ./logstash/pipeline dizinini, konteyner içindeki /usr/share/logstash/pipeline dizinine salt-okunur olarak bağlar. Bu dizin Logstash pipeline konfigürasyon dosyalarını içerir.
      - ./logstash/data:/usr/share/logstash/data  # Host sistemdeki ./logstash/data dizinini, konteyner içindeki /usr/share/logstash/data dizinine bağlar. Bu dizin, Logstash tarafından işlenecek veri dosyalarını içerebilir.
    depends_on:
      - elasticsearch  # Logstash servisinin, elasticsearch servisine bağımlı olduğunu belirtir. ElasticSearch konteyneri başlatıldıktan sonra Logstash konteyneri başlatılır.
```
Bu yapılandırma ile ElasticSearch, Kibana ve Logstash servislerini Docker Compose ile yönetebilirsiniz. ElasticSearch ve Kibana, log verilerini analiz etmek ve görselleştirmek için kullanılırken, Logstash logları işleyip ElasticSearch'e göndermek için kullanılır.

Yerel makinenizde ./logstash/data dizini oluşturun ve içine test.log adında bir dosya ekleyin:

```
mkdir -p ./logstash/data
```
Bu komut, ./logstash/data dizininde test.log adında bir dosya oluşturur ve içine "Hello, Logstash!" metni yazar.

- Docker Compose ile Servisleri Başlatma:

Aşağıdaki komut ile Docker Compose aracılığıyla tanımlanan servisleri başlatın.

```
docker-compose up -d
```
Bu komut, tüm servisleri arka planda çalışacak şekilde başlatır. ElasticSearch http://localhost:9200 adresinde, Kibana http://localhost:5601 adresinde erişilebilir olacak, ve Logstash log dosyalarını işlemeye başlayacaktır.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4qmrnlw8j0fcgrswf0m5.png)

- Dizin İzinlerini Kontrol Edin:

Host makinenizde, Logstash'in veri dizinine yazma izinlerinin olup olmadığını kontrol edin. Örneğin:

```
ls -l ./logstash/data
```

Ve eğer gerekliyse, yazma izinlerini ayarlayın:

```
chmod -R 777 ./logstash/data
```

**Not:** chmod 777 komutu, tüm kullanıcılara dizin üzerinde tam izinler verir. Bu, üretim ortamları için önerilen bir yöntem değildir. Ancak, test amaçlı yerel bir geliştirme ortamında ve sorunun kaynağını belirlemek için geçici bir çözüm olarak kullanılabilir.

- Test ve Doğrulama:

Dosya İçeriğini Güncelleyin: Logstash'in izlediği test.log dosyasına yeni satırlar ekleyerek Logstash'in bu değişiklikleri algılayıp algılamadığını kontrol edin. Örneğin:

```
echo "Hello, Logstash!" >> ./logstash/data/test.log
```

ElasticSearch'e indekslenen verileri kontrol etmek için şu curl komutunu kullanabilirsiniz:

```
curl -X GET "http://localhost:9200/logstash-*/_search?pretty&q=*:*"
```
Bu komut, Logstash tarafından oluşturulan tüm indekslerdeki tüm dokümanları listeler. q=*:* sorgusu, tüm dokümanları eşleştirir. Eğer Logstash üzerinden veri başarıyla işlendi ve ElasticSearch'e gönderildi ise, bu sorgu sonucunda verilerinizi görmelisiniz.

# Adım 4: Postman ve ElasticSearch-Kibana ile Test Etme

**Postman ile Test Etme:**

<u>Yeni Task Ekleme:</u>

- Method: POST
- URL: http://localhost:3000/task
- Body Type: JSON
- Body:

```
{
  "header": "Yeni Görev",
  "description": "Görev açıklaması"
}
```
Bu istek, yeni bir **Task** ekler ve ElasticSearch'e kaydeder. Yeni **Task** için ID otomatik olarak artan bir sayısal değer olarak atanır ve kullanıcı tarafından belirtilmesine gerek yoktur.

<u>Tüm Taskları Listeleme:</u>

- Method: GET
- URL: http://localhost:3000/tasks
Bu request ElasticSearch'teki tüm **Task** kayıtlarını listeler.

<u>Task Güncelleme:</u>

- Method: PUT
- URL: http://localhost:3000/task/[TaskID] ([TaskID] yerine güncellenecek task'ın ID'si gelmelidir.)
- Body Type: JSON
Body:

```
{
  "header": "Güncellenmiş Görev Başlığı",
  "description": "Güncellenmiş açıklama"
}
```
Bu request belirli bir ID'ye sahip Task'ı günceller.

<u>Task Silme:</u>

- Method: DELETE
- URL: http://localhost:3000/task/[TaskID] ([TaskID] yerine silinecek task'ın ID'si gelmelidir.)
Bu request belirli bir ID'ye sahip Task'ı siler.

<u>ElasticSearch-Kibana ile Test Etme:</u>

ElasticSearch ve Kibana kullanarak yukarıda oluşturduğunuz ve işlediğiniz verileri test etmek ve görselleştirmek için aşağıdaki adımları izleyebilirsiniz.


# Adım 5: ElasticSearch'te Veri Sorgulama:

ElasticSearch'te verileri doğrudan sorgulamak için curl komutunu kullanabilirsiniz. Aşağıdaki örnekte, tasks indeksinde saklanan tüm belgeleri listelemek için bir sorgu yer almaktadır:

```
curl -X GET "http://localhost:9200/tasks/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  }
}'
```
Bu komut, **tasks** indeksindeki tüm **Task** nesnelerini listeler ve **pretty** parametresi sayesinde JSON çıktısını okunabilir biçimde formatlar.


# Adım 6: Kibana ile Görselleştirme ve Sorgulama

ElasticSearch üzerindeki verilerinizi Kibana aracılığıyla sorgulamak ve görselleştirmek için Kibana'nın web arayüzünü kullanabilirsiniz.

<u>Kibana'yı Açın:</u>

- Web tarayıcınızda http://localhost:5601 adresine giderek Kibana'yı açın. Bu, varsayılan Kibana portudur.

<u>Index Pattern Oluşturun:</u>

- Kibana'nın sol menüsünden "Stack Management" > "Index Patterns" seçeneğine gidin.
- "Create index pattern" butonuna tıklayın ve tasks indeksini index pattern olarak ekleyin. Bu adım, Kibana'nın ElasticSearch'teki tasks indeksinden veri çekmesini sağlar.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zu6yhk5lcyqs7swnth50.png)


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9rck6rzly092zww57ws7.png)


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gg3jteox12m8hnvczp88.png)



![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k3aigzdaaj07jqu7j537.png)


<u>Discover Kullanarak Verileri Görüntüleme:</u>

- Sol menüden "Discover" seçeneğine tıklayın.
- Oluşturduğunuz **tasks** index pattern'ını seçin.
- Bu sayfada, ElasticSearch'teki **Task** nesneleri üzerinde çeşitli sorgular yapabilir ve sonuçları inceleyebilirsiniz.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i9h4nk6txococzd2aycg.png)

<u>Visualize ve Dashboard ile Görselleştirme:</u>

- Kibana'da "Visualize" ve "Dashboard" özelliklerini kullanarak verilerinizi çeşitli grafik ve görsellerle analiz edebilirsiniz.
- "Visualize" bölümüne giderek yeni bir görselleştirme oluşturun, ardından oluşturduğunuz görselleştirmeyi bir dashboard'a ekleyebilirsiniz.

