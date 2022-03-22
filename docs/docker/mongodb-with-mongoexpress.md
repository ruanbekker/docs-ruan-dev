# MongoDB and Mongo Express on Docker

This stack will boot a mongodb and mongo-express container

## Compose 

The `docker-compose.yml`:

```yaml
version: "3.8"

services:
  mongodb:
    image: mongo:4.4
    container_name: mongodb
    restart: unless-stopped
    command: mongod --wiredTigerCacheSizeGB 1 --oplogSize 128 --replSet rs0
    volumes:
      - ./data/mongo/data/db:/data/db
      - ./data/mongo/data/backups:/dump
    networks:
      - mongodb
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

  mongo-init-replica:
    image: mongo:4.4
    container_name: mono-init-replica
    command: >
      bash -c
        "for i in `seq 1 30`; do
          mongo mongodb/admin --eval \"
            rs.initiate({
              _id: 'rs0',
              members: [ { _id: 0, host: 'localhost:27017' } ]})\" &&
          s=$$? && break || s=$$?;
          echo \"Tried $$i times. Waiting 5 secs...\";
          sleep 5;
        done; (exit $$s)"
    depends_on:
      - mongodb
    networks:
      - mongodb
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

  mongo-express:
    image: mongo-express
    container_name: mongo-express
    restart: unless-stopped
    environment:
      - ME_CONFIG_MONGODB_URL=mongodb://mongodb:27017/
      - ME_CONFIG_MONGODB_ENABLE_ADMIN=true
      - ME_CONFIG_BASICAUTH_USERNAME=admin
      - ME_CONFIG_BASICAUTH_PASSWORD=admin
    ports:
      - 18080:8081
    networks:
      - mongodb
    depends_on:
      - mongodb
    logging:
      driver: "json-file"
      options:
        max-size: "1m"

networks:
  mongodb:
    name: mongodb
```

## Boot

Boot the stack with docker-compose:

```bash
docker-compose up -d
```

## Access MongoDB

For the UI, you can access Mongo Express on port `18080` and the admin password will be `admin`. 

For the CLI, you can access mongodb using docker:

```
docker-compose exec mongodb mongo
```

You will be dropped into the mongo shell:

```
MongoDB shell version v4.4.12
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("0358a67a-1cf5-40ae-b1b8-c2d5b7bfec0d") }
MongoDB server version: 4.4.12
Welcome to the MongoDB shell.
For interactive help, type "help".
---
rs0:PRIMARY>
```

For cheatsheets you can follow:
- https://github.com/ruanbekker/cheatsheets/tree/master/mongodb

