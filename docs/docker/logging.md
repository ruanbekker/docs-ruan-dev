# Docker Logging

Theres a couple of ways logging with docker

## JSON-File

To read more about the json-file logging driver, view their [documentation](https://docs.docker.com/config/containers/logging/json-file/)

A example `docker-compose.yml`:

```yaml
version: "3.7"

services:
  nginx-app:
    image: nginx:latest
    ports:
      - 80:80
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## Fluentd

Logging via fluentd using the [fluentd logging driver](https://docs.docker.com/config/containers/logging/fluentd/), you need to setup the fluentd daemon running on the host (as a container in this case) and then your application containers will point the fluentd log driver to the daemon.

Setup the fluentd configuration, in this case our config will just return the log to standard out. In our `test.conf`:

```
 <source>
   @type forward
 </source>

 <match *>
   @type stdout
 </match>
```

Our `docker-compose.fluentd.yml`:

```yaml
version: "3.7"

services:
  fluentd:
    image: fluent/fluentd:latest
    container_name: fluentd
    restart: always
    environment:
      - FLUENTD_CONF=test.conf
    volumes:
      - ./test.conf:/fluentd/etc/test.conf
    ports:
      - "24224:24224"
      - "24224:24224/udp"
```

Then run the fluentd daemon:

```sh
docker-compose -f docker-compose.fluentd.yml up -d
```

Now our application container can make use of the fluentd logging driver in `docker-compose.yml`:

```yaml
version: "3.7"

services:
  nginx-app:
    image: nginx:latest
    container_name: nginx-app
    ports:
      - 80:80
    logging:
      driver: fluentd
      options:
        fluentd-async-connect: "true"
        fluentd-address: localhost:24224
        tag: "example-{{.Name}}"
```

Boot the application:

```sh
docker-compose -f docker-compose.yml up -d
```

And because our application is logging via fluentd, we can tail the logs via the fluentd container:

```sh
docker logs -f fluentd
```
```
2021-03-31 09:58:46.000000000 +0000 example-nginx-app: {"log":"192.168.240.1 - - [31/Mar/2021:09:58:46 +0000] \"GET / HTTP/1.1\" 200 612 \"-\" \"curl/7.64.1\" \"-\"","container_id":"b4938f0e67bcded3e829b59631d65281d0b9ee96a433340dceccc34de82f2692","container_name":"/nginx-app","source":"stdout"}
```

Further configuration can be found on [fluentd](https://docs.fluentd.org/)

## Fluentbit with Loki

Grafana has a docker image which includes the loki plugin for fluentbit, which exposes a port for logging clients to ship its logs to fluentbit and then fluentbit ships the logs to loki.

My [loki stack](https://docs.ruan.dev/docker/loki) is running on the contained network, which makes it possible for the fluent-bit container to reach loki on then dns name `loki`:

```yaml
version: "3.7"

services:
  fluent-bit:
    image: grafana/fluent-bit-plugin-loki:latest
    container_name: fluent-bit
    restart: always
    environment:
      - LOKI_URL=http://loki:3100/loki/api/v1/push
    volumes:
      - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    networks:
      - contained

networks:
  public:
    name: contained
```

Our fluentbit configuration, `fluent-bit.conf`:

```conf
[INPUT]
    Name        forward
    Listen      0.0.0.0
    Port        24224
[Output]
    Name grafana-loki
    Match *
    Url ${LOKI_URL}
    RemoveKeys source,container_id
    Labels {job="fluent-bit"}
    LabelKeys container_name
    BatchWait 1s
    BatchSize 1001024
    LineFormat json
    LogLevel info
```

Boot the fluentbit container:

```bash
docker-compose up -d
```

Now our application container can make use of the fluentd logging driver in `docker-compose-app.yml`:

```yaml
version: "3.7"

services:
  nginx-app:
    image: nginx:latest
    container_name: nginx-app
    networks:
      - contained
    ports:
      - 8080:80
    labels:
      - "traefik.http.routers.nginx.rule=Host(`nginx-app.localdns.xyz`)"
      - "traefik.http.services.nginx.loadbalancer.server.port=80"
    logging:
      driver: fluentd
      options:
        fluentd-async-connect: "true"
        fluentd-address: localhost:24224

networks:
  contained:
    name: contained
```

Boot the application:

```sh
docker-compose -f docker-compose-app.yml up -d
```

Make a request to nginx:

```bash
curl http://nginx-app.localdns.xyz
```

Head over to grafana, select explore, select the Loki datasource and use the labels:

```json
{job="fluent-bit"}
```

And you should see your logs, if you have not setup your Loki datasource you can visit:
- https://docs.ruan.dev/docker/loki
