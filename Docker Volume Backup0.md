# Docker Volume Backup


[!dockervolume](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*xONk464vW-xNYxzE_HsSkw.png)

Docker volume, containerlarınızda sakladığınız verileri kalıcı hale getirmek için kullanabileceğiniz bir yöntemdir. Docker volume ile verileriniz host sistemi üzerinde fiziksel olarak ayrı bir yerde depolanır ve containerlar tarafından erişilebilir. Docker volume kullanmanın faydaları arasında verileri yedeklemek ve geri yüklemek, verileri daha iyi yönetmek, containerları daha güvenli ve stabil hale getirmek sayılabilir.


## Bu eğitimde yapacaklarımız ise kısa şöyle özetlenebilir;

- Bir docker-compose.yml dosyası ile 2 adet nginx,1 adet jareware ve 1 adet minio container oluşturacağız.
- Bu 2 adet nginx serverlarımızın geliştire yapılacak olan dosyalarını BBK adında bir volume'a bağlayacağız
- Ardından jareware/docker-volume-backup containerımız ile bu volume'ü önce bu containerın üstüne
- Sonrasında eşlenik bir dizinle tekrar bu containerımızdan local makinemizin üstüne alacağız.
- Son olarak ise minio containerımıza bağlanıp bir bucket oluşturduktan sonra, volume back upımızı bazı komutlarla bu bucket'ın içine atacağız.

## Giriş
Eğitim boyunca birçok farklı komut kullanacağız, fakat bu eğitim özelinde ana nokta bu yedekleme işleminin neden yapıldığını anlamak, ve bu doğrultuda dockor-compose.yml dosyasının inceleyerek oluşturduğumuz yapının anlaşılması olacaktır.Şimdi eğitime başlayabiliriz.


## Docker-compose dosyamızın oluşturulması

Docker-compose.yml dosyası, birden fazla Docker container’ını tek bir dosyada tanımlayarak yönetmeyi sağlayan bir orkestrasyon aracıdır. Bu dosya, container’ların nasıl oluşturulacağını, nasıl bağlanacağını, nasıl çalıştırılacağını ve nasıl ölçekleneceğini belirtir.

Önce bulunduğumuz dizinin altında touch komutu ile dosyamızı oluşturuyoruz

```sh
touch docker-compose.yml
```

Sonrasında editör veya vim komutu aracılığı ile dosyanın içine giriyoruz.(Editörden yapılması tavsiye edilir)

```sh
vim docker-compose.yml 
```

Bu noktada işlemleri teker teker yazmak yerine size verilen komutları yazmanız sonrasında ise bunun üstünden yorumlamamız daha doğru olacaktır.Bu doğrultuda aşağıda size vermiş olduğumuz komutları docker-compose.yml dosyanızın içine yapıştırabilirsiniz.

```sh
version: '3.0'

services:
  minio:
    image: minio/minio:latest
    container_name: minio
    networks:
      - ozanet
    ports:
      - 5000:5000
    environment:
      MINIO_ROOT_USER: ozancannn
      MINIO_ROOT_PASSWORD: ozancannn
      MINIO_ACCESS_KEY: ozancannn
      MINIO_ACCESS_SECRET: ozancannn
    command: server /data --console-address ":5000"

  web1:
    image: nginx:latest
    ports:
      - "1001:80"
    volumes:
      - bbk:/usr/share/nginx/html
    networks:
      - ozanet
  web2:
    image: nginx:latest
    ports:
      - "1002:80"
    volumes:
      - bbk:/usr/share/nginx/html
    networks:
      - ozanet

  backup:
    image: jareware/docker-volume-backup
    volumes:
      - bbk:/backup/nginx-data:ro
      - /backups:/archive
    networks:
      - ozanet
    environment:
      BACKUP_CRON_EXPRESSION: "* * * * *"   
    
volumes:
  bbk:

networks:
  ozanet:
   driver: bridge   
```
Şimdi yukarıda oluşturmuş olduğumuz docker-compose.yml dosyamızın mantığını açıklayabiliriz.
- 
 - Minio, açık kaynaklı bir nesne depolama (object storage) sistemidir.Yedeğini alacağımız volume'u burada oluşturduğumuz bir bucket içinde saklayacağız.
 - Nginx, bir web sunucu ve ters proxy sunucu yazılımıdır.Burada bulunan 2 adet nginx sunucularımızın /usr/share/nginx/html dizinlerini bbk volume'na bağladık
 - Docker volume'larını yedekleme işlemini sağlayan bir araçtır.Burda yaptığımız ise önce jareware containerımızın nginx-data dizinini sadece read only olarak bbk volume'ına bağlamak böylelikle o volume'a nginx serverler tarafından yazılan dosyaları görüp archive dizinine yedekleyebilecek.Sonrasında ise archive dizinimizi ise tekrar lokal makinemizdeki backups dizinine eşleyerek backup'ımızı hem jareaware üstüne hemde lokal makinemize almış olduk.Burdaki BACKUP_CRON_EXPRESSION: "* * * * *" işlemin hemen sonuç alması için her dakika şeklinde ayarlanmıştır.Yani ' * * * * * ' yazarak dakikada birkere backup al demiş olduk.Bu işlem tamamlandığında iki farklı yerde backup dosyamız bulunuyor hale geldi.Şimdi ise bu dosyamızı minio s3 bucket'ımıza aktaracağız.








