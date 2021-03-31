# Traefik 

Traefik is a Modern HTTP Reverse Proxy and Load Balancer that makes deploying microservices easily.

## The Proxy

To run a traefik service with docker-compose, the section will look like this:

```yaml
...
  traefik:
    image: traefik:v2.4.5
    container_name: traefik
    command: [ '--providers.docker', '--api.insecure' ]
    networks:
      - contained
    ports:
      - 80:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - "traefik.http.routers.traefik.rule=Host(`traefik.localdns.xyz`)"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"
...
```

As you can see we are exposing port `80` as we are only running this example in `http`, mounting the `docker.sock` so that traefik can read docker events and read information about other containers.

Then the labels is where the magic happens, we are defining the host rule `traefik.localdns.xyz` to route to port `8080` on the traefik service.

## Reverse Proxy to a Web App

Now if we want to use traefik as a reverse proxy to a web application, we define the bit like this:

```yaml
...
  web-center-name:
    image: ruanbekker/web-center-name-v2
    container_name: web-center-name
    environment:
      - APP_TITLE=Welcome
      - APP_URL=https://ruan.dev
      - APP_TEXT=Visit my Website
    networks:
      - contained
    depends_on:
      - traefik
    labels:
      - "traefik.http.routers.minio.rule=Host(`www.localdns.xyz`)"
      - "traefik.http.services.minio.loadbalancer.server.port=5000"
...
```

The full example of our `docker-compose.yml` will look like this:

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v2.4.5
    container_name: traefik
    command: [ '--providers.docker', '--api.insecure' ]
    networks:
      - contained
    ports:
      - 80:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - "traefik.http.routers.traefik.rule=Host(`traefik.localdns.xyz`)"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"

  web-center-name:
    image: ruanbekker/web-center-name-v2
    container_name: web-center-name
    environment:
      - APP_TITLE=Welcome
      - APP_URL=https://ruan.dev
      - APP_TEXT=Visit my Website
    networks:
      - contained
    depends_on:
      - traefik
    labels:
      - "traefik.http.routers.minio.rule=Host(`www.localdns.xyz`)"
      - "traefik.http.services.minio.loadbalancer.server.port=5000"

networks:
  contained:
    name: contained
```

To boot our service:

```sh
docker-compose -f docker-compose.yml up -d
```

And testing our application:

```sh
curl -I http://www.localdns.xyz
```
```
HTTP/1.1 200 OK
Content-Length: 1939
Content-Type: text/html; charset=utf-8
Date: Wed, 31 Mar 2021 14:46:22 GMT
Server: Werkzeug/1.0.1 Python/3.9.2
```
