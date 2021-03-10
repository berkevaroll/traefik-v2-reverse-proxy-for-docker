# traefik-v2-reverse-proxy-for-docker

Traefik v2 reverse proxy documentation for docker apps


### Gereksinimler

- CentOS7 kurulu bir makine
- Öncelikle makinemizde docker yüklü olmalı.
- Makinede docker-compose yüklü olmalı.
- Kontrol panellerine ulaşabilmek için bir de domain gerekiyor. (örn: your_domain) İlerleyen örneklerde ve konfigürasyon dosyalarında your_domain kısımlarını kendi domain adımızla değiştirmeliyiz.
### Traefik 

Öncelikle Traefik Dashboard erişim için bir yönetici parolası oluşturacağız. Traefik yapılandırma dosyasında yönetici giriş bilgileri şifreli bir şekilde tutuluyor. Bu bilgileri iki şekilde şifreleyebilirsiniz.

##### htpasswd kullanarak bilgileri şifreleyebilirsiniz;

Önce `htpasswd` için gerekli paketleri yüklüyoruz.

```bash
sudo yum install httpd-tools
```
Daha sonra `htpasswd` ile Dashboard için kullanacağımız şifreyi oluşturuyoruz. `SECURE_PASSWROD` alanına kullanmak istediğini şifrenizi girin.

```bash
htpasswd -nb admin SECURE_PASSWORD
```

Programdan çıktı şu şekilde görünecektir:

```bash
admin:$apr1$9r5rUTis$nqG/J4R7365QtB7JBlc2N0
```

##### Htpasswd Generator kullanarak bilgileri şifreleyebilirsiniz;

https://hostingcanada.org/htpasswd-generator/ adresi üzerinden `Username` ve `Password` alanlarını doldurduktan sonra `Mode` alanını **Apache specific salted MD5 (insecure but common)* seçerek şifrenizi oluşturabilirsiniz.

#### Traefik Statik Yapılandırma Dosyası

Traefik Dashboard şifremizi hazırlıladıktan sonra `traefik.toml` dosyamızı hazırlıyoruz. Ben repo içerisine örnek bir dosya bıraktım.

Yapılandırma dosyamız aşağıdaki gibi görünecek;

```toml
[entryPoints]
  [entryPoints.web]
    address = ":80"
    [entryPoints.web.http.redirections.entryPoint]
      to = "websecure"
      scheme = "https"

  [entryPoints.websecure]
    address = ":443"

[api]
  dashboard = true

[certificatesResolvers.lets-encrypt.acme]
  email = "your_email@your_domain"
  storage = "acme.json"
  [certificatesResolvers.lets-encrypt.acme.tlsChallenge]

[providers.docker]
  watch = true
  network = "web"

[providers.file]
  filename = "traefik_dynamic.toml"
```
- Traefik'i bütün istekleri `https` üzerine yönlendireck şekilde yapılandırıyoruz. 
- Geçerli TLS sertifikaları oluşturmak için Let's Encrypt kullanıyoruz. Traefik v2 hiçbir ek ayar yapmadan Let's Encrypt desteği sağlıyor ve acme tipi sertifika çözücü oluşturarak ile kolaylıkla konfigüre edilebiliyor.
- your_email@your_domain kısmını geçerli bir email adresiyle değiştiriyoruz.
#### Traefik Dinam Yapılandırma Dosyası

Şimdi de dinamik ayarların yer alacağı traefik_dynamic.toml dosyamızı oluştuyruyoruyz. Bunun da bir örneği repoda yer almakta.

Dinamik yapılandırma dosyamız şu şekilde görünecek:

```toml
[http.middlewares.simpleAuth.basicAuth]
  users = [
    "YOUR_SECURE_PASSWORD"
  ]

[http.routers.api]
  rule = "Host(`monitor.your_domain`)"
  entrypoints = ["websecure"]
  middlewares = ["simpleAuth"]
  service = "api@internal"
  [http.routers.api.tls]
    certResolver = "lets-encrypt"
```
- YOUR_SECURE_PASSWORD kısmına 2. adımda oluşturduğumuz 'admin:$apr1$9r5rUTis$nqG/J4R7365QtB7JBlc2N0' şeklindeki çıktıyı yazacağız. Burada middleware ların kullanacağı authentication bilgileri tanımlanıyor.
- your_domain kısmına ise kendi domain adımızı yazıyoruz. Böylece 'monitor.your_domain' adresinden dashboarda erişim sağlayabileceğiz.

