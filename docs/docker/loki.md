# Loki on Docker

This setup will show you how to setup a Grafana Loki stack on docker using docker-compose.

## Compose

This `docker-compose.yml` will boot a traefik, grafana and loki container:

```yaml
version: "3.7"

services:
  traefik:
    image: traefik:v2.4.5
    container_name: traefik
    command: [ '--providers.docker', '--api.insecure' ]
    ports:
      - 80:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - contained
    labels:
      - "traefik.http.routers.traefik.rule=Host(`traefik.localdns.xyz`)"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

  grafana:
    image: grafana/grafana:7.4.2
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=password
    volumes:
      - ./data/grafana:/var/lib/grafana
    networks:
      - contained
    ports:
      - 3000:3000
    depends_on:
      - loki
    labels:
      - "traefik.http.routers.traefik.rule=Host(`grafana.localdns.xyz`)"
      - "traefik.http.services.traefik.loadbalancer.server.port=3000"
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

  loki:
    image: grafana/loki:2.2.0
    container_name: loki
    command: -config.file=/mnt/loki-config.yml
    user: root
    restart: unless-stopped
    volumes:
      - ./data/loki/data:/tmp/loki
      - ./loki-config.yml:/mnt/loki-config.yml
    networks:
      - contained
    ports:
      - 3100:3100
    labels:
      - "traefik.http.routers.traefik.rule=Host(`loki.localdns.xyz`)"
      - "traefik.http.services.traefik.loadbalancer.server.port=3100"
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

networks:
  contained:
    name: contained
```

Then we wave our loki config, [loki-config.yml](https://raw.githubusercontent.com/grafana/loki/master/cmd/loki/loki-local-config.yaml):

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

ingester:
  wal:
    enabled: true
    dir: /tmp/wal
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 1h       
  max_chunk_age: 1h
  chunk_target_size: 1048576
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/boltdb-shipper-active
    cache_location: /tmp/loki/boltdb-shipper-cache
    cache_ttl: 24h         
    shared_store: filesystem
  filesystem:
    directory: /tmp/loki/chunks

compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: filesystem

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s
```

## Boot the Stack

Boot the loki stack with:

```bash
docker-compose up -d
```

## Access Grafana

Access grafana on http://grafana.localdns.xyz with the username `admin` and `password` then select settings and datasources and add the loki datasource with the url `http://loki:3100` and select save.

Now you should be able to view your logs from the loki datasource in the explore view.
