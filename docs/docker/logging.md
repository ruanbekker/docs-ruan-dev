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

## Promtail with Loki

[Promtail](https://grafana.com/docs/loki/latest/clients/promtail/) is an agent which ships the contents of local logs to Grafana Loki.

My [loki stack](https://docs.ruan.dev/docker/loki) is running on the contained network, which makes it possible for the promtail container to reach loki on then dns name `loki`:

```yaml
version: "3.7"

services:
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: always
    command: -config.file=/etc/promtail/docker-config.yml
    volumes:
      - /var/lib/docker/:/var/lib/docker:ro
      - ./promtail-config.yml:/etc/promtail/docker-config.yml
    networks:
      - contained
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

networks:
  public:
    name: contained
```

Our promtail configuration, `promtail-config.yml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: containers
  static_configs:
  - targets:
      - localhost
    labels:
      job: container-logs
      env: dev
      __path__: /var/lib/docker/containers/*/*log
```

Boot the promtail container:

```bash
docker-compose up -d
```

Now when you head over to grafana, select explore, select the Loki datasource and use the labels:

```json
{job="container-logs"}
```

And you should see your logs, if you have not setup your Loki datasource you can visit:
- https://docs.ruan.dev/docker/loki

### Container Names Workaround

You will however notice that your containers are identified by the container id's and if you have lots of containers that can be a bit painful. A way to do a workaround is to generate a template then make use of the [file_sd_configs](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) to relabel our data.

If we run a `docker ps` with the format flag we can manipulate the output we want, and we can get this output:

```bash
docker ps --format '- targets: ["{{.ID}}"]\n  labels:\n    job: "container-logs"\n    container_name: "{{.Names}}"'
- targets: ["bd940522d110"]
  labels:
    job: "container-logs"
    container_name: "sidecar"
- targets: ["d94c96ccb9cd"]
  labels:
    job: "container-logs"
    container_name: "nginx"
```

We will require some scripting, so lets assume the filepath `/opt/scripts/container-promtail-generator.sh` has the following content:

```bash
#!/usr/bin/env bash
# credit to:
# https://github.com/grafana/loki/issues/333#issuecomment-464164652 
docker ps --format '- targets: ["{{.ID}}"]\n  labels:\n    job: "containerlogs"\n    container_name: "{{.Names}}"' > /home/cyrax/promtail/promtail-targets.yml
```

And we are assuming that our promtail `docker-compose.yml` is located in the directory `/home/cyrax/promtail` as this script will write the content to `/home/cyrax/promtail/promtail-targets.yml`.

Now make this script executable:

```bash
chmod +x /opt/scripts/container-promtail-generator.sh
```

Then we add this to a cronjob using `crontab -e` so that it updates the targets file every 15 minutes:

```
*/15 * * * * /opt/scripts/container-promtail-generator.sh
```

Then in our `docker-compose.yml` we update our volumes section to include the targets file:

```yaml
version: "3.7"

services:
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: always
    command: -config.file=/etc/promtail/docker-config.yml
    volumes:
      - /var/lib/docker/:/var/lib/docker:ro
      - ./promtail-config.yml:/etc/promtail/docker-config.yml
      - ./promtail-targets.yml:/etc/promtail/promtail-targets.yml
    networks:
      - contained
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

networks:
  public:
    name: contained
```

And we also need to update our `promtail-config.yml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: containerlogs
  file_sd_configs:
  - files:
    - /etc/promtail/promtail-targets.yml
  relabel_configs:
  - source_labels: [job]
    target_label: job
  - source_labels: [__address__]
    target_label: container_id
  - source_labels: [container_id]
    target_label: __path__
    replacement: /var/lib/docker/containers/$1*/*.log
```

Then redeploy the promtail container:

```bash
docker-compose up -d
```

Now your logs should include the container id in `container_id`, and the containers name will mapped to the label `container_name`. 

Which will look like this for a specific container:

```
{
  job="containerlogs",
  container_name="debugger-app",
  container_id="b508dc50a6c2",
  filename="/var/lib/docker/containers/b508dc50a6xx9ace942b97da23/b508dc50a6xx9ace942b97da23-json.log"
}
```

You will notice if theres new containers you should restart the promtail container. Which is not ideal, but its a workaround.