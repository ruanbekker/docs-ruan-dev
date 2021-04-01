# Multistage Go Application

We will use multistage docker builds from a `alpine` image as our build image to get the dependencies, build the go binary and use our `scratch` image to place the built binary onto our target image to have small docker images.

## Go Application

We will use a library that generates random data from [go-randomdata](https://github.com/Pallinder/go-randomdata) in our application, `app.go`:

```go
package main

import (
    "fmt"
    "github.com/Pallinder/go-randomdata"
)

func main() {
    profile := randomdata.GenerateProfile(randomdata.Male | randomdata.Female | randomdata.RandomGender)
    fmt.Printf("The new profile's username is: %s and password (md5): %s\n", profile.Login.Username, profile.Login.Md5)
}
```

## Dockerize the Application

Our multi-stage `Dockerfile`:

```dockerfile
FROM golang:1.11.1 as builder
RUN mkdir -p /go/src/github.com/ruanbekker
WORKDIR /go/src/github.com/ruanbekker
RUN useradd -u 10001 app
COPY . .
RUN go get
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

FROM scratch
COPY --from=builder /go/src/github.com/ruanbekker/main /main
COPY --from=builder /etc/passwd /etc/passwd
USER app
CMD ["/main"]
```

Build the image:

```sh
docker build -t myapp:v2 .
```

Run a container:

```sh
docker run -it myapp:v2
```
```
# output
The new profile's username is: Kingcat and password (md5): 11a8c854fa051287aedc0bb3466e3d44
```
