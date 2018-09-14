# Testing

I'm learning a ton of new things from this project: namespaces and cgroups; concept of layering file systems; general Linux systems things; and Go!

Adding a test suite is too much "new" (at least, for now).

Instead, I'm going to document some of the manual tests I did to see my code was working as expected.

In the future I might automate these.

## Reexec
Added some print statements showing the PID of the parent and child processes.

```
$ go run container.go
Hello, I am main with pid 8111
Hello, I am container with pid 8115
I am exec
```

## Mount namespace

### Namespace
I can compare the mount namespace inside and outside the container:
```
vagrant@ubuntu-xenial$ ls -lh /proc/self/ns/mnt
lrwxrwxrwx 1 vagrant vagrant 0 Sep 12 03:02 /proc/self/ns/mnt -> mnt:[4026531840]
```
```
vagrant@ubuntu-xenial$ sudo ./go-container
Hello, I am main with pid 8153
Hello, I am container with pid 8157
root@ubuntu-xenial# ls -lh /proc/self/ns/mnt
lrwxrwxrwx 1 root root 0 Sep 12 03:02 /proc/self/ns/mnt -> mnt:[4026532129]
```

### Private mounts
```
vagrant@ubuntu-xenial$ sudo ./go-container
Hello, I am main with pid 8279
Hello, I am container with pid 8283
root@ubuntu-xenial# mkdir /mnt/iamprivate
root@ubuntu-xenial# mount -t tmpfs tmpfs /mnt/iamprivate
root@ubuntu-xenial# grep iamprivate /proc/mounts
tmpfs /mnt/iamprivate tmpfs rw,relatime 0 0
```

In another process outside the container:
```
vagrant@ubuntu-xenial$ grep iamprivate /proc/mounts
vagrant@ubuntu-xenial$
```

## Images and containers
Run the container and check that there is a copy of the image. This will get better.

```
vagrant@ubuntu-xenial$ ls containers
vagrant@ubuntu-xenial$ sudo ./go-container
Hello, I am main with pid 20991
Hello, I am container with pid 20996
root@ubuntu-xenial# ls containers/e01a04e8-6e30-4d5a-a99c-a722b02bad04/rootfs/
bin  dev  etc  home  lib  media  mnt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

## Pivot root
We had to change our shell to `/bin/sh` because alpine doesn't have bash. We can see we have a new view of the file system now.

Now there's nothing in some file systems like /proc and /sys, so we can't see our hosts mounts that way anymore.
```
vagrant@ubuntu-xenial:~/go/src/github.com/jmuia/go-container$ sudo ./go-container
Hello, I am main with pid 21656
Hello, I am container with pid 21661
/ # ls
bin    dev    etc    home   lib    media  mnt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # mount
mount: no /proc/mounts
/ # pwd
/
/ # exit
```

## Mount special file systems
We can see interesting things in our special filesystems now and devtmpfs has created a bunch of devices for us.

```
vagrant@ubuntu-xenial:~/go/src/github.com/jmuia/go-container$ sudo ./go-container
Hello, I am main with pid 22987
Hello, I am container with pid 22992
/ # ls /sys
block       class       devices     fs          kernel      power
bus         dev         firmware    hypervisor  module
/ # mount
home_vagrant_go_src_github.com_jmuia_go-container on / type vboxsf (rw,nodev,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
devtmpfs on /dev type devtmpfs (rw,nosuid,relatime,size=498876k,nr_inodes=124719,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,mode=600,ptmxmode=000)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,relatime)
/ # ls /dev
autofs              mapper              tty0                tty36               tty63               ttyS4
block               mcelog              tty1                tty37               tty7                ttyS5
...
loop6               stdout              tty34               tty61               ttyS30              zero
loop7               tty                 tty35               tty62               ttyS31
/ # ps | head
PID   USER     TIME  COMMAND
    1 root      0:06 {systemd} /sbin/init
    2 root      0:00 [kthreadd]
    3 root      0:00 [ksoftirqd/0]
    5 root      0:00 [kworker/0:0H]
    7 root      0:01 [rcu_sched]
    8 root      0:00 [rcu_bh]
    9 root      0:00 [migration/0]
   10 root      0:00 [watchdog/0]
   11 root      0:00 [watchdog/1]
```

We still see host processes, but we'll deal with that later.

We also see `home_vagrant_go_src_github.com_jmuia_go-container on / type vboxsf (rw,nodev,relatime)` in /proc/mounts.
That's a host mount? I happen to be using a Vagrant VM for testing, and I have the golang workspace setup as a shared folder.

If instead I mount a `tmpfs` to get around the `pivot_root` requirement:
```
// bind mount containerRoot to itself to circumvent pivot_root requirement.
if err := syscall.Mount("tmpfs", containerRoot, "tmpfs", 0, ""); err != nil {
    panic(fmt.Sprintf("Error changing root file system (mount tmpfs containerRoot): %s\n", err))
}
```
I see `tmpfs on / type tmpfs (rw,relatime)` instead.

I think I get it. `home_vagrant_go_src_github.com_jmuia_go-container` is just the "device" that's mounted. So even though in the call to `mount(2)` the source "device" is `containerRoot`, the actual bind mount will have the same device as the original. That makes sense when you consider what bind mounts actually accomplish.

We'll come back to making this better.


## UTS namespace
I've set the hostname to the container id (and updated the PS1, for fun). Since we're in a new UTS namespace, the host won't be affected.
```
vagrant@ubuntu-xenial:~/go/src/github.com/jmuia/go-container$ sudo ./go-container 
Hello, I am main with pid 3727
Hello, I am container with pid 3732
root@76e03801-23ed-4c71-a1ba-c47f94811d0d$ hostname
76e03801-23ed-4c71-a1ba-c47f94811d0d
root@76e03801-23ed-4c71-a1ba-c47f94811d0d$ hostname container
root@76e03801-23ed-4c71-a1ba-c47f94811d0d$ hostname
container
root@76e03801-23ed-4c71-a1ba-c47f94811d0d$ exit
vagrant@ubuntu-xenial:~/go/src/github.com/jmuia/go-container$ hostname
ubuntu-xenial
```
We can also compare the inode number in `/proc/self/ns/uts` for the container and the host.
