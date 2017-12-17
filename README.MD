# scs-processor-geocoding-reverse
## Based on [Spring Cloud Stream](https://cloud.spring.io/spring-cloud-stream/) 
### Message-driven data microservices to transform input flat coordinates signal to local address in json format(using MongoDB for Reverse-Geocoding Query with Tokyo GeoJson data).

# 1. Local Deployment as Spring Cloud Stream Application

## 1-1. Prerequisites for Local Deployment

* [Homebrew](http://brew.sh/) if you use Mac OS X and prefer homebrew otherwise you need to install/start all the middleware components manually.
* [Homebrew Services](https://github.com/Homebrew/homebrew-services)
* [Maven](http://brewformulas.org/Maven)
* [RabbitMQ](http://brewformulas.org/Rabbitmq)
* [MongoDB](http://brewformulas.org/Mongodb)
* [Kafka](http://brewformulas.org/Kafka) is an alternative for messaging.
* [http-source-rabbit](http://repo.spring.io/release/org/springframework/cloud/stream/app/http-source-rabbit/)

### 1-1-1. Download and Build

```
git clone https://github.com/komushi/scs-processor-geocoding-reverse.git
cd scs-processor-geocoding-reverse
mvn clean package
```

### 1-1-2. MongoDB for Reverse-Geocoding

Start MongoDB
```
brew services start mongodb
```
The default dbpath is explained [here](https://gist.github.com/subfuzion/9630872).

Import GeoJson for Reverse-Geocoding.
```
mongoimport --db geojson --collection blocks --type json --file "data/chiyoda_sample.json"
```

Create a user to enable geojson access
```
use geojson

db.createUser(
    {
      user: "root",
      pwd: "keepitsimple",
      roles: ["readWrite"]
    }
) 
```

Test in mongo shell
```
db.blocks.createIndex({ geometry: "2dsphere" })

db.blocks.find(
   {
     geometry: {
       $geoIntersects: {
          $geometry: {
             type: "Point" ,
             coordinates: [ 139.743317, 35.682955 ]
          }
       }
     }
   }
)
```

Result:
```
{
  "_id" : ObjectId("579c2aaa35a06dc3a623daf7"),
  "properties": {
    "district": "千代田区",
    "block": "隼町",
    "code": "1010070"
  },
  "geometry": {
    "type": "Polygon",
    "coordinates": [
      [
        [
          139.74167300322014,
          35.683406014170174
        ],
        ...
        ...
        [
          139.74167300322014,
          35.683406014170174
        ]
      ]
    ]
  }
}
```



### 1-1-3. Download http-source-rabbit
```
wget http://repo.spring.io/release/org/springframework/cloud/stream/app/http-source-rabbit/1.0.2.RELEASE/http-source-rabbit-1.0.2.RELEASE.jar
```

## 1-2. Test with Local Deployment

### 1-2-1. Start MongoDB if not started already
```
brew services start mongodb
```

### 1-2-2. Start RabbitMQ
```
brew services start rabbitmq
```

### 1-2-3. Start scs-source-http
```
java -jar scs-processor-http-1.0.0-RELEASE.jar
```

### 1-2-4. Start scs-processor-geocoding-reverse
```
java -jar target/scs-processor-geocoding-reverse-1.0.0-RELEASE.jar
```

### 1-2-5. Post a message

```
curl -H "Content-Type: text/plain" -X POST -d'468244D1361B8A3EB8D206CC394BC9E9,BB899DFEA9CC964B50C540A1D685CCFB,2013-01-01 00:00:00,2013-01-01 00:04:00,240,1.71,139.752268,35.677043,139.771699,35.697283,CSH,6.50,0.50,0.50,0.00,0.00,7.50' 'http://localhost:9000'
```

Log Output
```
payload:468244D1361B8A3EB8D206CC394BC9E9,BB899DFEA9CC964B50C540A1D685CCFB,2013-01-01 00:00:00,2013-01-01 00:04:00,240,1.71,139.752268,35.677043,139.771699,35.697283,CSH,6.50,0.50,0.50,0.00,0.00,7.50

tuple:
{
  "uuid": "fada04d3-8afd-4e07-89e6-3761f802483d",
  "route": "10103400_10100500",
  "timestamp": 1469850262440,
  "pickupAddress": "千代田区 外神田",
  "dropoffAddress": "千代田区 霞が関",
  "pickupLatitude": 35.697283,
  "pickupLongitude": 139.771699,
  "dropoffLatitude": 35.677043,
  "dropoffLongitude": 139.752268,
  "pickupDatetime": "2013-01-01 00:00:00",
  "dropoffDatetime": "2013-01-01 00:04:00"
}
```

### 1-2-6 Test with Google Map
[Longitude:139.752268 Latitude:35.677043 @霞が関](https://www.google.co.jp/maps/place/35.677043,139.752268)

![Top Ten List & Matrix](https://rawgit.com/komushi/scs-processor-geocoding-reverse/master/image/kasumigaseki.png)

[Longitude:139.771699 Latitude:35.697283 @外神田](https://www.google.co.jp/maps/place/35.697283,139.771699)

![Top Ten List & Matrix](https://rawgit.com/komushi/scs-processor-geocoding-reverse/master/image/sotokanda.png)

# 2. Another deployment option - spring boot in docker

## 2-1. assembly.xml under src/main/docker
```
<assembly
  xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2 http://maven.apache.org/xsd/assembly-1.1.2.xsd">
  <id>scs-processor-geocoding-reverse</id>
  <dependencySets>
    <dependencySet>
      <includes>
        <include>io.pivotal.spring.cloud.stream.processor.geocoding:scs-processor-geocoding-reverse</include>
      </includes>
      <outputDirectory>.</outputDirectory>
      <outputFileNameMapping>scs-processor-geocoding-reverse.jar</outputFileNameMapping>
    </dependencySet>
  </dependencySets>
</assembly>
```

## 2-2. Modify docker-maven-plugin inside pom.xml
### 2-2-1. Entry Point Shell Mode
Under plugin/configuration/images/image/build/ modify entryPoint like below.
```
...
<entryPoint>
    <shell>java -jar /maven/${project.artifactId}.jar $SCS_OPTS</shell>
</entryPoint>
...
```

### 2-2-2. Entry Point Exec Mode
Under plugin/configuration/images/image/build/, modify entryPoint like below.
```
...
<entryPoint>
    <exec>
        <arg>java</arg>
        <arg>-jar</arg>
        <arg>/maven/${project.artifactId}.jar</arg>
    </exec>
</entryPoint>
...
```

## 2-3. Build docker image by maven
```
mvn clean package docker:build
```

if necessay push to docker hub

```
mvn docker:push -Ddocker.username=[provide your username] -Ddocker.password=[provide password]
```

## 2-4. Run the image

Since we use docker container here and the Spring Cloud Stream App deault port is 8080 so the --server.port(the container port) parameter does not need to be specified for docker deployment.

```
docker run -p <host_port>:<container_port> komushi/scs-processor-geocoding-reverse <app_args>
```

## 2-5. Docker Run Test

### 2-5-1. Test with RabbitMQ

```
docker run -p 9000:9000 komushi/scs-source-http:1.0.0.RELEASE --spring.rabbitmq.host=docker.for.mac.localhost --spring.rabbitmq.port=5672 --spring.rabbitmq.username=guest --spring.rabbitmq.password=guest --spring.cloud.stream.defaultBinder=rabbit
```

```
docker run komushi/scs-processor-geocoding-reverse:1.0.0.RELEASE --spring.rabbitmq.host=docker.for.mac.localhost --spring.rabbitmq.port=5672 --spring.rabbitmq.username=guest --spring.rabbitmq.password=guest --spring.cloud.stream.defaultBinder=rabbit
```


### 2-5-2. Post a message

```
curl -H "Content-Type: text/plain" -X POST -d'468244D1361B8A3EB8D206CC394BC9E9,BB899DFEA9CC964B50C540A1D685CCFB,2013-01-01 00:00:00,2013-01-01 00:04:00,240,1.71,139.752268,35.677043,139.771699,35.697283,CSH,6.50,0.50,0.50,0.00,0.00,7.50' 'http://localhost:9000'
```

Log Output
```
payload:468244D1361B8A3EB8D206CC394BC9E9,BB899DFEA9CC964B50C540A1D685CCFB,2013-01-01 00:00:00,2013-01-01 00:04:00,240,1.71,139.752268,35.677043,139.771699,35.697283,CSH,6.50,0.50,0.50,0.00,0.00,7.50

tuple:
{
  "uuid": "fada04d3-8afd-4e07-89e6-3761f802483d",
  "route": "10103400_10100500",
  "timestamp": 1469850262440,
  "pickupAddress": "千代田区 外神田",
  "dropoffAddress": "千代田区 霞が関",
  "pickupLatitude": 35.697283,
  "pickupLongitude": 139.771699,
  "dropoffLatitude": 35.677043,
  "dropoffLongitude": 139.752268,
  "pickupDatetime": "2013-01-01 00:00:00",
  "dropoffDatetime": "2013-01-01 00:04:00"
}
```