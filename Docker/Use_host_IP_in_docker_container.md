# Use host IP in docker container

When testing in development, we can end up with a mixed environment, where parts of the infrastructure are running in docker containers, and parts of it in a test environment (e.g. from IDE). This gets interesting, when the system under test is the host of a given service.

Such a scenarion could be for example running a SCGI with an Nginx container in front of it.

To enable such a scenario the '--add-host' switch can be used. This addes a mapping into the `/etc/hosts` file in the container itself. This way we can bind a host name used insie the container to an IP address used outside of the container on the host machine.

For example let's have an SCGI forward config file:

```
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
      scgi_pass scgi-host:9000;
    }
  }
}
```

Please note the `scgi-host` in the `scgi_pass` mapping. Now when we run the following docker command:

```
#!/bin/bash

readonly nginx_config=$(pwd)/nginx.conf
readonly nginx_local_port=8080
readonly network=network-name
readonly docker_ip=$(ifconfig docker0 | grep inet | grep -v inet6 | awk '{ print $2}')

docker run --rm --add-host=scgi-host:${docker_ip} -v ${nginx_config}:/etc/nginx/nginx.conf:ro -p ${nginx_local_port}:80 nginx:latest
```
then we will map the `scgi-host` hostname to the external IP address; in our case to the IP stored in the `docker_ip` variable. To verify this, we can print the `/etc/hosts` file in the container.

```
$ docker exec $(docker container ps | grep nginx:latest | awk '{print $1}') cat /etc/hosts

127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.1	scgi-host
172.17.0.2	5fb3fd4960f3
```