#### Traefik Konteynırımızı Çalıştırıyoruz

Öncelikle Traefik i proxy olarak kullanacağımız containerlarla paylaşılacak 'web' adında bir docker network oluşturuyoruz. Bu adımdan sonra Traefik dashboardına erişebileceğiz.

```bash
docker network create web
```

Daha sonra Let's Encrypt bilgilerimizi tutmak için boş bir 'acme.json' dosyası oluşturuyoruz. Bu dosyayı, Traefik in kullanması için containerda paylaşacağız.

```bash
touch acme.json
```

Sonrasında bu dosyayı sadece owner ın yazıp okuyabilmesi için yetkilerini düzenliyoruz.

```bash
chmod 600 acme.json
```

Son olarak bu komut ile Traefik konteynırımızı oluşturuyoruz.

```bash
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD/traefik.toml:/traefik.toml \
  -v $PWD/traefik_dynamic.toml:/traefik_dynamic.toml \
  -v $PWD/acme.json:/acme.json \
  -p 80:80 \
  -p 443:443 \
  --network web \
  --name traefik \
  traefik:v2.2
```

Artık 'monitor.your_domain/dashboard/' adresinden Traefik dashboardımıza erişebiliriz. Arayüze girişte bizden kullanıcı adı ve şifre istenecek. Burada 2. adımda oluşturduğumuz şifre ve admin kullanıcı adı ile giriş yapıyoruz. Dashboarda girdiğinizde şu şekilde bir arayüzle karşılacaksınız: 

[![](https://github.com/berkevaroll/traefik-v2-reverse-proxy-for-docker/blob/main/traefik_2_http_routers.1.png)](https://github.com/berkevaroll/traefik-v2-reverse-proxy-for-docker/blob/main/traefik_2_http_routers.1.png)

#### Konteynırları Traefike Kaydetme

Traefik altında çalıştırmak istediğiniz uygulamalar için aşağıdaki gibi bir docker-compose.yml dosyası oluşturuyoruz:

```yml
version: "3"

networks:
  web:
    external: true

services:
  service-api:
    image: your_image
    container_name: service_container_name
    restart: always
    labels:
      - traefik.http.routers.your_router.rule=Host(`service.your_domain`)
      - traefik.http.routers.your_router.tls=true
      - traefik.http.routers.your_router.tls.certresolver=lets-encrypt
      - traefik.port=80
    networks:
      - web
```

Yukarıdaki gibi bir yapılandırma dosyasında:

- your_image yerini kullanacağımız servisin image yolunu/ adını(registry.your_domain.com/image)
- service_container_name yerini servisimizin içinde çalıştığı konteynır ismi
- your_domain kısmına kendi domain adımızı
- your_router kısmını ise kendi vereceğimiz router ismini
	
yazarak uygulamamızı traefik altında çalıştırmaya hazır hale getirebiliriz. Burada router ismine dikkat etmek gerekiyor. Çünkü bir router ın sadece birer tane 80 ve 443 port u var. Yani bir router aynı anda 2 porta birden yönlendirme yapabilir.
Her uygulamaya ayrı router açmak hem yönetim hem de çalışabilirlik olarak fayda sağlayacaktır. Yine dashboardda aşağıdaki ekranda routerlarınızı görüntüleyebilirsiniz:

[![](https://github.com/berkevaroll/traefik-v2-reverse-proxy-for-docker/blob/main/traefik_2_empty_dashboard.1.png)](https://github.com/berkevaroll/traefik-v2-reverse-proxy-for-docker/blob/main/traefik_2_empty_dashboard.1.png)

- https://www.digitalocean.com/community/tutorials/how-to-use-traefik-v2-as-a-reverse-proxy-for-docker-containers-on-ubuntu-20-04

- https://github.com/barisates/traefik-reverse-proxy-for-docker
