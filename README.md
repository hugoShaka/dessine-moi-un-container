# Dessine moi un container

## What is it ?

A list of steps one can do with a bash shell and a linux machine to understand
what is a container and how they are managed.

Name is a reference to https://github.com/jpetazzo/dessine-moi-un-cluster.

# Creating a first namespace: changing the hostname with a UTS namespace

## Requirements
* a linux machine
* bash
* python (2 or 3)

## Howto

Let's start with a root linux shell and check our hostname with the `hostname` program.
```
#> sudo -sE
#> hostname
DESKTOP-2GRFJ11
```

Now, we'll spawn a new `sh` process in a new UTS namespace (all other namespaces are inherited from ours).
*Note: it could have been bash instead of sh, but changing shell emphasis this is a new process and not the same.*

```
#> unshare -u sh
```

Let's edit the hostname. As there's no standard linux util to do it we'll craft the system call by hand in python. We'll run the `sethostname` syscall https://man7.org/linux/man-pages/man2/sethostname.2.html. According to the documentation it's the syscall number 170.

```
#> python
Python 2.7.17 (default, Nov  7 2019, 10:07:09)
[GCC 7.4.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> # ctypes allows python to call functions from C libraries
>>> # here we'll get the syscall function of the libc library
>>> import ctypes
>>> libc = ctypes.CDLL(None)
>>> syscall = libc.syscall
>>> # for python 2
>>> new_hostname = "not-my-computer"
>>> # for python 3
>>> new_hostname = b"not-my-computer"
>>> syscall(170, new_hostname, len(new_hostname))
0
>>> # mission is a success, let's exit python
>>> exit(0)
```

We can then use the `hostname` binary to read the hostname.

```
#> hostname
not-my-computer
```

The hostname has been changed in our namespace. Now if we close bash and get back to the bash outside the namespace we just created we can see the hostname has not been changed for it.
```
#> exit
#> hostname
DESKTOP-2GRFJ11
```

# Creating a cgroup

## Requirements
* a linux machine
* bash

*Note: I only tested this on cgroups v1*

## Howto

Open a root shell

```
#> sudo -sE
```

Cgroups are controlled via files. Something called a cgroup controller will take action based on file you create.

First, we want to find where croup controllers are mounted:

```
#> mount | grep cgroup
```

It usually is `/sys/fs/cgroup/`

We'll create our PID cgroup

```
#> cd /sys/fs/cgroup/pids
#> mkdir mycgroup
#> cd mycgroup
#> ls
cgroup.clone_children  cgroup.procs  notify_on_release  pids.current  pids.events  pids.max  tasks
```

A bunch of files appeared in the directory and will allow us to interact with the cgroup settings.

`tasks` contains the PIDs of processes subject to this cgroup.

We can add our current shell PID in the tasks

```
#> cat tasks
# we see there's no-one in
#> echo "$$" >> tasks
#> cat tasks
# There are now two PIDs, the one from your shell and the one doing the `cat`
```

Then we can restrict the max amount of PIDs to somehting way smaller than the default

```
#> echo 3 > pids.max
```

Then, here comes the leap of faith. Let's try to fork bomb ourselves.

```
#>  :(){ :|:& };:
```

Hopefully this should not break your linux box and throw a couple of errors because bash does not have enough pids.

