# GCELL-infra

## 인프라 설정파일

### API Server

**DockerFile**

```Dockerfile
FROM openjdk:17
COPY build/libs/*.jar app/app.jar
WORKDIR /app
ENTRYPOINT ["java", "-Xmx1024m","-jar","app.jar"]
```

**docker-compose.yml**

```yml
version: "3.9"

networks:
  gcell-api-servertmp_default:
    external: true
  api-db_default:
    external: true
  minio_default:
    external: true
  rabbitmq_default:
    external: true

services:
  api-was:
    image: mentoring-gitlab.gabia.com:5050/mentee/mentee_2023.01/team/weat/gcell-api-server:latest
    working_dir: /app
    command: ["./gradlew", "bootrun", "--spring.profiles.active=prod"]
    restart: on-failure
    expose:
      - "8080"
      - "8081"
    networks:
      - gcell-api-servertmp_default
      - api-db_default
      - minio_default
      - rabbitmq_default
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "1"
          memory: 1500M
    logging:
      driver: json-file
      options:
        tag: "{{.ImageName}}|{{.Name}}|{{.ImageFullID}}|{{.FullID}}"
    environment:
      - TZ=Asia/Seoul
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/api/actuator/health"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 60s
```
<br>

---

### Excel Server

**DockerFile**

```Dockerfile
FROM openjdk:17
COPY build/libs/*.jar app/app.jar
WORKDIR /app
ENTRYPOINT ["java", "-Xmx1024m","-jar","app.jar"]
```

**docker-compose.yml**

```yml
version: "3.9"

networks:
  excel-db_default:
    external: true
  minio_default:
    external: true
  rabbitmq_default:
    external: true

services:
  excel-was:
    image: mentoring-gitlab.gabia.com:5050/mentee/mentee_2023.01/team/weat/gcell-excel-server:latest
    working_dir: /app
    command: ["./gradlew", "bootrun"]
    restart: on-failure
    expose:
      - "8080"
      - "8081"
    networks:
      - excel-db_default
      - minio_default
      - rabbitmq_default
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "7"
          memory: 1500M
    logging:
      driver: json-file
      options:
        tag: "{{.ImageName}}|{{.Name}}|{{.ImageFullID}}|{{.FullID}}"
    environment:
      - TZ=Asia/Seoul
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/actuator/health"]
      interval: 5s
      timeout: 10s
      retries: 3
      start_period: 60s
```
<br>

---

### Webserber, Frontend

**Dockerfile**
```Dockerfile
FROM node:16-alpine as build-stage
WORKDIR /app
ADD . .
RUN npm install
RUN npm run build

# production stage
FROM nginx:stable-alpine as production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY ./nginx/mime.types /etc/nginx/mime.types
RUN chmod -R 755 /usr/share/nginx/html
CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf**
```conf
user  root;
pid        /var/run/nginx.pid;
worker_rlimit_nofile 8192;

events{
    worker_connections  1024;
}

http {

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    sendfile        on;
    keepalive_timeout  65;
    client_max_body_size 2000M;

    server {
        listen       80 default_server;
        listen      [::]:80 default_server;
        server_name  _;

        root   /usr/share/nginx/html;
        location / {
            try_files $uri $uri/ /index.html;
        }


        location ^~ /api {
           
            proxy_http_version 1.1;
            proxy_set_header   Connection "";
            proxy_pass http://api-was:8080/api;
            proxy_connect_timeout 600;      
            proxy_send_timeout 600;      
            proxy_read_timeout 600;      
            send_timeout 600;   
        }

        }

}
```

**docker-compose.yml**
```yml
version: '3.9'

networks:
  gcell-api-servertmp_default:
    external: true

services:
  fe-ws:
    image: mentoring-gitlab.gabia.com:5050/mentee/mentee_2023.01/team/weat/gcell-frontend:latest
    ports:
      - "80:80"
    restart: always
    networks:
      - default
      - gcell-api-servertmp_default
    logging:
      driver: json-file
      options:
        tag: "{{.ImageName}}|{{.Name}}|{{.ImageFullID}}|{{.FullID}}"
    environment:
      - TZ=Asia/Seoul
```

<br>

---

### RabbitMQ

**docker-compose.yml**
```yml
version: "3.9"

networks:
  rabbitmq_default:
    external: true

services:
  rabbitmq:
    image: 'rabbitmq:3.11.8-management'
    ports:
      # The standard AMQP protocol port
      - '5672:5672'
      # HTTP management UI
      - '15672:15672'
    restart: always
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./my_definition.json:/etc/rabbitmq/my_definition.json
    networks:
      - rabbitmq_default
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 1024M
```
<br>

---

### API DB

**docker-compose**
```yml
version: "3.9"

networks:
  api-db_default:
    external: true

services:
  db:
    image: mysql:8.0.31
    restart: always
    volumes:
      - ./mysqldata:/var/lib/mysql
    networks:
      - api-db_default
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 1024M
```
<br>

---

### EXCEL DB



**docker-compose**
```yml
version: "3.9"

networks:
  excel-db_default:
    external: true

services:
  db:
    container_name: excel-db
    image: mysql:8.0.31
    restart: always
    volumes:
      - ./mysqldata:/var/lib/mysql
    networks:
      - excel-db_default
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 1024M
```
<br>

---

### MinIO

**docker-compose.yml**
```yml
version: "3.9"

networks:
  minio_default:
    external: true

services:
  minio:
    image: minio/minio:RELEASE.2023-01-31T02-24-19Z
    command: server /data --console-address ":9001"
    restart: always
    shm_size: '1gb'
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./data:/data
    networks:
      - minio_default
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 512M
```
<br>

---

### Monitoring

**docker-compose.yml**
```yml
version: '3.9'
networks:
  monitor:
    driver: bridge
  gcell-api-servertmp_default:
    external: true
  excel-db_default:
    external: true

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./pm/:/etc/prometheus/
      - ./data:/prometheus
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitor
      - gcell-api-servertmp_default
      - excel-db_default
    restart: always
  grafana:
     container_name: grafana
     image: grafana/grafana:latest
     user: root
     volumes:
       - ./grafana/data/grafana.ini/grafana.ini:/etc/grafana/grafana.ini
       - ./grafana/data:/var/lib/grafana
       - ./grafana/provisioning:/etc/grafana/provisioning
     ports:
       - "3000:3000"
     depends_on:
       - prometheus
     networks:
       - monitor
     restart: always
  loki:
    image: grafana/loki
    expose:
      - 3100
    volumes:
      - ./loki/config/loki-config.yml:/etc/loki/local-config.yml
    networks:
      - monitor

  promtail:
    image: grafana/promtail
    volumes:
      - ./loki/config/promtail-config.yml:/etc/promtail/config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers
    networks:
      - monitor
    restart: always

```
