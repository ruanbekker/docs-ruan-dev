# Minio

Minio is a self-hosted object storage service similar to AWS S3 as it uses the same API

[![made-with-Markdown](https://img.shields.io/badge/Visit%20my-Website-orange.svg)](https://ruan.dev) [![FollowMe](https://img.shields.io/badge/Follow%20Me-@ruanbekker-00ACEE.svg)](https://twitter.com/ruanbekker)

## Minio with Traefik

Our `docker-compose.yml` which includes traefik reverse proxy:

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

  minio:
    image: minio/minio
    container_name: minio
    command: server /export
    environment:
      - MINIO_ACCESS_KEY=${MINIO_ACCESS_KEY:-myusername}
      - MINIO_SECRET_KEY=${MINIO_SECRET_KEY:-mypassword}
    volumes:
      - minio-data:/export
    networks:
      - contained
    depends_on:
      - traefik
    labels:
      - "traefik.http.routers.minio.rule=Host(`minio.localdns.xyz`)"
      - "traefik.http.services.minio.loadbalancer.server.port=9000"

volumes:
  minio-data:
    name: minio-data

networks:
  contained:
    name: contained
```

Boot the stack:

```sh
docker-compose -f docker-compose.yml up -d
```

## Access Minio via the UI

You can access Minio at `http://minio.localdns.xyz` and the username and password specified in the environment variables

## Access Minio via the API

Let's use the awscli tools to interact with Minio, first set the credentials:

```sh
export AWS_ACCESS_KEY_ID=myusername
export AWS_SECRET_ACCESS_KEY=mypassword
export AWS_DEFAULT_REGION=us-east-1
```

Now lets create a bucket:

```sh
aws --endpoint-url http://minio.localdns.xyz s3 mb s3://my-minio-bucket
```

Then list your buckets:

```sh
aws --endpoint-url http://minio.localdns.xyz s3 ls /
```
```
# output:
2021-03-31 16:53:16 my-minio-bucket
```

Put an object to Minio:

```sh
echo ok > file.txt
aws --endpoint-url http://minio.localdns.xyz s3 cp file.txt s3://my-minio-bucket/output/file.txt
```

Get an object from Minio:

```sh
aws --endpoint-url http://minio.localdns.xyz s3 cp s3://my-minio-bucket/output/file.txt ./download.txt
```

To learn more from Minio, see their [documentation](https://docs.min.io/docs/minio-quickstart-guide.html)
