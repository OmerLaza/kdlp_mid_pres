# Demo
## Create a shell with a new mount namespace

Let's create a new shell that derives all the namespaces from the parent shell besides the mount namespace.
For the mount namespace it will create a new mount namespace that derives the mount list from the parent process.

```console
user@my_pc:~$ sudo unshare -m /bin/bash
root@my_pc:/home/user#
```
The flag `-m` stands for a new mount namespace.

## Check the current process namespace

All namespaces are accessible via `/proc/self/ns/*`. Let's check the mount namespace, there are two ways to do that:
1. Via `ls -l` command:

```console
root@my_pc:/home/user# ls -l /proc/self/ns/mnt
lrwxrwxrwx 1 root root 0 Jul 22 20:24 /proc/self/ns/mnt -> 'mnt:[4026532261]'
```

2. Via read link command:

```console
root@my_pc:/home/user# readlink /proc/self/ns/mnt
mnt:[4026532261]
```

## Create a new mount that is accessible only to the child shell
Now we will mount a new tmpfs filesystem (A filesystem that exists only on the RAM):

```console
root@my_pc:/home/user# mkdir child_mount
root@my_pc:/home/user# mount -t tmpfs tmpfs child_mount
```

We'll create a new file inside of it that should be accessible only to the child process:

```console
root@my_pc:/home/user# cd child_mount
root@my_pc:/home/user/child_mount# echo "hello" > child.txt
root@my_pc:/home/user/child_mount# ls
child.txt
root@my_pc:/home/user/child_mount# cat child.txt
hello
```

Now, let's try to access it from another shell:
```console
user@my_pc:~/child_mount$ ls -la
total 8
drwxr-xr-x  2 root     root     4096 Jul 22 20:27 .
drwxr-x--- 25 user     user     4096 Jul 22 20:27 ..
```

## Check the currnet mounted filesystem

The reason behind it is that the each process has a different mount namespace and as so a different mount list, let's check the current mount with `df` command.

1. From the newly spawned shell:
```console
root@my_pc:/home/user/child_mount# df .
Filesystem     1K-blocks  Used Available Use% Mounted on
tmpfs            8119312     4   8119308   1% /home/user/child_mount
```

2. From the parent shell
```console
user@my_pc:/home/user/child_mount# df .
Filesystem      1K-blocks      Used Available Use% Mounted on
/dev/sdc       1055762868 124879304 877180092  13% /
```

## Enter a namespace of a different process

To conclude the demo, let's enter the child shell namespace from the parent shell. We can do this by using `nsenter` command with `-m` flag which stands for mount namespace and `--target` flag which require the PID of the process we want to enter its namespace.

To check the PID of the child shell:
```console
root@my_pc:/home/user/child_mount# echo $$
559
```

Finally, to enter the shell and check if the file exists:
```console
user@my_pc:~/child_mount$ sudo nsenter -m --target 559
root@my_pc:/# cd /home/user/child_mount/
root@my_pc:/home/user/child_mount# ls -la
total 8
drwxrwxrwt  2 root     root       60 Jul 22 20:29 .
drwxr-x--- 25 user     user     4096 Jul 22 20:27 ..
-rw-r--r--  1 root     root        6 Jul 22 20:29 child.txt
root@my_pc:/home/user/child_mount# cat child.txt
hello
```