## Volume Backuplarımız Minio Bucket'a aktarma
Artık backuplarımızı aldığımıza göre bunları S3 bucketlarımıza atabiliriz.Bunun için minio containerımıza shellden bağlanıp içinde bir bucket oluşturmamız gerekiyor.Shellden bağlanmak için minio container id'sine ihtiyacımız.Öğrenmek için;

```sh 
docker ps
```
komutunu çalıştırıp burdan minio id'sini alıyoruz.

Sonrasında container bağlanmak için;
```sh 
docker exec -it <container id> bash
```
komutunu çalıştırıyoruz.Artık minio'ya bağlı durumdayız.Şimdi bir bucket oluşturmamız gerekiyor bunun için önce;
```sh 
cd /data
```
komutu ile data dizinin içine gidip sonrasında ise;
```sh 
mc mb bucket0
```
komutu ile bucket0 adında bir bucket oluşturmuş oldum.

Şimdi ise artık istediğim backup dosyamı önce minio containerinin içine,sonrasında ise bucket içine atacağız.Şimdi yeni bir terminal açıp o terminalde lokalde backuplarımızın tutulduğu backup dizine;
```sh 
cd /backups
``` 
komutu ile gidip bu dizinin içinde listele yani;
```sh 
ls
```
komutunu çalıştırdığımız zaman alınan backuplarımızı görmüş olacağız.
Burdan sonrasında minio s3 bucketımıza atmak istediğimiz  backup'ı belirleyip öncelikle minio data dizinin içine sonrasında ise ordanda bucket0 içine atacağız.Bunun için;
```sh 
docker cp /backups/ilkbackup.tar.gz <miniocontainerid>:/data
```
komutu ile backup dosyamızı minio containerin içindeki data dizinine atmış olduk.
Ardından önce;
```sh
docker exec -it <container id> bash
```
komutu ile minio container'ına bağlanıp;
```sh
cd /data
```
ile data dizinine gelmiş oldum.Bucket0'da backup dosyamızda bu dizinin altında.Artık tek yapmamız gereken backup dosyamızı bu bucket0'nun içine atmak.Bunun için;
```sh
mc cp ilkbackup.tar bucket0 
```
komutunu çalıştırıyorum ve dosyamı bucket içine atmış oluyorum.

Kontrol etmek için;
```sh
cd bucket0
```
ile bucket içine girdikten sonra;
```sh
ls
```
ile dosyamı görebilirim.

Tüm işlemler tamamlandı ve sonuç olarak volume backup dosyam jahaware containerinin /archive dizini,lokal makinamın /backups dizini ve minio s3 contaınerımda /data/bucket0 içinde olmak üzere üç farklı lokasyonda tutuluyor hale geldi.

## Son
Sizlere bu senaryorda backup mantığını ve container mimarisinde nasıl kullanılabileceğine dair bir örneğini anlatmaya çalıştım.Bu mantık birçok farklı durumda farklı şekilde uygulanabilir,zira bende uygulanıp anlatabilecek birçok farklı senaryo arasından bunu seçtim.Bu senaryo sadece bir örneği gösteriyor.İlgilendiğiniz için teşekkürler.























  