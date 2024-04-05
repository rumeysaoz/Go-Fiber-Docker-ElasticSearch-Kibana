# Go + Fiber + Redis + Postgresql + Docker


# Go ve Fiber ile Basit Bir API Oluşturma

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

#Go Uygulaması oluşturma:
```
mkdir task && cd task
go mod init task
go get github.com/gofiber/fiber/v2
go get github.com/go-redis/redis/v8
go get github.com/jackc/pgx/v4
```
# main.go dosyası oluşturma:
nano main.go
```
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"github.com/gofiber/fiber/v2"
	"github.com/go-redis/redis/v8"
	"github.com/jackc/pgx/v4"
	"os"
	"time"
)

// Task modeli
type Task struct {
	ID          int64     `json:"id"`
	Header      string    `json:"header"`
	Description string    `json:"description"`
	CreationTime time.Time `json:"creation_time"`
}

var pgConn *pgx.Conn
var redisClient *redis.Client

func initPostgres() {
	var err error
	pgConn, err = pgx.Connect(context.Background(), "postgres://your_username:your_password@localhost:5432/taskdb")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Unable to connect to PostgreSQL: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("Connected to PostgreSQL")
}

func initRedis() {
	redisClient = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})

	_, err := redisClient.Ping(context.Background()).Result()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Unable to connect to Redis: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("Connected to Redis")
}

func createTask(c *fiber.Ctx) error {
	task := new(Task)

	if err := c.BodyParser(task); err != nil {
		return c.Status(400).SendString(err.Error())
	}

	task.CreationTime = time.Now()

	_, err := pgConn.Exec(context.Background(), "INSERT INTO tasks (header, description, creation_time) VALUES ($1, $2, $3)", task.Header, task.Description, task.CreationTime)
	if err != nil {
		return c.Status(500).SendString(err.Error())
	}

	return c.Status(201).JSON(task)
}

func getTask(c *fiber.Ctx) error {
    id := c.Params("id")
    ctx := context.Background()

    // Öncelikle Redis'te arayalım
    val, err := redisClient.Get(ctx, id).Result()
    if err == nil {
        // Cache'te bulundu
        task := new(Task)
        if err := json.Unmarshal([]byte(val), task); err != nil {
            return c.Status(500).SendString(err.Error())
        }
        return c.JSON(task)
    }

    // Redis'te bulunamazsa, PostgreSQL'den arayalım
    task := new(Task)
    err = pgConn.QueryRow(ctx, "SELECT id, header, description, creation_time FROM tasks WHERE id = $1", id).Scan(&task.ID, &task.Header, &task.Description, &task.CreationTime)
    if err != nil {
        return c.Status(404).SendString("Task not found")
    }

    // Sonucu Redis'e kaydedelim
    jsonData, err := json.Marshal(task)
    if err != nil {
        return c.Status(500).SendString(err.Error())
    }
    redisClient.Set(ctx, fmt.Sprintf("%d", task.ID), string(jsonData), 30*time.Minute) // 30 dakika boyunca cache'le

    return c.JSON(task) // Fonksiyonun sonunda başarıyla Task'ı JSON olarak döndür
}

func main() {
	initPostgres()
	initRedis()

	app := fiber.New()

	app.Post("/task", createTask)
	app.Get("/task/:id", getTask)

	app.Listen(":3000")
}


```
# Dockerfile oluşturma:
nano Dockerfile
```
# Go'nun resmi base image'ını kullanarak başlayın
FROM golang:1.18 AS builder

# Çalışma dizinini ayarlayın
WORKDIR /app

# Go mod dosyalarını kopyalayın ve bağımlılıkları indirin
COPY go.mod go.sum ./
RUN go mod download

# Kaynak kodunu kopyalayın
COPY . .

# Uygulamayı build edin
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# İkinci stage, uygulamayı çalıştırmak için hafif bir base image kullanın
FROM alpine:latest  
RUN apk --no-cache add ca-certificates

WORKDIR /root/

# İlk stage'den build edilen executable'ı kopyalayın
COPY --from=builder /app/main .

# Uygulamanın çalışacağı portu belirtin
EXPOSE 3000

# Uygulamayı çalıştırın
CMD ["./main"]
```
# Docker ile Uygulamayı Konteynerize Etme

**_!!_** Docker ve Docker-Compose kurulu olmalıdır. [Şu link](https://dev.to/aciklab/docker-ve-docker-compose-kurulumu-ubuntu-2004-3ch7) ile kurulum gerçekleştirilebilir.
**-docker-compose.yml dosyası oluşturma:**

nano docker-compose.yml
```
version: '3.8'

services:
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: taskdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:latest
    ports:
      - "6379:6379"

volumes:
  postgres_data:

```
# Docker konteynerlerini oluşturma:

```
docker-compose up --build
```
Bu işlem sonrasında bizim için gerekli olan redis ve postgre konteynerlerini ayağa kaldıracağız. Sonrasında ise uygulamayı çalıştırmak için aşağıdaki terminal kodunu çalıştıracağız:

```
go run main.go
```
# Şimdi sıra ayağa kaldırdığımız uygulamanın test işlemleri için gerekli Postman sorgularını kullanmaya geldi.

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
Bu istek, yeni bir **Task** ekler PostgreSql'e kaydeder.

<u>Task id ile spesifik bir taskı listeleme</u>

- Method: GET
- URL: http://localhost:3000/task/{id}


<u>Task Güncelleme:</u>
- Method: PUT
- URL: http://localhost:3000/task/{id}

```
{
  "header": "Yeni Görev güncelleme",
  "description": "Görev açıklaması güncelleme"
}
```

<u>Task Silme:</u>
- Method: DELETE
- URL: http://localhost:3000/task/{id}

<u>Yapılan işlemlerin ardından rediste tutulan bazı logların listelenmesi:</u>

- Method: get
- URL: http://localhost:3000/redis/key

