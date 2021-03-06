# go-container

A basic container runtime and container management system; developed for learning purposes; written in Go*.

The time spent coding, researching, and debugging errors is much more valuable than this code will ever be.

<small>\* first time ever writing Go; be wary!</small>

## Asciicast demos

* [Basics](https://asciinema.org/a/drxibVCMaW78ZaOwM4rxPc029)
* [Networking](https://asciinema.org/a/KtseNx1zxPQfa8cAs1r4vDhdp)
* [CPU Shares cgroup](https://asciinema.org/a/92bTMJT75CYCNkOSwvDnpvjVg)
* [Memory Limit cgroup](https://asciinema.org/a/xC9wQYNNSKmgL7m3bFQlOQgjj)
* [Overlay FS](https://asciinema.org/a/yZdwOnGJlD8MWo9TtfUCrtULT)

## Features

#### Namespaces and cgroups
Executes processes in separate mount, UTS, PID, IPC, and network namespaces.

Provides option to control resource usage with the CPU shares and memory limit cgroups.

#### `pivot_root` jail and private mounts
"Jails" processes with a `pivot_root`, limiting their view of the file system.

Mount changes don't propagate between the host and container.

#### Special file systems and devices
Special file systems and devices are created per the [Open Containers spec for Linux](https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md).

`/proc`, `/dev`, `/sys` and more are mounted.
Devices like `/dev/null`, `/dev/urandom`, etc. are also created.

#### Copy on write containers
Manages container images and creates copy-on-write copies using the Overlay file system.

#### Bridge network
Creates a veth pair for each container and adds them to the `goContainers0` bridge.
The host system can communicate with the containers and the containers can communicate with each other (if added to the same subnet).

#### Environment
Sets up a clean environment with it's very own hostname and a fancy PS1.

## Usage
```
Usage: sudo ./go-container [OPTIONS] <image name> <command>
  -bridge-addr string
    	CIDR bridge address; replaces current if present (default "10.10.10.1/24")
  -c string
    	directory to store containers (default "containers")
  -container-addr string
    	CIDR container veth address (default "10.10.10.2/24")
  -cpu int
    	cpu shares (relative weight)
  -i string
    	directory to find container images (default "images")
  -mem string
    	memory limit in bytes; suffixes can be used
```

Alpine Linux is included in the repository. A basic example looks like:
```
sudo ./go-container alpine /bin/sh
```

## Run/Install
The easiest way to try it out is using Vagrant (ironically, I had trouble with Docker capabilities). If you want to do it yourself, use the steps in the `Vagrantfile` as a guide.

```
# this may take a while to apt-get update
vagrant up --provision

# ssh into the vm
vagrant ssh

# run as root, in root's GOPATH
sudo su
cd /root/go/src/github.com/jmuia/go-container/

# run a container
./go-container alpine /bin/sh
```

## What's missing?
* Containers must be run as root and privileges are not dropped when exec'ing the process.
* User namespaces.
* Cgroup namespaces (unavailable in Go).
* Idiomatic Go. go-container is my first time ever using Go; it's not particularly nice code.
* Tests!
* More features... Docker does a lot.

## Testing
I'm learning a ton of new things from this project: namespaces and cgroups; concept of layering file systems; general Linux systems things; and Go! Adding a test suite is too much "new" (at least, for now).

Instead, I've documented some of the manual testing I did in [`TESTING.md`](TESTING.md).

## Bugs?
Open an issue or submit a PR. `go-container` is not bulletproof. For example, during testing I found odd cases where I couldn't do certain operations on a VirtualBox shared directory mount.

As an aside, Go had some limitations with some syscalls and programming patterns due to aspects of its runtime design.

## Resources
`go-container` was guided by:
* [rubber docker](https://github.com/Fewbytes/rubber-docker) by Avishai Ish-Shalom and Netanel Cohen
* [Linux Namespaces](https://medium.com/@teddyking/linux-namespaces-850489d3ccf) by Ed King
* [Cgroups, namespaces, and beyond: what are containers made from?](https://www.youtube.com/watch?v=sK5i-N34im8) by Jérôme Petazzoni
