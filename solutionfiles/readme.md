# petclinic-task:

Bu proje, Java Spring tabanlı bir uygulama olan "Spring PetClinic" örneğinin Docker ve Kubernetes ortamlarında çalıştırılma süreçlerini içermektedir. Ayrıca GitHub Actions ile süreç otomasyonu yapılandırmasını içermektedir.

# 1. proje dosyalarının indirilmesi (clone) :

Projenin kaynak kodları ve Maven bağımlılıkları indirilir. mvnw package komutuyla JAR dosyaları hazırlanır ve kullanılabilir hale getirilir.


https://github.com/spring-projects/spring-petclinic

ilk adım olarak proje dosyaları ve maven yüklenmesi, mvnw package ile jar dosyalarının kullanılır hale getirilmesi

```
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
./mvnw install
./mvnw package
java -jar target/*.jar
```
# 2. Docker File :

Docker konteyneri oluşturmak için kullanılan Dockerfile içeriği. OpenJDK 17 tabanlı bir konteyner oluşturulur, projenin JAR dosyası kopyalanır ve container içinde çalıştırılacak komutlar girilir.

```
FROM openjdk:17-jdk-alpine
WORKDIR /app
COPY target/spring-petclinic-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]

```

 # 3. Adım Docker Compose :

Docker Compose dosyası, PetClinic uygulamasını ve MySQL veritabanını içeren hizmetleri yapılandırır. Uygulama ve veritabanı konteynerleri oluşturulur ve bağlantıları sağlanır.
Mysql url bilgisi src/application-mysql.properties'ten alındı.

```
version: "3"

services:
  spring-petclinic:
    image: kayretli/fatihclinic:latest
    ports:
      - "8082:8080"
    environment:
      - MYSQL_USER=petclinic
      - MYSQL_PASSWORD=petclinic
      - MYSQL_URL=jdbc:mysql://mysql/petclinic

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
      - MYSQL_USER=petclinic
      - MYSQL_PASSWORD=petclinic
      - MYSQL_DATABASE=petclinic
    volumes:
      - "./conf.d:/etc/mysql/conf.d:ro"



```

# 4. Adım Deployment,Service ve Mysql :

Kubernetes yaml dosyaları ile petclinic uygulamasının deployment ve servis ayarları yapılandırılır. Ayrıca MySQL veritabanı için de deployment ve servis ayarları yapılandırılır. 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-petclinic-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fatihclinic
  template:
    metadata:
      labels:
        app: fatihclinic
    spec:
      containers:
        - name: spring-petclinic-container
          image: kayretli/fatihclinic:latest 
          ports:
            - containerPort: 8080
          env:
            - name: MYSQL_URL
              value: "jdbc:mysql://mysql-service.default.svc.cluster.local/petclinic"
            - name : MYSQL_USER
              value: petclinic
            - name : MYSQL_PASSWORD
              value: petclinic

---

apiVersion: v1
kind: Service
metadata:
  name: spring-petclinic-service
spec:
  selector:
    app: fatihclinic
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: LoadBalancer

  ```

# 5. Adım Mysql Deployment :

Bu adımda MySQL veritabanını içeren deployment ve servis ayarları konfigüre edilir.
MySQL konteynerı başlatılır ve servis bağlantısı sağlanır.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: fatih
            - name: MYSQL_DATABASE
              value: petclinic
            - name: MYSQL_ALLOW_EMPTY_PASSWORD
              value: true
            - name : MYSQL_USER
              value: petclinic
            - name : MYSQL_PASSWORD
              value: petclinic
          
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  type: ClusterIP
```

# 6. Adım Github Action CI:

Bu aşamada Github Action ile CI workflow oluşturulur. Maven ile paketleme işlemi gerçekleştirilir, oluşturulan JAR dosyası GitHub'a yüklenir. Ardından Docker imajı oluşturulur, etiketlenir ve Docker Hub'a yüklenir.
```
name: Java CI with Maven

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch: #manuel trigger için.

jobs:
  build:

    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: [ '17' ]

    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK ${{matrix.java}}
        uses: actions/setup-java@v2
        with:
          java-version: ${{matrix.java}}
          distribution: 'adopt'
          cache: maven
      - name: Build with Maven Wrapper
        run: ./mvnw -B package
      - name: Upload Jar File
        uses: actions/upload-artifact@v1
        with:
          name: pet-clinic-jar
          path: target
      

    
  docker:
  
    runs-on: ubuntu-latest  
    needs:
      - build
    steps:
      - uses: actions/checkout@v3
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Download pet-clinic Jar
        uses: actions/download-artifact@v1
        with:
          name: pet-clinic-jar
          path: target  
      -
        name: Build, tag, and push image to DockerHub
        id: build-image
        env:
         DOCKER_REGISTRY: kayretli
         DOCKER_REPOSITORY: fatihclinic
        run: |  
          docker build -t $DOCKER_REGISTRY/$DOCKER_REPOSITORY:${GITHUB_SHA:0:6} .
          docker push $DOCKER_REGISTRY/$DOCKER_REPOSITORY:${GITHUB_SHA:0:6}
          echo "::set-output name=image::$DOCKER_REGISTRY/$DOCKER_REPOSITORY:${GITHUB_SHA:0:6}"

```          