# SSH into container

This tutorial is a backup of "https://www.cyberciti.biz/faq/how-to-install-openssh-server-on-alpine-linux-including-docker/". 

## Create entrypoint shell script 'entrypoint.sh'

```
#!/bin/sh
ssh-keygen -A
exec /usr/sbin/sshd -D -e "$@"
```

## Create Dockerfile

```
FROM alpine:latest
RUN apk add --update --no-cache openssh 
RUN echo 'PasswordAuthentication yes' >> /etc/ssh/sshd_config
RUN adduser -h /home/myUser -s /bin/sh -D myUser
RUN echo -n 'myUser:myPassword' | chpasswd
ENTRYPOINT ["/entrypoint.sh"]
EXPOSE 22
COPY entrypoint.sh /
```

## Build image

```
podman  build -t alpine-sshd .
```

## Run image

Since we are using Podman in rootless mode, we are binding the port 2002.

```
podman run --rm --name sshd_app -d -p 2002:22 alpine-sshd:latest 
```

## SSH into container

```
ssh -p 2002 myUser@localhost
```
