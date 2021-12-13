Intro
=====

The goal of this tutorial is a simple showcase of creating a reverse proxy Docker image with custom rewrite rules. It isn't a Docker-deep-dive, but rather an Apache-related topic. Nevertheless, I didn't want to create a single Apache topic...

Preparation
-----------

First we need an Apache Docker image. E.g.

`> docker pull httpd:latest`

The reason we pull this first is to acquire a valid httpd.conf file. Like it is writte in the official Apache Docker image page (https://hub.docker.com/_/httpd):

`> docker run --rm httpd:latest cat /usr/local/apache2/conf/httpd.conf > my-httpd.conf`

Now we can start activating modules and adding our custom config. All the configuration changes are done on this `my-httpd.conf` file.

Reverse proxy
-------------

The following modules were activated (removed the comment hashtag):
```
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

I added the following config:

```
<VirtualHost *:80>
	ProxyPreserveHost Off
	ProxyRequests Off
	ProxyPass "/" "http://server-host:7000/"
	ProxyPassReverse "/" "http://server-host:7000/"
</VirtualHost>
```

The `server-host` is a value I am providing via the runtime parameters (as described in my other tutorial). The webserver we try reverse proxy to is running on port 7000 with no specific host set (e.g. 0.0.0.0). It is important not to run on *localhost* for example, otherwise our dockerized app won't be able to connect to it.

Rewrite rules
-------------

I experimented with `mod_sed` and tried getting it up and running, but didn't succeed. Then I found a great page (https://capttofu.tripod.com/wordpress_bak/) where they described the way how to define external filters. So what that was exactly what I did. But first, I had to activate the module.

`LoadModule ext_filter_module modules/mod_ext_filter.so`

And then add the configuration.

```
ExtFilterDefine external_sed_out mode=output cmd="/bin/sed s/Hello/Bye/g"
ExtFilterDefine external_sed_in mode=input cmd="/bin/sed s/sdf/\"xxx\"/g"

<Location />
	SetOutputFilter external_sed_out
	SetInputFilter external_sed_in
</Location>
```

Building and running the custom image
-------------------------------------

From this point the configuration of Apache HTTP server was ready. We just have to put it together. I tried mounting the custom config file as a volume, but it didn't wanted to work. So I created a simple Dockerfile.

```dockerfile
FROM httpd:latest
COPY ./my-httpd.conf /usr/local/apache2/conf/httpd.conf
```

This can be used to create a custom image.

`> docker build -t myproxy .`

And we can start it via the following command:

`> docker run --rm --add-host=server-host:172.18.0.1 -p 9000:80 myproxy:latest`

As you can see, the *server-host* is defined as it was introduced in the *httpd.conf* file.

Test preparation
----------------

First we need a very sofisticated test data to be sent to the test server. For this I am using the following:

`echo "asdfasdfasdfasdfasdf" > req.dat`

And we also need an HTTP server to send the data to and receive the response from. So I wrote a simple test server using Java's built in webserver which logs the received body (if any) and answers with a 'Hello, world'.

``` java
package com.ibm.sk.ustidnrvergabe.test;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.InetSocketAddress;
import java.util.stream.Collectors;
import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpServer;

public class Server {

  private static String RETURN = "<html><head><title>Test</title><body><p1>Hello, world</p1></body></html>";

  public static void main(final String[] args) throws IOException {
    final HttpServer server = HttpServer.create(new InetSocketAddress(7000), 0);
    server.createContext("/index", Server::handle);
    server.start();
  }

  private static void handle(final HttpExchange exchange) throws IOException {
    System.out.println("Handling request.");
    System.out.println("Received body: " + new BufferedReader(new InputStreamReader(exchange.getRequestBody())).lines().collect(Collectors.joining("\n")));
    exchange.sendResponseHeaders(200, RETURN.getBytes().length);
    exchange.getResponseHeaders().add("Content-Type", "text/html");
    exchange.getResponseBody().write(RETURN.getBytes());
    exchange.getRequestBody().close();
    exchange.getResponseBody().close();
  }
}
```

Testing
-------

First we can confirm, thath the simple Java webserver is working properly.

```bash
> cat req.dat 
asdfasfasdfasfasfds
> curl -X POST --data @req.dat localhost:7000/index
<html><head><title>Test</title><body><p1>Hello, world</p1></body></html>
```

And now we can check that the rewrite is working is well.

```bash
> curl -X POST --data @req.dat localhost:9000/index
<html><head><title>Test</title><body><p1>Bye, world</p1></body></html>
```

In the Java console we can see the following messages:

```
Handling request.
Received body: asdfasfasdfasfasfds
Handling request.
Received body: a"xxx"asfa"xxx"asfasfds
```

The first two lines are when called directly, the second half is via the Apache rewrite proxy.

Results
-------

It is failry simple to achieve reverse proxy functionalites with rewrite by using Apache. However, I am a bit buffled why the `mod_proxy` didn't work for me.
