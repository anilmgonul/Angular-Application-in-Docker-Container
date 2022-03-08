# Build and Run Angular Application in a Docker Container

Bu kisa makalede Angular aplikasyonunun Docker konteyneri icinde kullanimini gorecegiz. Is yukunu hafifletmek adina, multi-stage Docker 'a goz atacagiz. Sozu daha fazla uzatmadan, Angular uygulamamizi Docker konteyneri icine koyalim. Bu makalenin amacina uygun olmasi adina, daha onceden olusturulmus [bu uygulamayi](https://github.com/wkrzywiec/aston-villa-app) kullanacagiz.

![alt](models/avfc.png)


Oncelikli olarak, uygulamayi klonlamamiz gerekiyor. Terminali acalim ve asagidaki komutu yazalim:

```
$ git clone https://github.com/wkrzywiec/aston-villa-app.git

```

Daha sonra, local dosyamiza girelim. Node.js ve Angular CLI'in local bilgisayarda yukliu olmasi da gerekmekte. [Buradan](https://angular.io/guide/setup-local) set up yapabiliriz.

Eger onkosullar saglandiysa, Angular uygulamamizi derleyebiliriz.

```
$ ng build --prod
```

Bu komut ile **dist/aston-villa-app** adinda yeni bir klasor olusturacagiz. Boylece derlenmis butun dosyalar icinde barinacak.

Daha sonra ise, **Dockerfile** olusturup, proje root klasorune yerlestirelim. Dockerfile dosyasinin icinde olmasi gereken komutlar:

```
FROM nginx:1.17.1-alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY /dist/aston-villa-app /usr/share/nginx/html
```

Yukaridaki Dockerfile dosyasi ile bize anlatilmak istenen:

- Ilk olarak **nginx** Docker *1.17.1-alpine* etiketli imajini DockerHub'dan git al
- Sonra, default nginx konfigurasyonu kopyala yapistir
- Son olarak, derlenmis aplikasyonu kopyala yapistir

Dafault nginx konfigurasyon dosyasi asagidaki gibi gorunmekte:

```
events{}
http {
    include /etc/nginx/mime.types;
    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;
        location / {
            try_files $uri $uri/ /index.html;
        }
    }
}
```

Genel olarak, burada aplikasyonun host edilecegi server'i, port numarasini ve davranis seklini belirtiliyoruz.

Simdi, terminale geri donelim ve asagidaki komutu yazalim:

```
$ docker build -t av-app-image .
```

Lokal olarak mevcut olan Docker imajlarini kontrol edecek olursak:

```
$ docker image ls

REPOSITORY    TAG            IMAGE ID      
av-app-image  latest         a160a7494a19      
nginx         1.17.1-alpine  ea1193fd3dde
```

Yeni olusturdugumuz imajimizi calistirmak icin :

```
$ docker run --name av-app-container -d -p 8080:80 av-app-image
```

Yukaridaki komutla beraber konteynerimiza isim veriyoruz (```--name av-app-container```), daha sonra ise arka planda calisacaginda emin oluyoruz (```-d```), sonra, konteyner port'unu yonlendiriyoruz (```-p 8080:80```) ve son olarak, olusturdugumuz base Docker imajini seciyoruz (```av-app-image```).

Konteynerin calistigini kontrol etmek icin:

```
$ docker container ls

CONTAINER ID  IMAGE         STATUS         NAMES
2523d9f77cf6  av-app-image  Up 26 minutes  av-app-container
```
Veyahut, web browser uzerinden erisim saglanabilir. [http://localhost:8080/](http://localhost:8080/login?from=%2F)

## What is Multi-Stage Docker Build?

Multi-stage Docker build Docker 17.05 ile beraber cikti ve Dockerfile'larin okunurlulugunu kaybetmeden daha kucuk konteyner uretmeyi amaclar. Bu yaklasimla beraber Docker imajlarini daha kucuk parcalara bolebiliriz ve bir konteynerin sonucunu bir digerinde kullanabiliriz.

Bu islem iki asamada gerceklesecek:

1. Kaynak kodunu urun icerisinde derlemek
2. Derlenmis uygulamayi Docker imaji icerisinde calistirmak

Kontey korunmasi icin, birinci evrenin ciktisi ikinci evreye aktarilir. Simdiye kadar ikinci kisimda islem yaptik, simdi ise birinci kisima odaklanacagiz. Icinde **Node.js** barindiran baska bir Docker imaj kullanarak, Dockerfile olusturalim.

```
FROM node:12.7-alpine AS build
WORKDIR /usr/src/app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build
```

Bu senaryoyu goz onune aldigimizda:

- **FROM** registry'den bize *node* Docker imajini aliyor ve derleme asamasini adlandirarak **build** ediyor.
- **WORKDIR** default depolama bolumu icin dosyayi olusturuyor
- **COPY** ise *package.json* ve *package-lock.json* dosyalarini local root directory'den koplyaliyor
- **RUN** gerekli kutuphaneleri yukluyor
- **COPY** ise geriye kalan tum dosyalari kopyaliyor
-**RUN** uygulamamizi derliyor.

Bunun ardinda, dosyamiz su sekilde gorulecektir:

```
dist
node_modules
```

Tum Docker asamalarini birlestirebiliriz ve soyle bir sonuc elde ederiz:

```
### STAGE 1: Build ###
FROM node:12.7-alpine AS build
WORKDIR /usr/src/app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

### STAGE 2: Run ###
FROM nginx:1.17.1-alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=build /usr/src/app/dist/aston-villa-app /usr/share/nginx/html
```

Terminale geri donup, Docker imajimizi olusturalim:

```
$ docker build -t av-app-multistage-image .
```

Daha sonra, farkli bir port'ta uygulamamizi calistiralim:

```
$ docker run --name av-app-multistage-container -d -p 8888:80 av-app-multistage-image
```

Sonuc olarak, projeyi buradan gorebiliriz. [http://localhost:8888/](http://localhost:8888/)
