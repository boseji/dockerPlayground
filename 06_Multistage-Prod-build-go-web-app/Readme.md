# Example of Production Multi-stage build of Golang web app

This examples builds upon the Last 2 examples:

 1. [04_golang-webapp-with-glide-support](https://github.com/boseji/dockerPlayground/tree/master/04_golang-webapp-with-glide-support) Explaining the ***Glide*** package management

 2. [05_Live-Reload-golang-web-app](https://github.com/boseji/dockerPlayground/tree/master/05_Live-Reload-golang-web-app) Explaining Live Reload via the [codegangsta/Gin](https://github.com/codegangsta/gin) tool.

The main intention of this example is to Show 
  
  *  How build a Go web app into Executable

  *  How the Multi Stage build works

  *  What are the problems of the default `Multisage Build` as on date (Feb'2018)

  *  How to Improve upon and create a fix for non-removable intermediate docker images

## Building the Go Web App for Production

Lets get started by first understanding how we are going to build an executable.

Typically a Golang executable needs to built for the specific environment/OS.

```shell
go build src/main.go
```

This should typically build it for our Host. However the build image would take
the name of the directory.

In this case `06_Multistage-Prod-build-go-web-app` if you are building this particular directory.

Thats not ideal and we need a proper name for our app.

So we improve the command:

```shell
go build -o main src/main.go
```

This command would build the things into an executable named `main` and we can run this by typing `./main`.

However we need to be very specific about how we want the executable to work.
Since we are actually going to do this inside the *Docker Container*.

Here is the Improved upon command:

```shell
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main src/main.go
```

Here are the Details:

  * `CGO_ENABLED=0` disables the `cgo` compiler (not sure about this one !). It was added from the default build example at https://golang.org

  * `GOOS=linux` Set the target build operation to generate a linux binary

  * `-a` flag forces rebuilding of packages that are already up-to-date

  * `-installsuffix cgo` renames the build dependencies and directory with added suffix to separate them from default builds. In this case we are using `cgo` and the suffix.

  * `-o main` specifies the output file name

Lets now do and build a docker Image:

[**ex1.Dockerfile**](https://github.com/boseji/dockerPlayground/blob/master/06_Multistage-Prod-build-go-web-app/ex1.Dockerfile)

```Dockerfile
FROM golang:1.8
# install glide
RUN go get github.com/Masterminds/glide
# Create the Working Directory
WORKDIR /go/src/app
# add glide.yaml and glide.lock
ADD glide.yaml glide.yaml
ADD glide.lock glide.lock
# install packages
RUN glide install
# add source code
ADD src src
# build the source
#RUN go build src/main.go
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main src/main.go
# Finally Execute the Program
CMD ["./main"]
```

We can `docker build` this by using:

```shell
docker build -t go-docker-prod -f ex1.Dockerfile .
```

The above command would specifically target the file `ex1.Dockerfile` and use the same directory as the default root of operations.

Lets now run the newly created docker container:

```shell
docker run --rm -it -p 8080:8080 go-docker-prod
```

This command is wrapped in the file `run-docker-prod-app.sh`.

This above command should work successfully ans show the following log message:

```shell
2018/02/21 02:29:06 Starting Server (with httprouter) on Port 8080
```

Thats fine a great but there is some thing to note.

Lets now look at **Size** of the docker container created for production:

```shell
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
go-docker-prod      latest              8799f9509e95        4 minutes ago       736MB
golang              1.8                 0d283eb41a92        3 days ago          713MB
alpine              3.7                 3fd9065eaf02        6 weeks ago         4.15MB
```

Notice that our `go-docker-prod` is **736 MB** !!

That's not very good, we would ideally like to have a *smaller Image*.
Since there might be other service container and even the *smallest VMs* you can buy at $5 don't have much space and memory.

## Building Lean production Image

We now we know how to build the file inside the docker image.
But due to the size of the default parent image we can't use our earlier image for production.

Typically at this stage we might need to find a proper parent image which is lean enough.

Here are some candidates at https://hub.docker.com/ :

  *  [baseimage-docker](https://phusion.github.io/baseimage-docker/)
  This is a trimmed down version of the ubuntu image. However the size is still not that great.

  * [alpine](https://hub.docker.com/_/alpine/) Official repository of the Alpine linux. It's a very small but functional distro at **5MB** size.

  * [busybox](https://hub.docker.com/_/busybox/) Official repository of the Busybox base Image. This image is again very small at **4MB** size but lacks the full back end package management.

So from the above 3 we would select `alpine` since it has package management and is small.

Lets first create our custom image to build the Golang app:

```shell
docker build -t gobuilder -f ex1.Dockerfile .
```

We use the previous Dockerfile itself and create a new image `gobuilder` as our final production image will now be some thing different.

And then Run this -- 

```shell
docker run --rm -it -v "$PWD"/dist:/dist gobuilder cp ./main /dist/
```

to get the executable File into the `dist` directory on the HOST.

Now we need to create our `alpine` docker image for production:

[**ex2.Dockerfile**](https://github.com/boseji/dockerPlayground/blob/master/06_Multistage-Prod-build-go-web-app/ex2.Dockerfile)

```Dockerfile
FROM alpine:3.7
# add ca-certificates in case you need them
RUN apk update && apk add ca-certificates && rm -rf /var/cache/apk/*
# set working directory
WORKDIR /app
# copy the binary from builder
ADD dist/main /app/main
# run the binary
CMD ["./main"]
```

Build this:

```shell
docker build -t go-docker-prod -f ex2.Dockerfile .
```

Finally Run It:

```shell
$ docker run --rm -it -p 8080:8080 go-docker-prod
2018/02/21 03:28:58 Starting Server (with httprouter) on Port 8080
```

We now have a working `alpine` Image with our app.

Lets look at the Size:

```shell
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
go-docker-prod      latest              93bce994d76d        3 minutes ago       10.6MB
gobuilder           latest              57a13e16446c        10 minutes ago      736MB
golang              1.8                 0d283eb41a92        3 days ago          713MB
alpine              3.7                 3fd9065eaf02        6 weeks ago         4.15MB
```

Wow the `go-docker-prod` is only **10.6 MB** !!

That's Great !!

But the whole process of doing things to times. 

  * Build run 2 times 
  
  * 2 Separate docker files and corresponding images
  
  * Maintain a copy of the executable on the Host

  * Cleanup needed 

This is kind of cumbersome. 

So lets clean up the earlier images and look at another method.

## Multi-Stage Build

We have a solution to our problems :

*Welcome the Docker Multi-stage build*

For more details: https://docs.docker.com/develop/develop-images/multistage-build/

In our case we are having one build image and then pushing the executable into the actual production image. Docker allows creation of multiple images in the same docker file. But we need to have some way of identifying and using the specific images and their contents.

The Multi-stage build solves this adding 3 important features:

  *  Stage the Image creation using a special `as` keyword in the `FROM` line.<br>This helps to **'tag'** name a specific build image stage.<br><br>
  e.g. `FROM golang:1.8` becomes `FROM golang:1.8 as builder` this means that we can now reference this in other images.

  *  `COPY --from=builder /go/src/github.com/alexellis/href-counter/app .`<br><br>
  Here the `--from` flag names the previews build stage.<br>
  Also one can do this from an external image:<br><br>
  `COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf`

  *  Stop at specific build stage:<br><br>
  `docker build --target builder -t alexellis2/href-counter:latest . `<br><br>
  Here the build would stop at `builder` stage only.

Now Lets modify and combine our two `Dockerfile` examples:

[**ex3.Dockerfile**](https://github.com/boseji/dockerPlayground/blob/master/06_Multistage-Prod-build-go-web-app/ex3.Dockerfile)

```Dockerfile
FROM golang:1.8 as builder
# install glide
RUN go get github.com/Masterminds/glide
# Create the Working Directory
WORKDIR /go/src/app
# add glide.yaml and glide.lock
ADD glide.yaml glide.yaml
ADD glide.lock glide.lock
# install packages
RUN glide install
# add source code
ADD src src
# build the source
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main src/main.go

FROM alpine:3.7 as production
# add ca-certificates in case you need them
RUN apk update && apk add ca-certificates && rm -rf /var/cache/apk/*
# set working directory
WORKDIR /app
# copy the binary from builder
COPY --from=builder /go/src/app/main .
# run the binary
CMD ["./main"]
```

Lets build this new file:

```shell
docker build -t go-docker-prod -f ex3.Dockerfile .
```

Now lets run the same:

```shell
$ docker run --rm -it -p 8080:8080 go-docker-prod
2018/02/21 03:57:54 Starting Server (with httprouter) on Port 8080
```

Great ! now we have a same functionality only with a Single Command and One unified docker file.

Lets now look at the images created:

```shell
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
go-docker-prod      latest              a1778c484601        4 minutes ago       10.6MB
<none>              <none>              5c7def6e1073        4 minutes ago       736MB
golang              1.8                 0d283eb41a92        3 days ago          713MB
alpine              3.7                 3fd9065eaf02        6 weeks ago         4.15MB
``` 

We do have our correct `go-docker-prod` image with size of **10.6 MB** again.

But wait, we notice a strange Image:

`<none>              <none>              5c7def6e1073        4 minutes ago       736MB`

Now this is created with our `go-docker-prod` build.

This particular image is basically the intermediate 'builder' Image that we created to get to the final production image.

**Note: Docker does remove the Intermediate containers but does not remove intermediate build images.This is default at the time (Feb'2018)**

This is more to this:

https://github.com/moby/moby/issues/34151

https://forums.docker.com/t/how-to-remove-none-images-after-building/7050

There are many suggested methods to remove this final wrinkle.

## Improving upon default 'Multi-Stage' Builds

We found that tagging intermediate builds is the best way to remove the `<none>` images.

We first created out combo `Dockerfile`

[**combi.Dockerfile**](https://github.com/boseji/dockerPlayground/blob/master/06_Multistage-Prod-build-go-web-app/combo.Dockerfile)

```Dockerfile
# Builder Image for Making the Executable
FROM golang:1.8 as builder
# install glide
RUN go get github.com/Masterminds/glide
# Create the Working Directory
WORKDIR /go/src/app
# add glide.yaml and glide.lock
ADD glide.yaml glide.yaml
ADD glide.lock glide.lock
# install packages
RUN glide install
# add source code
ADD src src
# build the source
RUN go build src/main.go
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main src/main.go

# Minimum System For the Production Image
FROM alpine:3.7 as production
# add ca-certificates in case you need them
RUN apk update && apk add ca-certificates && rm -rf /var/cache/apk/*
# set working directory
WORKDIR /root
# copy the binary from builder - note Here the External Image Tag is used
#   do not confuse this with the 'builder' tag used above
COPY --from=gobuilder:latest /go/src/app/main .
# run the binary
CMD ["./main"]
```

Notice here we have used 2 build stage tags `builder` and `production`. You can multiple stages cascaded in this way.

Now lets build it in a staged fashion:

  1. Build and tag the `builder` Image as `gobuilder`
```shell
docker build --target builder -t gobuilder -f combo.Dockerfile .
```

  2. Build and tag the `production` Image as `go-docker-prod`
```shell
docker build --target production -t go-docker-prod -f combo.Dockerfile .
```
  **Note: Even though we ask for `production` stage , docker builds both the `builder` and the `production` together. This is default at the time (Feb'2018)**
  No need to worry as docker has the 'builder' image cached so nothing will change there.

  3. Remove the 'builder' image
```shell
docker rmi gobuilder
```

All the above steps have been packaged in a single script file:

[**build-prod-docker.sh**](https://github.com/boseji/dockerPlayground/blob/master/06_Multistage-Prod-build-go-web-app/build-prod-docker.sh)

Now lets look at the images:

```shell
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED                  SIZE
go-docker-prod      latest              3c3443fa02fb        Less than a second ago   10.6MB
golang              1.8                 0d283eb41a92        3 days ago               713MB
alpine              3.7                 3fd9065eaf02        6 weeks ago              4.15MB
```

Well do not have the un-needed intermediate image now.

And our `go-docker-prod` is created satisfactory with the **10.6 MB** size.

Lets test out the App:

```shell
$ docker run --rm -it -p 8080:8080 go-docker-prod
2018/02/21 04:20:50 Starting Server (with httprouter) on Port 8080
```

Yay:dizzy:! The app works and now with a compact and clean build system.:laughing:

we have packaged the run command into the script file `run-docker-prod-app.sh`.
