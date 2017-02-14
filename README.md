<!--
# Copyright © (C) 2017 Emory Merryman <emory.merryman@gmail.com>
#   This file is part of base.
#
#   base is free software: you can redistribute it and/or modify
#   it under the terms of the GNU General Public License as published by
#   the Free Software Foundation, either version 3 of the License, or
#   (at your option) any later version.
#
#   base is distributed in the hope that it will be useful,
#   but WITHOUT ANY WARRANTY; without even the implied warranty of
#   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#   GNU General Public License for more details.
#
#   You should have received a copy of the GNU General Public License
#   along with base.  If not, see <http://www.gnu.org/licenses/>.
-->
# base

This image is a suitable base image for many projects.

It includes sudo, docker, and a regular user.
Run processes as the regular user (without access to docker).

Inject dependencies using docker.
The regular user invokes a script in '/usr/local/bin' that sudo calls a script in '/usr/local/sbin'.
Use '/etc/sudoers.d' to permit the sudo call.
The script in '/usr/local/sbin' (as root) invokes docker and solves the dependency.

The hope is that this allows us to use docker as a dependency injection framework without exposing too much security vulnerabilities.

## Dependency Injection Framework

If you want to use 'git' with a remote like 'git@github.com:tidyrailroad/base.git' then ssh is necessary.
On a red hat like system, you would execute `dnf install --assumeyes openssh-clients`.
On a ubuntu like system, you would execute `apt-get install --assumeyes openssh-clients`.
On a alpine like system, you would execute `apk add openssh'.

If you need 'ssh' on a container derived from the base image (and there are '/etc/sudoers.d', '/usr/local/bin', and '/usr/local/sbin' mounts) then you could alternatively (because the image is based on alpine you could also use the alpine way) inject the dependency into the volumes.

Dependency injection is more docker like.
Everything becomes a docker process.

Dependency injection allows you to customize the dependencies more easily.
You will have multiple 'ssh' installations and each one can have its own '~/.ssh' directory.

You can use cross platform dependencies.
For example, I am not aware of any package in alpine that installs the 'uuidgen' binary.
This binary is easily installed in redhat like systems with (`dnf install --assumeyes util-linux`)(https://github.com/tidyrailroad/uuidgen/blob/0.0.0/Dockerfile).
Dependency injection allows you to install the 'uuidgen' binary by reference to a (docker image)[https://hub.docker.com/r/tidyrailroad/uuidgen/].

## Security Vulnerabilities

The classic way to run docker from a docker container is ('docker-out-of-docker' dood)[https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/].
A docker container that is running with dood has root access to the host system.

Consider

```
docker run --interactive --tty --volume /var/run/docker.sock:/var/run/docker.sock:ro alpine:3.4 sh
     > evil-program.sh
     ... docker run --interactive --tty --privileged --volume /:/usr/local/src alpine:3.4 sh
          > # we now have root access to the host system
```

The first call to docker did not seem to grant root privileges, but since it can invoke docker, it can obtain any privilege including root.

The remedy we propose has not been thoroughly vetted, but we believe it might work.

The process would look like
```
docker run --interactive --tty --volume /var/run/docker.sock:/var/run/docker.sock:ro --user user tidyrailroad/base:0.2.1 sh
     > good-program.sh
     > some-dependency
     ... invokes /usr/local/bin/some-dependency
     ... sudo /usr/local/sbin/some-dependency.sh (allowed by /etc/sudoers.d/some-dependency)
     ... docker run --interactive --tty --privileged --volume /:/usr/local/src alpine:3.4 some-dependency
     > evil-program.sh
     ... docker run --interactive --tty --privileged --volume /:/usr/local/src alpine:3.4 sh
     ... rejected b/c user user does not have docker access
```
