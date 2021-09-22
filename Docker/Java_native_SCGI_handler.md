# Intro
This small project serves two purposes:

* a small PoC project for my jSCGI library
* to try out Docker with native Java app

The idea is to create a small HTTP POST form handler. The frontend is hosted via Nginx which connects to a SCGI server. For the SCGI server I will be using my jSCGI implementation of it (https://github.com/szabogabriel/jSCGI) as a basis and the additional logic will reside in a new project. After testing out the library, a native version of the Java application will be created and run in a separate Docker image.

We will end up with 4 Docker images. Nginx, Java runtime via JVM, Java runtime with native, Graalvm compiler.

## Docker preparations

We are going to work with two Docker containers which will be exchanging data. To achieve this, we need to have a network set up.

`$ docker network create scgi-test`

This creates the `scgi-test` network. The only thing we need to remember is to set the correct network aliases for the started containers so we can reference them. We also must specify the network upon starting the given container by providing the `--network` attribute.

## SCGI Java application

The Java applicaiton in its simplest form could be a simple Main class which would only echo data received via POST. By using the jSCGI library that would look something like this:

    package scgi.form;
    
    import java.io.IOException;
    import jscgi.server.SCGIServer;
    
    public class Main {
    
        private static final String RESPONSE_HEADER = "HTTP/1.1 200 OK\n";
        private static final String CONTENT_TYPE = "Content-Type: %s\n";
        private static final String CONTENT_LENGTH = "Content-Length: %d\n";
        private static final String FINISH_HEADER = "\n";
    
        public static void main(String[] args) {
            try {
                new SCGIServer(9000, (req, res, mode) -> {
                    try {
                        String message = "Hello, %s!";
    
                        if (req.isBodyAvailable()) {
                            message = String.format(message, new String(req.getBody()));
                        } else {
                            message = String.format(message, "World");
                        }
    					
                        res.write(RESPONSE_HEADER.getBytes());
                        res.write(String.format(CONTENT_TYPE, "text/plain").getBytes());
                        res.write(String.format(CONTENT_LENGTH, message.getBytes().length).getBytes());
                        res.write(FINISH_HEADER.getBytes());
                        res.write(message.getBytes());
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                });
            } catch (IOException e1) {
                e1.printStackTrace();
            }
        }
    }

After creating an `app.jar` from this file which includes the jSCGI library as well, we can create a new Docker container by using Oracle's Java image which will serve this app.

    FROM openjdk:16-alpine3.13
    WORKDIR /app
    COPY ./app.jar .
    CMD ["java", "-cp", "app.jar", "scgi.form.Main"]

This will pull the OpenJDK and copy our `app.jar` into the working directory. Let us build this image and then try running a container. We must include it into the correct network, and set a network alias to be used by Nginx to forward the request to.

`$ docker build --tag scgi-test .`

and then

`$ docker run --rm --network scgi-test --network-alias scgi-server scgi-test`

The network is set via `--network scgi-test` and the network alias for this container is `--network-alias scgi-server`.

## Nginx

Let's start with setting up the Nginx container (https://hub.docker.com/_/nginx/  /_) with SCGI (https://nginx.org/en/docs/http/ngx_http_scgi_module.html). First we can start exploring the Nginx image.

`$ docker run --rm -v /some/content:/usr/share/nginx/html:ro -p 8080:80 -d nginx`

This will host the `/some/content` folder by using volumen mount (`-v`) as read only (`:ro`) on the forwarded port 8080 (`-p 8080:80`) and remove the container after finishing (`--rm`).

Now we can create our custom `nginx.conf` file. We can take a minimal Nginx configuration file and add our SCGI forward to it.

    worker_processes  5;
    worker_rlimit_nofile 8192;
    
    events {
      worker_connections  4096;  ## Default: 1024
    }
    
    http {
      index    index.html index.htm index.php;

      default_type application/octet-stream;
      log_format   main '$remote_addr - $remote_user [$time_local]  $status '
        '"$request" $body_bytes_sent "$http_referer" '
        '"$http_user_agent" "$http_x_forwarded_for"';
      sendfile     on;
      tcp_nopush   on;
      server_names_hash_bucket_size 128; # this seems to be required for some vhosts

      server { # simple load balancing
        listen          80;

        location / {
          include   scgi_params;
          scgi_pass scgi-server:9000;
        }
      }
    }

This will include all the `scgi_params` standardly defined by Nginx and forward all the requests to the `scgi-server` host at port `9000`. The `scgi-server` is the network alias provided when starting our `scgi-test` container in the previous chapter.

Now we can start an Nginx container by mounting our custom `nginx.conf` file into it.

`$ docker run --rm --network scgi-test --network-alias nginx -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro -p 8080:80 nginx:latest`

We could have omit `--network-alias nginx`, but once again we must include the `--network scgi-test` attribute to include this container into the correct network and thus be able to reach the `scgi-server` container.

Now we can test our application.

    $ curl -X POST  -d "Darling" http://localhost:8080/
    Hello, Darling!

## Create the native Java application

This part is based on this tutorial: https://blog.softwaremill.com/small-fast-docker-images-using-graalvms-native-image-99c0bc92e70b and information in the officiall GraalVM https://www.graalvm.org/docs/getting-started/container-images/, https://www.graalvm.org/reference-manual/native-image/StaticImages/ and https://www.graalvm.org/reference-manual/native-image/.

Now we want to create a native runnable application. For that, we will use the `native-image` utility. We could install everyting locally and create the image on our own machines. There are some small issues with this, however. First, we will have to install not only the GraalVM but also some additional features, like the `native-image` tool and some other 3rd party libraries like `musl` and `zlib` to be able to statically link every library into our application. Also, we must be running on a Linux system (https://www.graalvm.org/reference-manual/native-image/StaticImages/).

The easier way is, to take an already set up Docker image and use the container to create the native runnable. The dockerfile would look like this.

    FROM ghcr.io/graalvm/graalvm-ce:latest
    WORKDIR /opt/graalvm
    RUN gu install native-image
    ENTRYPOINT ["native-image"]

We take a community edition of the GraalVM image, install the `native-image` utility and set it as an entry point. Let's create the image.

`$ docker build -t graalvm-native-image .`

Now, using this image, we can build our own native Java application. The preparation for this is, to have our application packaged into a JAR file. I am using a single JAR which holds all the dependencies, but using several library files should be also possible by listing them all. I put the JAR file to be converted into the `./srcpath` folder and I also created a `./target` folder to hold the created runnable application.

`$ docker run --rm -it -v $(pwd)/srcpath:/opt/cp -v $(pwd)/target:/opt/graalvm graalvm-native-image "--static" "-H:Name=out" "-H:+ReportExceptionStackTraces" "-jar" "/opt/cp/app.jar" "scgi.form.Main"`

We are mounting the `./srcpath` and `./target` folders to image-specific folders. `/opt/graalvm` will be used by the `native-image` tool to output the created application.
* `--static` is used to statically link libraries.
* `-H:Name=out` outputs the name of the application into a txt file.
* `-H:+ReportExceptionStackTraces` outputs a full stack trace upon error.
* `-jar` and `/opt/cp/app.jar` specifies the path to the JAR file to be compiled.
* `scgi.form.Main` is the Main class.

After running this command, we should have a native runnable application. We should be able to start it normally on a Linux system by using the `./scgi.form.Main` command.

## Creating an image with our native application

The last step is to create an image with our native application in it. For this, we will use Alpine linux as a basis. The dockerfile would look like thi:

    FROM alpine:3.9.4
    COPY scgi.form.Main /opt/docker/sfh
    RUN chmod +x /opt/docker/sfh
    ENTRYPOINT ["/opt/docker/sfh"]

We are taking Alpine linux, copy our native application into it, make sure it is runnable and set it as a main entry point. Let's build the image.

`$ docker build --tag scgi-test-native .`

Now we can start the container with our native app.

`$ docker run --rm --network scgi-test --network-alias scgi-server scgi-test-native`

Once again, we include our container into the correct network and set the correct network alias. Stop the previously running Docker image running our `scgi-test` container (the JAR-version). The result of a curl call will be the same as previously.

# Results

It is rather interesting to compare the created images.

    $ docker images
    REPOSITORY                   TAG             IMAGE ID       CREATED             SIZE
    scgi-test-native             latest          9e5331fca88f   About an hour ago   28MB
    graalvm-native-image         latest          a70b6578b397   2 hours ago         1.34GB
    scgi-test                    latest          27dc7036e36b   21 hours ago        324MB
    nginx                        latest          4f380adfc10f   2 weeks ago         133MB

As we can see, the image holding the full JVM with our application included is 324MB. Our native image is only 8.6% of the JVM version. The drawback is holding another 1.34GB to be able to create the native application. Also, creating the application itself takes a longer time. Distributing such images is however significantly faster.

After saving and zipping the image, we receive a signifantly smaller image - only 5.2% of the JVM-based image.

    $ docker save scgi-test-native > scgi-native-image.tar
    $ docker save scgi-test > scgi-image.tar
    $ gzip scgi-native-image.tar
    $ gzip scgi-image.tar
    $ ls -hal scgi-*.tar.gz
    -rw-rw-r-- 1 user1 user1 9.4M Jul  9 12:10 scgi-native-image.tar.gz
    -rw-rw-r-- 1 user1 user1 181M Jul  9 12:13 scgi-image.tar.gz

To make this fair however, we must add, that having a more complex application would also mean, that we are going to end up with a larger native application file as well, but the JAR-files themselves won't grow as rapidly. So with a more complex application, these differences would be less significant.

Also, probably the biggest difference is not in the image size, but in the startup time of the application.
