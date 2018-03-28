# Using Wercker to build an image from a Dockerfile

This example shows how wercker can be used to build a docker image from a Dockerfile and push it to a image registry.

The application is a simple Go application, the same as in [getting-started-golang](https://github.com/wercker/getting-started-golang).
It listens for HTTP requests on port 5000. Access it using `curl` or a browser, and it will reply with some text.

# Setup

You'll need docker and the [Wercker CLI](http://www.wercker.com/cli) on your machine, as well as the `git` and `curl` commands.

The image registry used in this particular example is [Docker Hub](https://hub.docker.com/), so you'll need to obtain a free Docker Hub account and have your user name and password ready.  

Open a command window, clone this repository and `cd` into it.
```
git clone https://github.com/nigeldeakin/docker-build-golang.git
cd docker-build-golang
```

# Build, push and run the image using the docker command 

Before you try using Wercker, let's try building and pushing the image using the docker command directly.
We will then do exactly the same thing using wercker. 

If you like, you can skip this section and go straight on [Build and run the image using wercker](#build-and-run-the-image-using-wercker)

## Dockerfile

First of all take a look at [Dockerfile](Dockerfile) in this directory:
``` Dockerfile
FROM golang  
WORKDIR /work
ADD . .
RUN go test ./...
RUN go build -o /bin/myapp .
WORKDIR /
RUN rm -r /work
CMD ["/bin/myapp"]  
```
This is a simple single-stage Dockerfile. Using a golang base image, it runs the tests and then builds the executable, which it writes to `/bin/myapp`.
It then deletes the source code and tests.

Wercker also supports multi-stage Dockerfiles. For an example see [Multistage.md](Multistage.md)

## Build (using docker command)

Now use the docker command directly to build the image. We'll later see how to do exactly the same thing in a wercker pipeline.
```
docker build . -t my-image
```
This will build an image using the Dockerfile in this directory and apply the tag `my-image`.

## Push (using docker command)

Before you can push this image to the DockerHub image registry you need to login. Set the following environment variables to hold your Docker Hub user name and password. 
``` bash
export X_USERNAME=<dockerhub-username>
export X_PASSWORD=<dockerhub-password>
```

Now you can push your image. This involves using `docker login` to set your Docker Hub credentials, `docker tag` to specify where to push it to, and `docker push` to perform the push.
In the following command, replace 
``` bash
docker login -u $X_USERNAME -p $X_PASSWORD
docker tag my-image $X_USERNAME/docker-build-golang:latest
docker push $X_USERNAME/docker-build-golang
```

## Run

Now run the new image
```
docker run --rm -p 5000:5000 $X_USERNAME/docker-build-golang
```
This will start your image in the foreground.

In another command window, access the application 
```
curl localhost:5000
```
this will return
```
Hello World!
```
Finally press Control+C in the first window to terminate the application and remove the container.

# Build and run the image using wercker 

Now let's use Wercker to build an image using the same Dockerfile and push it to the image registry.

## wercker.yml

First of all take a look at [wercker.yml](wercker.yml) in this directory:
``` yml
build:
  box: google/golang
  steps:
    # Test the project
    - script:
        name: Run tests
        code: go test ./...     
    - internal/docker-build: 
        dockerfile: Dockerfile 
        tag: my-new-image # temporary tag used to refer to this image in a subsequent step
    - internal/docker-push: 
        image: my-new-image
        username: $USERNAME # Docker Hub username. When using CLI, set using "export X_USERNAME=<username>"  
        password: $PASSWORD # Docker Hub password. When using CLI, set using "export X_PASSWORD=<password>" 
        registry: https://hub.docker.com
        repository: $USERNAME/docker-build-golang
        tag: latest
```
This defines a Wercker pipeline called `build` that 
* runs the tests 
* uses the `internal/docker-build` step to build the image using the Dockerfile 
* uses the `internal/docker-push` step to tag the image and push it to the image registry

When running the wercker CLI the values of `$USERNAME` and `$PASSWORD` are obtained from the environment variables `X_USERNAME` and `X_PASSWORD`.
If you have not already done so, set these now:

``` bash
export X_USERNAME=<dockerhub-username>
export X_PASSWORD=<dockerhub-password>
```
Note that when running in wercker.com Wercker provides a secure way to configure and save these variables. 

## Build, tag and push (using wercker CLI)

Now run the `build` pipeline in `wercker.yml`:
```
wercker build
```
This will build an image using the Dockerfile in this directory and push it to the image registry.

## Run

As before, run the new image
```
docker run --rm -p 5000:5000 $X_USERNAME/docker-build-golang
```
This will start your image in the foreground.

In another command window, access the application 
```
curl localhost:5000
```
this will return
```
Hello World!
```
Finally press Control+C in the first window to terminate the application and remove the container.

# What's next?

In this example you've used the wercker CLI and the `internal/docker-build` and `internal/docker-push` step to create a docker image from a Dockerfile and push it to an image registry.

A typical Wercker pipeline for building an image from a Dockerfile would 
* use `internal/docker-build` to build an image
* use `internal/docker-run` to start the image in a new container
* test the image
* use `internal/docker-kill` to terminate the container
* use `internal/docker-push` to publish the new image to your chosen docker image registry.

---
Sign up for Wercker: http://www.wercker.com

Learn more at: http://devcenter.wercker.com
