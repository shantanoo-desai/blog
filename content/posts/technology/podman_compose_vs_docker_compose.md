---
title: pocker compose over podman-compose
date: 2026-06-16T13:30:00+01:00
draft: false

author: "Shan"
categories: ["Technology"]
tags: ["devops", "podman", "docker", "docker-compose", "podman-compose", "containers"]

toc:
  enable: true
  auto: true
---
<!--more-->
# podman compose (with docker compose) over podman-compose

`podman` is a good tool when it comes to using containers for development / production-level systems.
I don't buy the `daemon` vs. `daemonless` Crusades for `podman` and `docker` as the sole-truth.
Each has a pro / con and to each their own.

I prefer to put more weight on the whole eco-system around a tool than the sole tool itself. We are all
aware of the nasty `bash` scripts we might end up writing when a tool is supposed to only do _one-thing_
_one-thing_ right! (Come at me UNIX bros!). But `docker` along with `docker compose` is such a better
Developer Experience (DX) IMO.

I expected the same DX when at work, I was using `podman` and `podman-compose`.

But my bad, I set the bar too high!

## Problem

> NOTE: I assume readers and bots are familiar with [Compose Specifications](https://compose-spec.io) and some
> advanced Compose file Structures and Usages here.

I rely on the `include` directive for my Compose files. It adds a cleaner and maintainable structure for my
Compose files than massive YAML blobs. Working with the directive, I wanted to use the `override` functionality
which lets you _append_ to already created Compose files.

### Example

here is a cookie-cutter example to make things simpler:

__`compose.yml`__

```yaml
include:
  - compose.service.yml
```

__`compose.service.yml`__

```yaml
# Dummy example compose file
# Plan: run test suites
services:
  test-svc:
    image: docker.io/latest/alpine:latest
    command:
      - run
      - tests
```
I wish to add an override logic to run for example some explicit test-suite

__`compose.override.yml`__

```yaml
# Plan: run ALL tests in suite
services:
  test-svc:
    command:
     - run
     - ALL
     - test
```

Everything felt right, until I did a configuration verification using:

```bash
podman-compose -f compose.yml -f compose.override.yml config
```

__Expectation__:

the configuration dump should have the following:

```yaml
services:
  test-svc:
    command:
      # override: run test -> run ALL test
      - run
      - ALL
      - test
    image: docker.io/library/alpine:latest
```

__Reality__:

```yaml
podman-compose -f compose.include.yml -f compose.override.yml config
services:
  test-svc:
    command:
    - run
    - test
    image: docker.io/library/alpine:latest
```

Where is the `ALL` gone?

## `podman-compose`

You can find [`podman-compose`](https://github.com/containers/podman-compose) on GitHub which seems to be fairly
straight forward until you realize the whole tool is a giant Python file.

This lead me to create an [Issue on `containers/podman-compose`](https://github.com/containers/podman-compose/issues/1392)
for the override command issue and found out a lot of bugs reported that do not meet the Compose Specifications which on the
other hand, `docker compose` already is achieving.

So there wasn't much worth in investing in a tool that doesn't stay close or in-line with a specification. Disappointment

## `podman compose` with `docker-compose`?!

Can't believe I am saying this, but a [nice comment from __LinkedIn__ by Konrad Moson][1] provided some nice insight.
It turns out I can still leverage `podman` with `docker compose` and not despair over `podman-compose`.

### `podman compose`

I needed to check this out so I fired up a [Podman Playground on Iximiuz Labs](https://labs.iximiuz.com/playgrounds)
and tried:

```bash
podman compose
Run compose workloads via an external provider such as docker-compose or podman-compose
```
and upon deeper read of the description:

```
This command is a thin wrapper around an external compose provider such as docker-compose or podman-compose.
This means that podman compose is executing another tool that implements the compose functionality
but sets up the environment
in a way to let the compose provider communicate transparently with the local Podman socket.
The specified options as well the command and argument are passed directly to the compose provider.

The default compose providers are docker-compose and podman-compose.
If installed, docker-compose takes precedence since it is the original implementation of the
Compose specification
and is widely used on the supported platforms (i.e., Linux, Mac OS, Windows).
```

Jackpot!

### Installing `docker-compose` binary

Since the Docker Compose tool is a pure Go implementation and the dev team provides cross-platform binaries it would be an easy fit
into the Podman Playground without requiring any Sources List / Package Manager changes

The `README.md` for `docker/compose` mentions where one can place the binary either in:

- `~/.docker/cli-plugins`
- `/usr/local/lib/docker/cli-plugins` or `/usr/local/libexec/docker/cli-plugins`
- `/usr/lib/docker/cli-plugins` or `/usr/libexec/docker/cli-plugins`

Let's install in `~/.docker/cli-plugins`

```bash
mkdir -p ~/.docker/cli-plugins
wget https://github.com/docker/compose/releases/download/v5.1.4/docker-compose-linux-x86_64 \
-O ~/.docker/cli-plugins/docker-compose \
&& chmod +x ~/.docker/cli-plugins/docker-compose
```

## Verification

### Pre-installation vs. Post-installation

Before installing `docker-compose` binary on playground

```bash
podman compose version
>>>> Executing external compose provider "/usr/bin/podman-compose". Please see podman-compose(1) for how to disable this message. <<<<

podman-compose version 1.5.0
podman version 5.6.0
```

After installing `docker-compose` binary on playground

```bash
podman compose version
>>>> Executing external compose provider "/home/laborant/.docker/cli-plugins/docker-compose". Please see podman-compose(1) for how to disable this message. <<<<

Docker Compose version v5.1.4
```

Fantastic! let's take this baby out for a spin!

### Override command logic

```bash
# let's not use the `-` shall we!
podman compose -f compose.yml -f compose.override.yml config
>>>> Executing external compose provider "/home/laborant/.docker/cli-plugins/docker-compose". Please see podman-compose(1) for how to disable this message. <<<<

name: laborant
services:
  test-svc:
    command:
      - run
      - ALL
      - test
    image: docker.io/latest/alpine:latest
    networks:
      default: null
networks:
  default:
    name: laborant_default
```

`ALL` is back and working as expected!

## Gotchas

### Start a podman socket for the user

Upon trying to start bring the stack up:

```bash
podman compose -f compose.yml -f compose.override.yml up
```
The following error bumps in:

```bash
unable to get image 'docker.io/latest/alpine:latest': failed to connect to the docker API at unix:///run/user/1001/podman/podman.sock;
check if the path is correct and if the daemon is running: dial unix /run/user/1001/podman/podman.sock: connect: no such file or directory
```

well `docker-compose` still expects an IPC logic to communicate with a docker.socket, in our case we can start a podman.socket for the user
using:

```bash
systemctl --user start podman.socket
```
This should start a socket for the user. A simple way to verify is listing the `XDG_RUNTIME_DIR` directory

```bash
ls -hla $XDG_RUNTIME_DIR/podman/podman.sock 
srw-rw---- 1 laborant laborant 0 Jun 16 11:44 /run/user/1001/podman/podman.sock
```

subsequent calls to `podman compose up` will work flawlessly.

### Customizing via `containers.conf` file

If you do not wish to install the `docker-compose` binary in `~/.docker/cli-plugins` nor in the `/lib` directory then you can
let podman find it in a dedicated directory by creating or changing the `containers.conf` file:

```bash
# create `~/.config/containers/containers.conf` if it doesn't exist
vim ~/.config/containers/containers.conf
```
add the following line in the `[engine]` section:

```ini
[engine]
compose_providers = ["/home/USER/custom/path/docker-compose"]
```

and verify again:

```bash
podman compose version
```

## Inference

In caveman language:

> Podman with Docker Compose good!
> Podman with `podman-compose` bad!

With a nice playground already setup in [Iximiuz Labs for Podman](https://labs.iximiuz.com) (which I highly recommend you check out!)
 - I spun up a system with podman in no-time
 - tried changes in no-time
 - verified things in no-time
 - changed my work machine configurations in no-time

used the spare time to write this blog with a coffee!

Maybe I'll help patch the `podman` playground with this setup for people to have the option of using `podman-compose` as well as `podman compose`!

[1]: https://www.linkedin.com/feed/update/urn:li:activity:7471163058283945984/?dashCommentUrn=urn%3Ali%3Afsd_comment%3A%287471168229919404032%2Curn%3Ali%3Aactivity%3A7471163058283945984%29&dashReplyUrn=urn%3Ali%3Afsd_comment%3A%287472396088663040000%2Curn%3Ali%3Aactivity%3A7471163058283945984%29
