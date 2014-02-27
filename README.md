
Intro
=====

orq is a really really simple docker-based server manager. It does three main
things:

- Pushes locally-built images to a server (without messing with registries).
- Acts as a process monitor for running docker containers on a server.
- Automatically configures a proxy (hipache) to point to those containers.

It's intended for sharing small servers among small numbers of apps packaged in
separate containers. It's not intended for managing large numbers of servers.

The `orq` script is written in Python and is completely self-contained (besides
an upstart config to keep the daemon running), so it's very simple to install
(or uninstall).


Quick start
===========

First, set up docker on a server. You're on your own here.

Then:

```
$ ./install server.com
```

This will copy the script and upstart config, and make sure python-redis is
installed. The script currently assumes something Ubuntu-ish. Do it manually if
you want.

```
$ ssh server.com start orq
```

This will start the orq daemon, which will then pull, build, and run a couple of
containers, one for hipache, and one for a private registry.

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
  "host": "server.com"
}]
$ ./orq run myapp.orq
```

The `run` command will do the following:

- Do a `docker build` of the directory given as `dockerdir`.
- ssh to `server.com` and set up a couple ssh tunnels.
- Do a `docker push` to push the newly-built image to the registry on the
  server, over the ssh tunnel.
- Contact the orq daemon and tell it to [re]start the image.
- The daemon will start the new image in a new container.
- Then it will update the hipache config to point `myapp.com` to port 3000 on
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
- `dockerdir`: The directory containing the Dockerfile for this task. If
  missing, the build step will be skipped.
- `domain`: The domain to register as a virtual host for hipache. If missing,
  don't mess with hipache.
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

Don't use this:

- `exposed_ports`: Expose public ports. The intention is for hipache to be the
  only public port, and everything else to be proxied through there.


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


CLI
===

`orq run myapp.orq` runs all tasks in the file as described above.

`orq stop myapp.orq` stops all tasks in the file.

`orq cleanup server.com` cleans up unused containers and images on the server.


Rationale
=========

Pros:

- It's very simple and self-contained.
- Integrated deployment, process management, and proxy configuration.
- All image-building happens locally and images get pushed to servers. If it
  works locally, it'll work on the server.
- All communication is over ssh.
- Zero-downtime deploys.

Cons:

- It's missing lots of features.
- It's not suitable for managing large numbers of servers.


TODO
====

- SSL
- Easy backups of specific volumes
- Image build dependencies (images depending on intermediate images)
- Running multiple copies of a task
- Log rotation for hipache?
- Links? (unix sockets in shared directories are easier)
- Allow switching nginx, etc. for hipache



License: MIT
============

Copyright (C) 2014 David Reiss

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

