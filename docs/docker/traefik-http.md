# Traefik in HTTP Only

[![](https://img.shields.io/badge/website-ruan.dev-red.svg)](https://ruan.dev) [![](https://img.shields.io/badge/twitter-@ruanbekker-00acee.svg)](https://twitter.com/ruanbekker) [![](https://img.shields.io/badge/github-cheatsheets-orange.svg)](https://github.com/ruanbekker) [![Say Thanks!](https://img.shields.io/badge/dm-saythanks.io-07B63F.svg)](https://saythanks.io/to/ruanbekker)  [![Ko-fi](https://img.shields.io/badge/-Buy%20Me%20a%20Coffee-ff5f5f?logo=ko-fi&logoColor=white)](https://ko-fi.com/ruanbekker)

Traefik is a Modern HTTP Reverse Proxy and Load Balancer that makes deploying microservices easily.

## About

This will run a traefik proxy in http only mode, and wire a application behind the proxy for demonstration.

## Configuration

To run a traefik proxy and a web container with docker-compose, the `docker-compose.yml` will look like this:

```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v2.7
    container_name: traefik
    command:
      - "--log.level=INFO"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=public"
      - "--entrypoints.web.address=:80"
    ports:
      - 80:80
      - 8080:8080
    networks:
      - public
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        
  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.127.0.0.1.nip.io`)"
      - "traefik.http.routers.whoami.entrypoints=web"
      - "traefik.http.services.whoami.loadbalancer.server.port=80"
    networks:
      - public
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

networks:
  public:
    name: public

```

As you can see we are exposing port `80` as we are only running this example in `http`, mounting the `docker.sock` so that traefik can read docker events and read information about other containers.

Then the labels is where the magic happens, we are defining the host rule `whoami.127.0.0.1.nip.io` to route to port `80` on the `whoami` container. 

## Deploy

To boot our service:

```sh
docker-compose -f docker-compose.yml up -d
```

And testing our application:

```sh
curl -I http://whoami.127.0.0.1.nip.io
```

The response should be:

```
HTTP/1.1 200 OK
Content-Length: 1939
Content-Type: text/html; charset=utf-8
Date: Wed, 22 Jun 2022 13:39:02 GMT
Server: Werkzeug/1.0.1 Python/3.9.2
```

## Thank You

Thanks for reading, feel free to check out my [website](https://ruan.dev/), feel free to subscribe to my [newsletter](http://digests.ruanbekker.com/?via=ruanbekker-blog) or follow me at [@ruanbekker](https://twitter.com/ruanbekker) on Twitter.
