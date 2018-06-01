# Using  Wercker to build an image from a Dockerfile

This example shows how wercker can be used to build a docker image from a Dockerfile, run the new image, test the image, and finally push the image to a registry.

The application is a simple Go application, the same as in [getting-started-golang](https://github.com/wercker/getting-started-golang).
It listens for HTTP requests on port 5000. Access it using `curl` or a browser, and it will reply with some text.

# Setup

You'll need docker and the [Wercker CLI](http://www.wercker.com/cli) on your machine, as well as the `git` and `curl` commands.

The image registry used in this particular example is [Docker Hub](https://hub.docker.com/), so you'll need to obtain a free Docker Hub account and have your user name and password ready.  

Open a command window, clone this repository and `cd` into it.
```
git clone https://github.com/wercker/docker-build-golang.git
cd docker-build-golang
```

# Build, run, test and push an image using the docker command 

Before you try using Wercker, let's go through the manual steps to build, run, test and push a Docker image. 

You will perform the following manual steps:
* Use the `docker build` command to build a Docker image.
* Use the `docker run` command to start it. 
* Perform a simple test using the `curl` command.
* Use the `docker push` command to tag the image and push it to the Docker Hub image repository.

If you like, you can skip this section and go straight on to [build, run, test and push the image using Wercker](#build-and-run-the-image-using-wercker)

## Set environment variables

Set the following environment variables to hold your Docker Hub user name and password. 
``` bash
export X_USERNAME=<dockerhub-username>
export X_PASSWORD=<dockerhub-password>
```

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

## Run (using docker command)

Now run the new image and test it
```
docker run --rm -p 5000:5000 $X_USERNAME/docker-build-golang
```
This will start your image in the foreground.

## Test

In another command window, access the application 
```
curl localhost:5000
```
this will return
```
Hello World!
```
Finally press Control+C in the first window to terminate the application and remove the container.

## Push (using docker command)

You have tested the new image and confirmed that it works as expected. You can now push it to the DockerHub image registry.

This involves using `docker login` to set your Docker Hub credentials, `docker tag` to specify where to push it to, and `docker push` to perform the push.

Set the following environment variables to hold your Docker Hub user name and password. 
``` bash
export X_USERNAME=<dockerhub-username>
export X_PASSWORD=<dockerhub-password>
```

Now run the following:
``` bash
docker login -u $X_USERNAME -p $X_PASSWORD
docker tag my-image $X_USERNAME/docker-build-golang:latest
docker push $X_USERNAME/docker-build-golang
```

# Build, run, test and push an image using Wercker

Now let's use Wercker to do the same thing.
Wercker will build an image using the same Dockerfile, run and test the new image, and push it to the image registry.

## wercker.yml

First of all take a look at [wercker.yml](wercker.yml) in this directory:
``` yml
build:
  box: google/golang
  steps:
    # Test the project
    - script:
        name: Unit tests
        code: go test ./...     
    - internal/docker-build: 
        dockerfile: Dockerfile 
        image-name: my-new-image # name used to refer to this image until it's pushed   
    - internal/docker-run:
        image: my-new-image
        name: myTestContainer     
    - script: 
        name: Test the container
        code: |
            if curlOutput=`curl -s myTestContainer:5000`; then 
                if [ "$curlOutput" == "Hello World!!" ]; then
                    echo "Test passed: container gave expected response"
                else
                    echo "Test failed: container gave unexpected response: " $curlOutput
                    exit 1
                fi   
            else 
                echo "Test failed: container did not respond"
                exit 1
            fi        
    - internal/docker-kill:
        name: myTestContainer               
    - internal/docker-push: 
        image-name: my-new-image
        username: $USERNAME # Docker Hub username. When using CLI, set using "export X_USERNAME=<username>"  
        password: $PASSWORD # Docker Hub password. When using CLI, set using "export X_PASSWORD=<password>" 
        repository: docker.io/$USERNAME/docker-build-golang
        tag: latest
```
This defines a Wercker pipeline called `build` that 
* runs the unit tests 
* uses the `internal/docker-build` step to build the image using the Dockerfile 
* uses the `internal/docker-run` step to start a container using the newly-built image
* uses a `script` step to test that the container responds to a HTTP request as expected. If this fails the pipeline will be terminated and the image will not be pushed.
* uses the `internal/docker-kill` step to terminate the container 
* uses the `internal/docker-push` step to tag the image and push it to the image registry

When running the wercker CLI the values of `$USERNAME` and `$PASSWORD` are obtained from the environment variables `X_USERNAME` and `X_PASSWORD`.
If you have not already done so, set these now:

``` bash
export X_USERNAME=<dockerhub-username>
export X_PASSWORD=<dockerhub-password>
```
Note that when running in wercker.com Wercker provides a secure way to configure and save these variables. 

## Build, run, test, tag and push (using wercker CLI)

Now run the `build` pipeline in `wercker.yml`:
```
wercker build
```
This will build an image using the Dockerfile in this directory, start the newly-created image, test it, and (if the test is successful) and push the it to the image registry.

---
Sign up for Wercker: http://www.wercker.com

Learn more at: http://devcenter.wercker.com
