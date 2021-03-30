# Python Flask with Redis Docker Example

## About Python Flask

Flask is a micro web framework written in Python. It is classified as a 
microframework because it does not require particular tools or libraries.

## About Redis

Redis is an in-memory data structure store, used as a distributed, 
in-memory key–value database, cache and message broker, with optional durability.

## About Docker

Docker is a set of platform as a service (PaaS) products that use 
OS-level virtualization to deliver software in packages called containers. 

Containers are isolated from one another and bundle their own software, 
libraries and configuration files, they can communicate with each other 
through well-defined channels


## More information about Flask, Redis and Docker

* [Flask](https://flask.palletsprojects.com/en/latest/)
* [Redis](https://redis.io/)
* [Docker](https://www.docker.com/)

## Instructions

This example will show you how to dockerize a python flask application that 
connects to a redis container using docker.

### Python Flask App

Our python flask application code: `app.py`:

```python
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host="redis")

@app.route("/")
def hello():
    visits = redis.incr('counter')
    html = "<h3>Hello World!</h3>" \
           "<b>Visits:</b> {visits}" \
           "<br/>"
    return html.format(visits=visits)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

Our `requirements.txt`:

```text
Flask
Redis
```

### Our Docker Code

First we create our `Dockerfile`:

```dockerfile
FROM python:3.7

WORKDIR /app

ADD requirements.txt /app/requirements.txt
RUN pip install -r requirements.txt

ADD app.py /app/app.py

EXPOSE 80

CMD ["python3", "app.py"]
```

And our `docker-compose.yml`:

```yaml
web:
  build: .
  dockerfile: Dockerfile
  links:
    - redis
  ports:
    - "80:80"
redis:
  image: redis
```

### Build, Deploy, Test

Now that we have everything sorter, lets build our application:

```bash
$ docker-compose -f docker-compose.yml build
```

Then run our containers:

```bash
$ docker-compose -f docker-compose.yml up -d
```

Now test our application:

```bash
$ curl http://localhost:80/
<h3>Hello World!</h3><b>Visits:</b> 1<br/>
```

### Tear Down

Tear down the application:

```bash
$ docker-compose -f docker-compose.yml down
```
