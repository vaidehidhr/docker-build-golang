# Using Wercker to build an image from a multi-stage Dockerfile

This example also contains a two-stage Dockerfile [Dockerfile-multistage](Dockerfile-multistage) and a corresponding [wercker-multistage.yml](wercker-multistage.yml).

The use of multi-stage Dockerfiles requires Docker 17.05. 
To run this example your local Docker daemon must be this version or later.

At the time of writing wercker.com does not support this version of Docker,
so you cannot yet use `docker-build` in wercker.com with multi-stage Dockerfiles.

``` Dockerfile
# Step #1 Run unit tests and build an executable that doesn't require the go librraies
FROM golang as firststage
WORKDIR /work
ADD . .
RUN go test ./...
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o myapp .
#
# Step #2: Copy the executable into a minimal image (less than 5MB) which doesn't contain the build tools and artifacts
FROM alpine:latest  
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=firststage /work/myapp .
CMD ["./myapp"]  
```
This is a simple two-stage docker build. The first stage, which uses a golang base image, runs the tests and then builds the executable.
The second stage, which uses an alpine linux image, simply copies the executable from the previous stage into the new image.
The result of the second stage is a very small image that does not contain any source code or golang build tools.

# Build, push and run the image using the docker command 

To build the image using the docker command:
```
docker build . -f Dockerfile-multistage -t my-image  
``` 
See the main [README](README.md) for the subsequent steps to tag, push, run and test this image.

# Build and run the image using wercker 

This example also contains a file [wercker-multistage.yml](wercker-multistage.yml) which uses [Dockerfile-multistage](Dockerfile-multistage) to build the image. 

To build and push the image using Wercker:
```
wercker build --wercker-yml wercker-multistage.yml
```
See the main [README](README.md) for the environment variables you need to set first, and for the subsequent steps to run and test this image.

---
Sign up for Wercker: http://www.wercker.com

Learn more at: http://devcenter.wercker.com
