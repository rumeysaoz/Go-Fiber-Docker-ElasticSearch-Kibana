# Filebeat, Kafka, Logstash, Elasticsearch ve Kibana Entegrasyonu


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2nezsds8ewwazt74gh8v.png)


**Adım 1: Docker-Compose Dosyasını İndirme**

EFK Stack'i başlatmak için öncelikle ilgili Docker-Compose dosyasını indirin.


```
git clone https://github.com/eunsour/docker-efk.git
```

İndirilen dizine gidin:

```
cd docker-efk
```
**Adım 2: Docker-Compose ile EFK Stack'i Başlatma**

Docker-Compose kullanarak EFK Stack'i başlatın:

```
docker-compose up -d --build
```

**Adım 3: İzin Ayarları**

Elasticsearch tarafından oluşturulan ve kullanıcının okuma/yazma izinlerine sahip olması gereken log ve veri dizinleri için izinleri ayarlayın.


```
sudo chown -R $USER:$USER ./elasticsearch/logs
sudo chmod -R 755 ./elasticsearch/logs
sudo chown -R $USER:$USER ./elasticsearch/data
sudo chmod -R 755 ./elasticsearch/data
```
 
Daha sonra **http://X.X.X.X:5601/** adresine giderek gerekli işlemleri yapabilirsiniz.
