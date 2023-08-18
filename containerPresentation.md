# How Containers Work Behind the Scenes
## OVERVIEW

Containers are a technology that allows us to run the group of process in an independent environment with other processes on the same computer. So how does the container do that?

To do that, the container is built from a few features of the Linux kernel, of which the two main features are “namespaces” and “cgroups”.

Development of these features was driven in part by the Linux Containers project, LXC, which began at Google in 2006. with more than 30 command-line tools and configuration files, it’s quite complicated.
The first few releases of Docker were essentially user-friendly wrappers that made LXC easier to use

## NAMESPACES

This is a feature of Linux that allows us to create something like a virtual machine, quite similar to the function of virtual machine tools. This main feature makes our process completely separate from the other processes.


- mnt: constructing namespaces that do not have full access to the host's filesystem. allows you to mount and unmount the filesystem without affecting the host filesystem. shows
processes a customized view of the filesystem hierarchy. whenever you create a new mount namespace, a copy of the mount points from the parent namespace is created in the new mount namespace. systemd defaults to recursively sharing the mount points with all new namespaces.

- net: allows us to run the program on any port without conflict with other processes running on the same computer.

- user: allows processes to run as specific users. Among other things, this is useful for creating a namespace as an unprivileged user that can still be root in the namespace. There are obvious limitations regarding the tasks these namespaces can accommodate, as they do not have root on the host. The user namespace is most often used in combination with the other namespaces to provide a greater level of isolation than would otherwise be possible.
Every container inherits its permissions from the user who created the new user namespace.

- PID: isolate the process ID number space, meaning that processes in different PID namespaces can have the same PID. isolating processes from each other. PID namespaces allow containers to provide functionality such as suspending/resuming the set of processes in the container and migrating the container to a new host while the processes inside the container maintain the same PIDs. It also allows running multiple versions of an application that relies on isolated process trees.

- IPC: isolate between different IPC resources like message queues, semaphores and shared memory. all IPC objects created in an “IPC namespace” are visible only to all processes/tasks that are members of the same namespace.
IPC namespace is a feature in Linux that allows processes to have isolated communication channels with each other. It provides a way for processes running on the same host to exchange information and synchronize their activities.
Each IPC namespace has its own set of IPC objects. These objects are isolated within their respective namespace, meaning processes in one namespace cannot directly access or interfere with IPC objects in another namespace.
By using IPC namespaces, applications running in different namespaces can communicate without interference from other processes or namespaces. This allows for better resource management and isolation between different groups of processes within the same host.
  
- UTS: you can change the hostname or the Network Information Service (NIS) domain that a process reports.

## CGROUPS

We could have created a process separate from the other process with Linux namespaces. But if we create multiple namespaces, then how can we limit the resources of each namespace so that it doesn’t take up the resources of another namespace?

Cgroups originally put forward by Google engineers in 2006, cgroups were eventually merged into the Linux kernel around 2007. 

limit the use of system resources and prioritize certain processes over others. Cgroups prevent runaway containers from consuming all available CPU and memory

    sudo cgcreate -g memory:my-process

One folder is created at the path /sys/fs/cgroup/memory.
We will see a lot of files, these are the files that define the limit of the process, the file that we are interested in now is memory.kmem.limit_in_bytes, it will define the memory limit of a process, and the value used in bytes.

Each type of controller (cpu, blkio, memory, etc.) is subdivided into a tree-like structure. Each branch or leaf has its own weights or limits. Each child inherits and is restricted by the limits set on the parent cgroup.

## PRACTICAL

    unshare -Umrpf --net --mount-proc
creates a new process namespace, unshares the user, mount, network and IPC namespaces, and preserves the file system mount points. It also isolates the network stack and mounts the /proc directory for the new namespace.

    mount -t proc proc /proc
mounts the proc filesystem on the /proc directory. The proc filesystem provides information about processes.

    chroot . /bin/sh
changes the root directory to the current directory (.), and then starts a new shell (/bin/sh) in that new root directory. This effectively creates a new isolated environment with its own root filesystem.

### network
    pidof unshare
run on the host machine and returns the process ID of the previous unshare command. It is used to get the process ID of the newly created namespace.

    ip link add name h-if type veth peer name c-if
creates a virtual Ethernet device pair with the names h-if and c-if. These devices will be used for communication between the host and the new namespace.

    ip link set c-if netns cpid 
moves the c-if device into the network namespace of the process with ID cpid. This effectively makes the c-if device available only inside that namespace.

    ip link set h-if master docker0 up
connects the h-if device to the docker0 bridge and brings it up, allowing network communication between the host and the new namespace.

    ip link set lo up
brings up the loopback interface

    ip link set c-if name eth0 up
brings up the network interface eth0

    ip add add 172.17.0.50/16 dev eth0
assigns the IP address 172.17.0.50 with a subnet mask of /16 to the eth0 interface

    ip route add default via 172.17.0.1 
adds a default route

---

[Presentation slide](https://www.canva.com/design/DAFrz2c4PWM/teCifTW8oVhFNRfrpi01xw/view?utm_content=DAFrz2c4PWM&utm_campaign=designshare&utm_medium=link&utm_source=publishsharelink#5)

## References

https://akashrajpurohit.com/blog/build-your-own-docker-with-linux-namespaces-cgroups-and-chroot-handson-guide/

https://www.redhat.com/sysadmin/building-container-namespaces

https://www.redhat.com/sysadmin/mount-namespaces

https://www.redhat.com/sysadmin/pid-namespace

https://www.redhat.com/sysadmin/net-namespaces

https://www.redhat.com/sysadmin/uts-namespace

https://www.redhat.com/sysadmin/cgroups-part-one


