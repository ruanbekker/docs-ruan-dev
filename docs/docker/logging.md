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
