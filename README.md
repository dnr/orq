
Intro
=====

orq is a really really simple docker-based server manager. It does three main
things:

- Pushes locally-built images to a server (automatically handling registries).
- Acts as a process monitor for running docker containers on a server.
- Automatically configures a proxy (nginx) to point to those containers.

It's intended for sharing small servers among small numbers of apps packaged in
separate containers. It's not intended for managing large numbers of servers.

The `orq` script is written in Python and is completely self-contained, so it's
very simple to install (or uninstall).


Quick start
===========

First, set up docker on a server. You're on your own here.

Then:

```
$ ./orq install server
```

This will copy the script and have it set up a systemd service, and
automatically start. This will then pull, build, and run a couple of
containers, one for nginx, and one for a private registry.

The script currently assumes something Ubuntu-ish. Do it manually if you want.

```
$ cd myapp
$ cat Dockerfile
FROM ...
...
$ cat myapp.orq
[{
  "name": "myapp",
  "dockerdir": ".",
  "domain": "myapp.com",
  "http_port": 3000,
  "host": "server",
  "ssl_cert": "path/to/my.chained.crt",
  "ssl_key": "path/to/my.key",
  "force_ssl": true
}]
$ ./orq run
```

The `run` command will do the following:

- Do a `docker build` of the directory given as `dockerdir`.
- ssh to `server` and set up a couple ssh tunnels.
- Do a `docker push` to push the newly-built image to the registry on the
  server, over the ssh tunnel.
- Contact the orq daemon and tell it to [re]start the image.
- The daemon will start the new image in a new container.
- Then it will update the nginx config to point `myapp.com` to port 3000 on
  the new container.
- Then it will take down the old container.
- The orq daemon checks that all known containers are running every ten seconds,
  and will restart ones that aren't running.


Details
=======

The orq file is JSON. It contains a list of "task" objects. Tasks can have these
keys:

- `name`: An identifying name for the app. This controls which containers
  replace which others.
- `dockerdir`: The directory containing the Dockerfile for this task.
  If both this and `nix-build` are missing, the build step will be skipped.
- `nix-build`: If present, build a docker image using Nix. The value of this
  key is passed on the `nix-build` command line. The Nix build must use
  `dockerTools.streamLayeredImage`, and the result will be piped to `docker
  load` to create the image. For example, `"nix-build": "-A docker"` will build
  the docker image specified by the `docker` attribute in `default.nix` in the
  current directory.
- `domain`: The domain to register as a virtual host for nginx. If missing,
  don't mess with nginx.
- `http_port`: The port the container process is listening on.
- `host`: The server that this task should run on.
- `volumes`: A list of volumes (see below).
- `env`: Environment variables to set {key: value}.
- `argv`: Arguments to add to `docker run` (array).
- `raw_image_name`: If present, just run this image (e.g. an image from the
  public registry).
- `stop_old_first`: Set to true if your container can't handle two instances
  running at once.
- `up_wait_time`: How long to wait (seconds) before directing traffic at the
  new container (default: 1).
- `down_wait_time`: How long to wait (seconds) after directing traffic away
  from an old container before stopping it (default: 1).
- `ssl_cert`: Path to an ssl cert (nginx chained form expected).
- `ssl_key`: Path to an ssl key.
- `force_ssl`: If true, add a 301 redirect in nginx from http to https for this
  domain. Otherwise send all traffic to the service.
- `use_acme`: Request a certificate for this domain with acmetool.
- `health_check`: Path to fetch over http as a health check.
- `serve_static`: Allows some of an application to be served by nginx directly.
  This should be an object with two keys: `url` is the url prefix to be handled,
  and `dir` is the mount point within the application container.

Don't use these:

- `exposed_ports`: Expose public ports. The intention is for nginx to be the
  only public port, and everything else to be proxied through there.
- `restart_policy`: Sets docker restart policy. orq assumes application
  containers will die on failure and that it will restart new ones. If you use
  this, orq will get confused and fail to direct traffic to your application
  (because it won't know when the IP changes).


Volumes
=======

orq also "manages" volumes for you. Volumes are identified with an id. Two tasks
that refer to the same volume id (on the same host) will share the same
directory. This is useful for sharing unix sockets (which is much easier to
manage than docker links).

Volumes are initialized to be owned by root:root with permissions 1777 (like
`/tmp`).

To use volumes:

```
  "volumes": [
      {"id": "myapp_sockets", "container": "/sockets"},
      {"id": "myapp_logs", "container": "/logs"}
  ],
```

The `container` key is the path to mount the volume inside the container.

orq puts all volumes in `/var/lib/orq/volumes`.


ACME
====

Using ACME (LetsEncrypt) requires a little interactive setup. After the nginx
image is running, do `docker exec -it <nginx container> bash`, then `acmetool
quickstart`. Choose "webroot", enter `/certs/acmetool/challenges/` for the
webroot challenge directory, and say no to the cron job. Then you can run a
service with `'use_acme': True`.


Health checks
=============

Example:

```
    "health_check": "/_healthy",
```

If this is present, orq will try to fetch that path over the usual http port
once a second after starting a new container, and will not redirect traffic to
it until it returns http 200. It will give up after 30 seconds.

Currently, health checks are not done after container startup. (That is, it's
more like a "readiness check" for now.)


Static files
============

To serve static files directly:

```
    "serve_static": {
      "url": "/_st",
      "dir": "/static"
    },
```

This will configure nginx to serve paths like `/_st/myfile` directly out of the
directory mounted at `/static` in your container. This will be a fresh bind
mount that will cover any existing directory in your docker image. You should
*copy* all the static files you want to serve into the directory. If you've
configured a health check, make sure you copy them before the health check
returns success.

**Note:** Files will be served with this header for best caching:

```
Cache-Control: public, max-age=31536000, immutable
```

That means you *must* do your own cache-busting in the file path!


CLI
===

`orq run` runs all tasks in the .orq file in the current directory.

`orq run foo bar` runs tasks that contain the strings `foo` or `bar` in their names.

`orq stop` stops all tasks in the .orq file in the current directory.

`orq stop foo bar` stops tasks that contain the strings `foo` or `bar` in their names.

`orq upload_cert` copies the ssl cert (and optionally key) to the
server for inclusion in the nginx config. If the key is readable, push the cert
and key. Otherwise push only the cert. This is the only command that touches the
cert and key files, so you're free to leave them encrypted during normal
operation.

`orq cleanup server.com` cleans up unused containers and images on the server.


Rationale
=========

Pros:

- It's very simple and self-contained.
- Integrated deployment, process management, and proxy configuration.
- All image-building happens locally and images get pushed to servers. If it
  works locally, it'll work on the server.
- All communication is over ssh.
- Health checks.
- Zero-downtime deploys.
- Container logs are automatically saved to a volume on crash/restart.

Cons:

- It's missing lots of features.
- It's not suitable for managing large numbers of servers.


TODO
====

- Easy backups of specific volumes
- Image build dependencies (images depending on intermediate images)
- Running multiple copies of a task
- Log rotation for nginx?



License: MIT
============

Copyright (C) 2014-2020 David Reiss

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

