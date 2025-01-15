---
title: "on Docker Compose Configs"
date: "2025-01-15T21:01:35+01:00"
draft: false

author: "Shan"
tags: ["devops", "docker", "docker-compose", "traefik"]
categories : ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
# Trying out `configs` spec in Docker Compose v2

As a part of the working on [Komponist][1] I was looking into
the updated [Docker Compose Specs][2] and it supports some nice
features like [`include`][3] specification which essentially makes
your Compose files split up into individual service files and an
entry-point YAML file which includes the these services as if they
were _header_ files, making container stacks more modular and manageable.

One more interesting spec I am currently looking into including in my
project is the [`configs`][4] specification, which adds any sort of
configuration files needed for the container to run - integrated into
the compose YAML. In a nutshell, configurations can be now part of the
the container's compose YAML file and there can be less files to mount
into the container. In many cases, a container's Compose YAML file can
be the complete __rule__ that describes how it should be configured and
run.

## Example

As a simple example I wanted to setup a `traefik` reverse-proxy with
a Basic Authentication logic for an example `whoami` container. Nothing
complicated, nothing fancy.

We will use the `include` spec along with `configs` in specific
Compose files.

Below is the Compose YAML file for the `whoami` service configured
to have basic authentication when trying to reach it via command-line
tools like `curl` or by hitting the url `whoami.localhost` in the browser.
The authentication user and password will exist in the `/etc/traefik/adminuser`
file.

```yaml
# docker-compose.whoami.yml
# Description: Whoami container as an example

services:
  whoami:
    image: traefik/whoami:latest # don't use latest for prod!
    container_name: whoami-test
    security_opt:
      - "no-new-privileges=true"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.localhost`)
      - "traefik.http.routers.entrypoints=web"
      - "traefik.http.routers.middlewares=whoamiBasicAuth"
      - "traefik.http.middlewares.whoamiBasicAuth.basicauth.usersfile=/etc/traefik/adminuser"
```

The `traefik` reverse-proxy container is configured the standard way
to listen on the `localhost` ports for HTTP (80, 8080). The `configs`
section describes the admin user and the __BCrypt__ encrypted password
for the plain-text value `testPaSS`.

```yaml
# docker-compose.traefik.yml
# Description: Reverse-Proxy to listen on localhost HTTP ports
services:
  traefik:
    image: traefik:latest # don't use latest for prod!
    container_name: reverse-proxy-traefik
    ports:
      - "127.0.0.1:80:80"
      - "127.0.0.1:8080:8080"
    labels:
      - "traefik.enable=true"
    security_opt:
      - "no-new-privileges=true"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - "--api-insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entryPoints.web.address=:80"
    # Configs to be made available to container on runtime
    configs:
      - source: traefik_basicauth_adminuser
        target: /etc/traefik/adminuser

configs:
  traefik_basicauth_adminuser:
    content: |
      admin:$2y$08$1iQ3sGBP1ef/hAKTk2vjAOK8b66cULtTZ54lDR/4We6fpVK3SQGEq
```
{{< admonition type=info title="encrypting creds in terminal" open=true >}}
htpasswd -nb -B -C 8 admin testPaSS
{{< /admonition >}}

The main entrypoint compose YAML file will just need to include
the two above services in it using `include` as follows:

```yaml
# compose.yml
# Description: main entrypoint compose file
name: testproject
include:
  - ./docker-compose.traefik.yml
  - ./docker-compose.whoami.yml
```

### Configuration check

In order to be sure that there isn't any misconfiguration perform
the following command in the directory where the `compose.yml` file exists:

```bash
docker compose config
```

This command should provide a valid YAML structure in your `stdout`.

If there is any error there will output available with specific lines
mentioning where the potential error might be.

### Testing

bringing the services up:

```bash
docker compose up -d
```

the `whoami` service is now available behind the reverse-proxy under the
`whoami.localhost` domain.

If things are correctly configured, we can perform an __HTTP GET__ and we
get a `401 Unauthorized`

```bash
> curl http://whoami.localhost/
401 Unauthorized

> curl -u admin:testPaSS http://whoami.localhost/
Hostname: f1771c6c9f56
IP: 127.0.0.1
IP: ::1
IP: 172.18.0.2
RemoteAddr: 172.18.0.3:55496
GET / HTTP/1.1
Host: whoami.localhost
User-Agent: curl/8.11.1
Accept: */*
Accept-Encoding: gzip
Authorization: Basic YWRtaW46dGVzdFBhU1M=
X-Forwarded-For: 172.18.0.1
X-Forwarded-Host: whoami.localhost
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: db3ea9878cc6
X-Real-Ip: 172.18.0.1
```

## When `content` in file won't work

One needs to be careful when it comes to `$` in Compose files.
`$` is used an interpolation logic in compose files, which could
mean that any value after a `$` symbol may be interpreted as if 
it were an environment variable that Compose will try to resolve
/ interpolate.

In some cases, the `docker compose config` command will throw a
warning stating the value after `$` cannot be interpolated and
an empty value will be used instead.

This will also be evident with the `stdout` output of the
`docker compose config` command, where anything post the `$`
may be left empty. In such cases, one can rely on the `file`
parameters instead of `content` in the Compose file.

### Alternate way for the user creds in Example

```yaml
# docker-compose.traefik.yml
services:
  traefik:
    image: traefik:latest
    container_name: reverse-proxy-traefik
    ports:
      - "127.0.0.1:80:80"
      - "127.0.0.1:8080:8080"
    labels:
      - "traefik.enable=true"
    security_opt:
      - "no-new-privileges=true"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - "--api-insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entryPoints.web.address=:80"
    # Configs to be made available to container on runtime
    configs:
      - source: traefik_basicauth_adminuser
        target: /etc/traefik/adminuser

configs:
  traefik_basicauth_adminuser:
    file: ./adminuser # adminuser file contains the bcrypt creds in them
```

This will require an external file in the directory called `adminuser`
with the encrypted credentials in them and obtain the same results.
More than one way to skin a cat!

## Inference

`configs` spec for Compose is really great! you don't have to worry
about weird volume file-mounts, but you can try to make you service's compose
file hold almost all of the things required to run it!.

Pair it up with `secrets` spec and you can also handle credentials
in a safer way!

A well-designed spec for sure!


[1]: https://github.com/shantanoo-desai/komponist
[2]: https://compose-spec.io
[3]: https://github.com/compose-spec/compose-spec/blob/main/14-include.md
[4]: https://github.com/compose-spec/compose-spec/blob/main/08-configs.md
[5]: https://www.youtube.com/watch?v=A4275MAu7JY
