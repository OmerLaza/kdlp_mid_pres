# The `mount namespace`
## How does `namespace`s work in general
Each `task_struct` has:
1. Pointer to `fs_struct`- struct that contains file system information related to the current process 
1. Pointer to `nsproxy`- struct that contains info about (most of) it's namespaces


[include/linux/sched.h](https://elixir.bootlin.com/linux/v6.5.13/source/include/linux/sched.h#L738)
```c
struct task_struct {
    ...
    /* Filesystem information: */
	struct fs_struct		*fs;
    ...
    /* Namespaces: */
	struct nsproxy			*nsproxy;
    ...
};
```


The `nsproxy` struct contain pointers to all per-process (excluding the `user namespace`), we care about the `struct mnt_namespace *mnt_ns` which represents the mount namespace.

[include/linux/nsproxy.h](https://elixir.bootlin.com/linux/v6.5.13/source/include/linux/nsproxy.h#L31)
```c
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns_for_children;
	struct net 	     *net_ns;
	struct time_namespace *time_ns;
	struct time_namespace *time_ns_for_children;
	struct cgroup_namespace *cgroup_ns;
};
```
The `nsproxy` is shared by tasks which share all namespaces. As soon as a single namespace is cloned or unshared, the `nsproxy` is copied.

This is the `struct mnt_namespace`

[fs/mount.h](https://elixir.bootlin.com/linux/v6.5.13/source/fs/mount.h#L8)
```c
struct mnt_namespace {
	struct ns_common	ns;
	struct mount *	root;
	struct rb_root		mounts; /* Protected by namespace_sem */
	struct user_namespace	*user_ns;
	struct ucounts		*ucounts;
	u64			seq;	/* Sequence number to prevent loops */
	wait_queue_head_t poll;
	u64 event;
	unsigned int		nr_mounts; /* # of mounts in the namespace */
	unsigned int		pending_mounts;
} __randomize_layout;
```

We won't look much into this struct itself.

## How do the `unshare` syscall interact with the mount namespace

Let's take a look at the `unshare` syscall.


[kernel/fork.c](https://elixir.bootlin.com/linux/v6.5.13/source/kernel/fork.c#L3515)
```c
SYSCALL_DEFINE1(unshare, unsigned long, unshare_flags)
{
	return ksys_unshare(unshare_flags);
}
```

As the [`man unshare`](https://man7.org/linux/man-pages/man2/unshare.2.html) states, we can use the `CLONE_NEWNS` to:
> Unshare the mount namespace, so that the calling
              process has a private copy of its namespace which is not
              shared with any other process
     
[kernel/fork.c](https://elixir.bootlin.com/linux/v6.5.13/source/kernel/fork.c#L3396)
```c
int ksys_unshare(unsigned long unshare_flags)
{
	struct fs_struct *fs, *new_fs = NULL;
...
	struct nsproxy *new_nsproxy = NULL;
...
	int err;
...
	/*
	 * If unsharing namespace, must also unshare filesystem information.
	 */
	if (unshare_flags & CLONE_NEWNS)
		unshare_flags |= CLONE_FS;
...
	err = unshare_nsproxy_namespaces(unshare_flags, &new_nsproxy,
					 new_cred, new_fs);
...
	if (new_fs || new_fd || do_sysvsem || new_cred || new_nsproxy) {
...
		if (new_nsproxy)
			switch_task_namespaces(current, new_nsproxy);
...
	return err;
}
```
We will look more into `unshare_nsproxy_namespaces`, where we `unshare` all the `namespaces` spesiffied by the flags and create new `nsproxy` if any of the `namespaces` are changed.

[kernel/nsproxy.c](https://elixir.bootlin.com/linux/v6.5.13/source/kernel/nsproxy.c#L213)
```c
/*
 * Called from unshare. Unshare all the namespaces part of nsproxy.
 * On success, returns the new nsproxy.
 */
int unshare_nsproxy_namespaces(unsigned long unshare_flags,
	struct nsproxy **new_nsp, struct cred *new_cred, struct fs_struct *new_fs)
{
	struct user_namespace *user_ns;
...
	if (!(unshare_flags & (CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC |
			       CLONE_NEWNET | CLONE_NEWPID | CLONE_NEWCGROUP |
			       CLONE_NEWTIME)))
		return 0;
...
	*new_nsp = create_new_namespaces(unshare_flags, current, user_ns,
					 new_fs ? new_fs : current->fs);
...
}
```

Next we will take a look at `create_new_namespaces` where the new `nsproxy` is created

[kernel/nsproxy.c](https://elixir.bootlin.com/linux/v6.5.13/source/kernel/nsproxy.c#L67)

```c
/*
 * Create new nsproxy and all of its the associated namespaces.
 * Return the newly created nsproxy.  Do not attach this to the task,
 * leave it to the caller to do proper locking and attach it to task.
 */
static struct nsproxy *create_new_namespaces(unsigned long flags,
	struct task_struct *tsk, struct user_namespace *user_ns,
	struct fs_struct *new_fs)
{
	struct nsproxy *new_nsp;
	int err;

	new_nsp = create_nsproxy();
...
	new_nsp->mnt_ns = copy_mnt_ns(flags, tsk->nsproxy->mnt_ns, user_ns, new_fs);
...
}
```

Now we get to `copy_mnt_ns` the function that actually does the coping itself from the parent `mount namespace` to the child new `mount namespace` (each mount based on it's _propagation type_ [more info here](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)).
The new process will have it's own `mount namespace` and from now on the parent and child `mount namespace`s are separate (although some mounts can propagate into a `mount namespace`)

[fs/namespace.c](https://elixir.bootlin.com/linux/v6.5.13/source/fs/namespace.c#L3744)
```c
__latent_entropy
struct mnt_namespace *copy_mnt_ns(unsigned long flags, struct mnt_namespace *ns,
		struct user_namespace *user_ns, struct fs_struct *new_fs)
{
	struct mnt_namespace *new_ns;
	struct vfsmount *rootmnt = NULL, *pwdmnt = NULL;
	struct mount *p, *q;
	struct mount *old;
	struct mount *new;
	int copy_flags;

	BUG_ON(!ns);

	if (likely(!(flags & CLONE_NEWNS))) {
		get_mnt_ns(ns);
		return ns;
	}

	old = ns->root;

	new_ns = alloc_mnt_ns(user_ns, false);
	if (IS_ERR(new_ns))
		return new_ns;

	namespace_lock();
	/* First pass: copy the tree topology */
	copy_flags = CL_COPY_UNBINDABLE | CL_EXPIRE;
	if (user_ns != ns->user_ns)
		copy_flags |= CL_SHARED_TO_SLAVE;
	new = copy_tree(old, old->mnt.mnt_root, copy_flags);
	if (IS_ERR(new)) {
		namespace_unlock();
		free_mnt_ns(new_ns);
		return ERR_CAST(new);
	}
	if (user_ns != ns->user_ns) {
		lock_mount_hash();
		lock_mnt_tree(new);
		unlock_mount_hash();
	}
	new_ns->root = new;
	list_add_tail(&new_ns->list, &new->mnt_list);

	/*
	 * Second pass: switch the tsk->fs->* elements and mark new vfsmounts
	 * as belonging to new namespace.  We have already acquired a private
	 * fs_struct, so tsk->fs->lock is not needed.
	 */
	p = old;
	q = new;
	while (p) {
		q->mnt_ns = new_ns;
		new_ns->mounts++;
		if (new_fs) {
			if (&p->mnt == new_fs->root.mnt) {
				new_fs->root.mnt = mntget(&q->mnt);
				rootmnt = &p->mnt;
			}
			if (&p->mnt == new_fs->pwd.mnt) {
				new_fs->pwd.mnt = mntget(&q->mnt);
				pwdmnt = &p->mnt;
			}
		}
		p = next_mnt(p, old);
		q = next_mnt(q, new);
		if (!q)
			break;
		// an mntns binding we'd skipped?
		while (p->mnt.mnt_root != q->mnt.mnt_root)
			p = next_mnt(skip_mnt_tree(p), old);
	}
	namespace_unlock();

	if (rootmnt)
		mntput(rootmnt);
	if (pwdmnt)
		mntput(pwdmnt);

	return new_ns;
}
```

## Sources

1. [Kernel v6.5.13 source code from bootlin](https://elixir.bootlin.com/linux/v6.5.13/)
1. [`man 2 unshare`](https://man7.org/linux/man-pages/man2/unshare.2.html)
1. [`man 7 mount_namespaces`](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)