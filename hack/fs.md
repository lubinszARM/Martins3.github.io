# linux 文件系统

<!-- vim-markdown-toc GitLab -->

- [打通fs的方法](#打通fs的方法)
    - [有待继续的内容](#有待继续的内容)
- [CFS : Concrete FS](#cfs-concrete-fs)
- [VFS](#vfs)
    - [lookup](#lookup)
    - [path](#path)
- [VFS standard file operation](#vfs-standard-file-operation)
    - [inode_operations::fiemap](#inode_operationsfiemap)
    - [file_operations::mmap](#file_operationsmmap)
- [file io](#file-io)
    - [aio(dated)](#aiodated)
    - [io uring](#io-uring)
- [file writeback](#file-writeback)
- [event poll](#event-poll)
- [flock](#flock)
    - [fcntl](#fcntl)
    - [ioctl](#ioctl)
- [ext2](#ext2)
- [nobdv fs](#nobdv-fs)
- [virtual fs library summary](#virtual-fs-library-summary)
    - [libfs](#libfs)
- [virtual fs for debug](#virtual-fs-for-debug)
- [file descriptor](#file-descriptor)
- [virtual fs interface](#virtual-fs-interface)
- [mount](#mount)
    - [pivot_root](#pivot_root)
- [superblock](#superblock)
    - [super operations](#super-operations)
- [dentry](#dentry)
    - [path](#path-1)
- [helper](#helper)
- [attr](#attr)
- [open.c](#openc)
- [fuse](#fuse)
- [block layer](#block-layer)
- [initfs](#initfs)
- [overlay fs](#overlay-fs)
- [inode](#inode)
- [dcache](#dcache)
- [lvm](#lvm)
- [splice and pipe](#splice-and-pipe)
- [IO model](#io-model)
- [devpts](#devpts)
- [dup(merge)](#dupmerge)
- [IO buffer(merge)](#io-buffermerge)
- [notify](#notify)
- [attr](#attr-1)
- [direct IO](#direct-io)
- [block device](#block-device)
- [char device](#char-device)
- [tmpfs](#tmpfs)
- [exciting](#exciting)
- [benchmark](#benchmark)
    - [fio](#fio)
    - [filebench](#filebench)
- [nvme](#nvme)
- [fallocate](#fallocate)
- [union fs](#union-fs)

<!-- vim-markdown-toc -->



用户层的角度:
1. fs 可以用于访问文件: ext2, ext3
2. fs 用于导出信息: (audit 为什么使用网络)
    1. 系统的状态 : proc 
    2. 设备状态 : sysfs
    3. 调试 : debugfs
3. 设备可以通过文件系统进行访问 : fs/char_dev.c fs/block_dev.c

内核的角度：
1. 路径 hardlink softlink 创建文件 等

还是认为，只要将 linux kernel lab 中间搞定了，那么就是掌握了核心内容，其余的问题在逐个分析就不难了:

1. 似乎从来遇到过的 dentry 的 operation 的操作是做什么的 ? 还是不知道，甚至不知道 ext2 是否注册过

感觉需要快速浏览一下书才可以心中有个大致的概念。


## 打通fs的方法
1. journal 的实现了解一下
2. lfs 的实现 (了解)
3. 文件系统的启动过程
  1. 想一下，/dev/ mount 时间是在 initramfs，也就是在根文件系统被 mount 之后，但是根文件系统的 mount 似乎需要 /dev/something
  2. mount 的接口是什么 ? 可以 mount 的是 路径 fd inode 设备还是 ?
  3. 根文件系统的 mount 位置是靠什么指定的 ? 内核参数 root=/ 在 initramfs 中间，这是什么探测的
  4. mount 参数 ： 设备，路径。根文件系统的设备，内核参数或者 initramfs，其他的各种设备，探测，然后放到 /dev/sda 之内的位置
    1. 所以，/dev/sda 被探测是确定的
4. 文件系统启动除了 mount 还有什么 ? inode cache, dcache 以及其他的数据结构需要初始化吗 ?
5. uring io
6. io scheduler
7. 磁盘驱动，不要基于内存的虚假驱动

8. fuse
9. nfs

#### 有待继续的内容
1. file lock
2. 内部的 lock 的设计


## CFS : Concrete FS
mkfs.ext4 居然存在那么多的选项，才知道文件系统也是具有各种选项的



## VFS
基本元素:

superblock : 持有整个文件系统的基本信息。 superblock operation 描述创建 fs specific 的 inode 

inode : 描述文件的静态属性，比如大小，创建的时间等等。

file : 描述文件的动态属性，也就是打开一个文件之后，读写指针的位置在何处

dir entry : 目录也是一个文件，这是目录保存的基本内容。(那么目录文件中间会保存 . 和 .. 吗 ? 应该不会吧)

VFS 提供的功能:
1. 提供一些统一的解决方案 : file lock，路径解析
2. 提供一些 generic 的实现 : 
3. 一些需要具体文件系统实现的接口 :

file_operations::mkdir 作为 file 的inode 和 dir 有区别吗 ?

| x    | inode_operations                                | file_operations     |
|------|-------------------------------------------------|---------------------|
| dir  | inode operation 应该是主要提供给 dir 的操作的， | 主要是 readdir 操作 |
| file | attr acl 相关                                   | 各种IO              |

所以，不相关的功能被放到同一个 operation 结构体中间了。


#### lookup

1. 流程
2. 如何处理锁机制
3. namei.c 的作用是什么: 各种文件名的处理吧，尤其处理 hardlink, symbol link 之类的操作
3. 基于 pwd 的是怎么回事


4. follow_dotdot



* ***核心数据结构***
1. 都是在什么时候装配的 ?
2. 

```c
struct nameidata {
	struct path	path;
	struct qstr	last;
	struct path	root;
	struct inode	*inode; /* path.dentry.d_inode */
	unsigned int	flags;
	unsigned	seq, m_seq, r_seq;
	int		last_type;
	unsigned	depth;
	int		total_link_count;
	struct saved {
		struct path link;
		struct delayed_call done;
		const char *name;
		unsigned seq;
	} *stack, internal[EMBEDDED_LEVELS];
	struct filename	*name;
	struct nameidata *saved;
	unsigned	root_seq;
	int		dfd;
	kuid_t		dir_uid;
	umode_t		dir_mode;
} __randomize_layout;
```





* ***流程***
walk_component 

```c
static const char *walk_component(struct nameidata *nd, int flags) // 调用 lookup_fast 和 lookup_slow
```

lookup_fast : 调用 `__d_lookup`
`__lookup_slow` : 
1. d_alloc_parallel : ?
2. `old = inode->i_op->lookup(inode, dentry, flags);` : vfs 提供 looup 接口

使用ext2作为例子:
```c
static struct dentry *ext2_lookup(struct inode * dir, struct dentry *dentry, unsigned int flags)
{
	struct inode * inode;
	ino_t ino;
	
	if (dentry->d_name.len > EXT2_NAME_LEN)
		return ERR_PTR(-ENAMETOOLONG);

	ino = ext2_inode_by_name(dir, &dentry->d_name);
	inode = NULL;
	if (ino) {
		inode = ext2_iget(dir->i_sb, ino);
		if (inode == ERR_PTR(-ESTALE)) {
			ext2_error(dir->i_sb, __func__,
					"deleted inode referenced: %lu",
					(unsigned long) ino);
			return ERR_PTR(-EIO);
		}
	}
	return d_splice_alias(inode, dentry);
}
```

1. 第一个:
```c
ino_t ext2_inode_by_name(struct inode *dir, const struct qstr *child)
{
	ino_t res = 0;
	struct ext2_dir_entry_2 *de;
	struct page *page;
	
	de = ext2_find_entry (dir, child, &page); // 将 direcory 的内容加载到 page cache 中间，其实 dentry 和 这个没有什么关系，此处加载的 page 其实就是当做普通文件内容即可
	if (de) {
		res = le32_to_cpu(de->inode);
		ext2_put_page(page);
	}
	return res;
}
```

2. 第二个: d_splice_alias
  1. 是不是 inode 和 dentry 不是一一对应的，是由于硬链接的原因吗 ? TODO
  2. 如果不考虑 alias 问题，等价于 `__d_add`


#### path
1. kern_path 的实现原理，感觉反反复复一直看到
2. path 的定义，其实还是不懂，为什么路径需要 mount 和 dentry 两个项

```c
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
} __randomize_layout;
```



```c
int kern_path(const char *name, unsigned int flags, struct path *path)
{
	return filename_lookup(AT_FDCWD, getname_kernel(name),
			       flags, path, NULL);
}

filename_lookup : 装配 nameidata，call path_lookupat
path_lookupat : 就是查询的核心位置了
```


## VFS standard file operation 
1. 
```c
/*
 * Support for read() - Find the page attached to f_mapping and copy out the
 * data. Its *very* similar to do_generic_mapping_read(), we can't use that
 * since it has PAGE_SIZE assumptions.
 */
static ssize_t hugetlbfs_read_iter(struct kiocb *iocb, struct iov_iter *to)
```

2. 很窒息，为什么 disk fs 和 nobev 的fs 的 address_space_operations 的内容是相同的。
    1. simple_readpage : 将对应的 page 的内容清空
    2. simple_write_begin : 将需要加载的页面排列好
    3. 所以说不过去
```c
static const struct address_space_operations myfs_aops = {
	/* TODO 6: Fill address space operations structure. */
	.readpage	= simple_readpage,
	.write_begin	= simple_write_begin,
	.write_end	= simple_write_end,
};

static const struct address_space_operations minfs_aops = {
	.readpage = simple_readpage,
	.write_begin = simple_write_begin,
	.write_end = simple_write_end,
};
```


#### inode_operations::fiemap
// TODO 
// 啥功能呀 ?

```c
const struct inode_operations ext2_file_inode_operations = {
#ifdef CONFIG_EXT2_FS_XATTR
	.listxattr	= ext2_listxattr,
#endif
	.getattr	= ext2_getattr,
	.setattr	= ext2_setattr,
	.get_acl	= ext2_get_acl,
	.set_acl	= ext2_set_acl,
	.fiemap		= ext2_fiemap,
};

/**
 * generic_block_fiemap - FIEMAP for block based inodes
 * @inode: The inode to map
 * @fieinfo: The mapping information
 * @start: The initial block to map
 * @len: The length of the extect to attempt to map
 * @get_block: The block mapping function for the fs
 *
 * Calls __generic_block_fiemap to map the inode, after taking
 * the inode's mutex lock.
 */

int generic_block_fiemap(struct inode *inode,
			 struct fiemap_extent_info *fieinfo, u64 start,
			 u64 len, get_block_t *get_block)
{
	int ret;
	inode_lock(inode);
	ret = __generic_block_fiemap(inode, fieinfo, start, len, get_block);
	inode_unlock(inode);
	return ret;
}
```

#### file_operations::mmap

唯一的调用位置: mmap_region 
```c
static inline int call_mmap(struct file *file, struct vm_area_struct *vma) {
	return file->f_op->mmap(file, vma);
}

/* This is used for a general mmap of a disk file */
int generic_file_mmap(struct file * file, struct vm_area_struct * vma)
{
	struct address_space *mapping = file->f_mapping;

	if (!mapping->a_ops->readpage)
		return -ENOEXEC;
	file_accessed(file);
	vma->vm_ops = &generic_file_vm_ops; // TODO 追踪一下
	return 0;
}
```

## file io
epoll
poll
select

aio 和 uring :

> file_operations : 每一个项目的内容都应该清楚吧 !

特别关注的内容 : 才发现几乎没有一个看的懂
2. owner 的作用是什么，什么时候注册的 ?
3. iopoll 的 epoll 有没有关系 ?
4. iterate 和 iterate_shared 是用于遍历什么的 ?
5. fsycn 和 fasync 的区别
6. flock 
7. splice_write 和 splice_read 和 pipe 的关系是什么 ?
8. setfl 是做什么的 ?

```c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iopoll)(struct kiocb *kiocb, bool spin);
	int (*iterate) (struct file *, struct dir_context *);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	unsigned long mmap_supported_flags;
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*setfl)(struct file *, unsigned long);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
	ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
			loff_t, size_t, unsigned int);
	loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
				   struct file *file_out, loff_t pos_out,
				   loff_t len, unsigned int remap_flags);
	int (*fadvise)(struct file *, loff_t, loff_t, int);
} __randomize_layout;
```

2. 

#### aio(dated)
1. 但是似乎是可以替代 read 和 write 的功能，比如 ext2 中间就没有为 ext2_file_operation 注册 read 和 write 
2. read_iter 和 write_iter 是用来实现 异步IO 的。 为什么可以实现 aio ? aio 在用户层的体现是什么 ? 

aio 在工程上的具体使用 libaio 的库， https://oxnz.github.io/2016/10/13/linux-aio/


第一个问题: 如何实现 write(2)
在 fs/read_write.c 中间描述
```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count) { return ksys_write(fd, buf, count); }
```
=> ksys_write => vfs_write

vfs_write
1. file_start_write 和 file_end_write 处理锁相关的(应该吧)
2. `__vfs_write` : 首先尝试使用file_operation::write，然后尝试使用 new_sync_write
3. new_sync_write : 首先初始化 aio 相关的结构体，然后调用 write_iter

```c
static ssize_t new_sync_write(struct file *filp, const char __user *buf, size_t len, loff_t *ppos)
{
	struct iovec iov = { .iov_base = (void __user *)buf, .iov_len = len };
	struct kiocb kiocb;
	struct iov_iter iter;
	ssize_t ret;

	init_sync_kiocb(&kiocb, filp);
	kiocb.ki_pos = (ppos ? *ppos : 0);
	iov_iter_init(&iter, WRITE, &iov, 1, len);

	ret = call_write_iter(filp, &kiocb, &iter);
	BUG_ON(ret == -EIOCBQUEUED);
	if (ret > 0 && ppos)
		*ppos = kiocb.ki_pos;
	return ret;
}
```

第二个问题: aio 如何使用 [^2]
The Asynchronous Input/Output (AIO) interface allows many I/O requests to be submitted in parallel without the overhead of a thread per request.
The purpose of this document is to explain how to use the Linux AIO interface,
namely the function family `io_setup`, `io_submit`, `io_getevents`, `io_destroy`. Currently, the AIO interface is best for O_DIRECT access to a raw block device like a disk, flash drive or storage array.
> 这些系统调用都是在 aio.c 中间

```c
int io_setup(int maxevents, io_context_t *ctxp);
int io_destroy(io_context_t ctx);
```
io_context_t 是指向 io_context 的指针类型，其中 maxevents 是参数，ctxp 是返回值。

io_context 是公共资源:
An `io_context_t` object can be shared between threads, both for submission and completion. 
No guarantees are provided about ordering of submission and completion with respect to interaction from multiple threads. 
There may be performance implications from sharing `io_context_t` objects between threads.

```c
struct kiocb {
	struct file		*ki_filp;

	loff_t			ki_pos;
	void (*ki_complete)(struct kiocb *iocb, long ret, long ret2);
	void			*private;
	int			ki_flags;
	u16			ki_hint;
	u16			ki_ioprio; /* See linux/ioprio.h */
	unsigned int		ki_cookie; /* for ->iopoll */
};

struct iocb {
	PADDEDptr(void *data, __pad1);	/* Return in the io completion event */
	/* key: For use in identifying io requests */
	/* aio_rw_flags: RWF_* flags (such as RWF_NOWAIT) */
	PADDED(unsigned key, aio_rw_flags);

	short		aio_lio_opcode;	
	short		aio_reqprio;
	int		aio_fildes;

	union {
		struct io_iocb_common		c;
		struct io_iocb_vector		v;
		struct io_iocb_poll		poll;
		struct io_iocb_sockaddr	saddr;
	} u;
};

struct io_iocb_common {
	PADDEDptr(void	*buf, __pad1);
	PADDEDul(nbytes, __pad2);
	long long	offset;
	long long	__pad3;
	unsigned	flags;
	unsigned	resfd;
};	/* result code is the amount read or -'ve errno */
```
The meaning of the fields is as follows: data is a pointer to a user-defined object used to represent the operation

- `aio_lio_opcode` is a flag indicate whether the operation is a read (IO_CMD_PREAD) or a write (IO_CMD_PWRITE) or one of the other supported operations
- `aio_fildes` is the `fd` of the file that the iocb reads or writes
- `buf` is the pointer to memory that is read or written
- `nbytes` is the length of the request
- `offset` is the initial offset of the read or write within the file

> 为什么 kiocb 中间没有 nbytes 之类内容

`io_prep_pread` and `io_prep_pwrite` 初始化 iocb 之后，然后使用 io_submit 提交，使用 `io_getevents` 获取。

作者还分析了 io_getevents 的参数不同取值的含义。

第三个问题: aio 内核如何实现的 
1. 分析 syscall io_submit ，可以非常容易的跟踪到 aio_write，并且在其中调用 call_write_iter
2. 如何实现系统调用 : io_getevents

io_getevents => read_events => aio_read_events => aio_read_events_ring 

aio_read_events_ring 会访问 kioctx::ring_pages 来提供给用户就绪的 io，使用 aio_complete 向其中添加。
aio_complete 会调用 eventfd_signal，这是实现 epoll 机制的核心。

> 1. io_submit 提交请求之后，然后返回到用户空间，用于完成IO的内核线程是如何管理的 ?　暂时没有找到，io_submit 开始返回的位置。
> 2. aio 不能使用 page cache, 那么对于 metadata 的读取， aio 可以实现异步吗 ? (我猜测，应该所有的文件系统都是不支持的吧!)

#### io uring
怎么会有 8000 行啊
大概了解一下吧，暂时可以看其他更加重要的东西:

io_uring 有如此出众的性能，主要来源于以下几个方面[^8]：
1. 用户态和内核态共享提交队列（submission queue）和完成队列（completion queue）
2. IO 提交和收割可以 offload 给 Kernel，且提交和完成不需要经过系统调用（system call）
3. 支持 Block 层的 Polling 模式
4. 通过提前注册用户态内存地址，减少地址映射的开销
5. 不仅如此，io_uring 还可以完美支持 buffered IO，而 libaio 对于 buffered IO 的支持则一直是被诟病的地方。

https://github.com/axboe/liburing : 其中的 test 包含很多demo

[^8] 是 smartx 的人写的，后面还讲解了 io uring 的接口的使用，值得一看 TODO

read write 会直到事情做完
epoll : 当有的工作做好之后，通知，其实是一个 thread 可以监控一大群的 socket，任何一个 ready 之后就开始工作

https://thenewstack.io/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/ : 这个也可以参考参考
https://kernel-recipes.org/en/2019/talks/faster-io-through-io_uring/

read the doc : [^10]
aio's limitation :
1. only support async IO for O_DIRECT
> TODO how O_DIRECT works
> 1. checked 
> 2. skip page cache
2. some constrans for sync submit
3. api lead noticeable copying

TODO please continue the documentation


## file writeback
fs/file-writeback.c 中间到底完成什么工作
// 具体内容有点迷惑，但是file-writeback.c 绝对不是 page-writeback.c 更加底层的东西
// 其利用 flusher thread ，然后调用 do_writepages 实现将整个文件，甚至整个文件系统写回，


## event poll
// TODO
// 找到简书上内容上，写 enomia 的时候介绍 epoll 机制的内容

## flock
1. advisory lock 指的是 ?
2. open file lock ?
2. mandatory lock ?
3. lost lock
4. recoid lock
3. flock 和 fcntl 获取的 lock 不兼容 ?
4. [filelock](https://man7.org/linux/man-pages/man3/flockfile.3.html)


[^9] 
Traditionally, locks are **advisory** in Unix. They work only when a process explicitly acquires and releases locks, and are ignored if a process is not aware of locks.

**There are several types of advisory locks available in Linux**:
1. BSD locks (flock)
2. POSIX record locks (fcntl, lockf)
3. Open file description locks (fcntl)

The following features are common for locks of all types:
1. All locks support blocking and non-blocking operations.
2. Locks are allowed only on files, but not directories.
3. Locks are automatically removed when the process exits or terminates. It’s guaranteed that if a lock is acquired, the process acquiring the lock is still alive.

TODO 该文档写的很好，实际上，这个提供给用户的 lock 和 VFS 实现的 lock 根本就是两回事。
主要内容在 : fs/locks.c


* ***如何使用***

man flock(2)

If a process uses open(2) (or similar) to obtain more than one file descriptor for the same file,
these file descriptors are treated independently by flock(). 
An attempt to lock the file using one of these file descriptors may be denied by a lock that the calling process has already placed via another file descriptor.
> 1. 一共存在两种锁


* ***内核文档***

https://www.kernel.org/doc/html/latest/filesystems/locking.html : 说明几乎VFS api 调用的时候需要持有的锁

1. mandatory locking 是怎么回事啊

* ***Understanding Linux Kernel***

The POSIX standard requires a file-locking mechanism based on the *fcntl()* system
call. It is possible to lock an arbitrary region of a file (even a single byte) or to lock
the whole file (including data appended in the future). Because a process can choose
to lock only a part of a file, it can also hold multiple locks on different parts of the
file.
> 1. 使用系统调用 fcntl
> 2. 可以按照区域锁
> 3. 一个文件可以设置多个锁

TODO fcntl 的内容很多，首先打住一下

#### fcntl
1. lease ?
2. dnotify 
3. seal

来源自 man fcntl(2)

主要的功能:
1. Duplicating a file descriptor
2. File descriptor flags

The principal difference between the two lock types is that whereas traditional record locks are associated with a process,
open file description locks are associated with the open file description on which they are acquired, much like locks acquired with flock(2).
Consequently (and unlike traditional advisory record locks), open file description locks are inherited across fork(2) (and clone(2) with CLONE_FILES),
and are only automatically released on the last close of the open file description, instead of being released on any close of the file.

> Warning: the Linux implementation of mandatory locking is unreliable.  See BUGS below.  Because of these bugs, and the fact that the feature is believed to be little used, since Linux 4.5, mandatory locking has been made an optional feature, governed by a configuration option (CONFIG_MANDATORY_FILE_LOCKING).  This is an initial step toward removing this feature completely.

#### ioctl
> TODO 另一个大而全的系统调用 ?

## ext2
内部结构:
https://www.nongnu.org/ext2-doc/ext2.html#s-creator-os

## nobdv fs
tmp proc sysfs

ramfs 和 ext2 fs 应该可以作为两个典型

> 怀疑这个分类有点问题，正确的分类应该参考几个mount 函数


sysfs : device model 的形象显示，更加重要的是，让驱动有一个和用户沟通的快捷方式。
使用 ioctl 似乎不能解决问题：不是所有的驱动都是有 /dev/ 下存在设备的啊 !
这种根本没有在文件系统中间出现的东西，应该不可以使用 ioctl 吧 !

## virtual fs library summary
kernfs seq libfs

kernfs 也是利用 seq 中间的内容，那么 libfs 和 kernfs 的区别在于什么地方呀 ?

#### libfs
通过 ramfs 理解 libfs 吧 !


## virtual fs for debug
1. kprobes
2. debugfs
3. tracefs

wowotech 有一个嵌入式工程师都应该知道的 debug

## file descriptor
为什么需要 file descriptor，而不是直接返回 inode ?
首先，为什么需要 file struct，因为 file struct 描述了读写的过程，
多以，可以存在多个 file struct 对应同一个 inode。
那么，当一个进程需要打开一个文件，最简单显然是返回一个数值，让用户来找到该文件。


在同一个进程中间不同 fd 指向相同的 file struct : 为什么存在这种需求 ?
This situation may arise as a result of a call to dup(), dup2(), or fcntl() (see Section 5.5).

## virtual fs interface
1. fd 
5. dentry :
    1. 实在是想不懂为什么会出现 dops 这种东西
```c
// 其中的想法是什么 : 
struct dentry_operations {
        int (*d_revalidate)(struct dentry *, unsigned int);
        int (*d_weak_revalidate)(struct dentry *, unsigned int);
        int (*d_hash)(const struct dentry *, struct qstr *);
        int (*d_compare)(const struct dentry *,
                         unsigned int, const char *, const struct qstr *);
        int (*d_delete)(const struct dentry *);
        int (*d_init)(struct dentry *);
        void (*d_release)(struct dentry *);
        void (*d_iput)(struct dentry *, struct inode *);
        char *(*d_dname)(struct dentry *, char *, int);
        struct vfsmount *(*d_automount)(struct path *);
        int (*d_manage)(const struct path *, bool);
        struct dentry *(*d_real)(struct dentry *, const struct inode *);
};
```
1. 存在 system wide 的 inode table 吗 ?
    2. 如果不同的文件系统 inode 被放在一起吗 ?
3. 什么时候不同的 file struct 指向同一个 inode 上 ?
    1. This occurs because each process independently called open() for the same file. A similar situation could occur if a single process opened the same file twice.
    2. 是不是只要打开一次文件就会出现一次 file struct ?
4. 所以标准输入，标准输出的 fd 为 0 1 2 设置于支持都是如何形成的 ?


[^1] 提供了所有的API的解释说明，主要是几个 operation，需要解释一下:

```c
int generic_file_mmap(struct file * file, struct vm_area_struct * vma)
{
	struct address_space *mapping = file->f_mapping;

	if (!mapping->a_ops->readpage)
		return -ENOEXEC;
	file_accessed(file);
	vma->vm_ops = &generic_file_vm_ops;
	return 0;
}

struct vm_operations_struct generic_file_vm_ops = {
	.fault		= filemap_fault, // 标准 page fault 函数
	.map_pages	= filemap_map_pages, // TODO 从 do_fault_around 调用 filemap_map_pages https://lwn.net/Articles/588802/ 但是具体实现有点不懂，什么叫做 easy access page
	.page_mkwrite	= filemap_page_mkwrite, // 用于通知当 page 从 read 变成可以 write 的过程
};
```

ext2 的内容:
```
#define ext2_file_mmap	generic_file_mmap
```

ext4 的内容:
```c
static const struct vm_operations_struct ext4_file_vm_ops = {
	.fault		= ext4_filemap_fault,
	.map_pages	= filemap_map_pages,
	.page_mkwrite   = ext4_page_mkwrite,
};

static int ext4_file_mmap(struct file *file, struct vm_area_struct *vma)
{
	struct inode *inode = file->f_mapping->host;
	struct ext4_sb_info *sbi = EXT4_SB(inode->i_sb);
	struct dax_device *dax_dev = sbi->s_daxdev;

	if (unlikely(ext4_forced_shutdown(sbi)))
		return -EIO;

	/*
	 * We don't support synchronous mappings for non-DAX files and
	 * for DAX files if underneath dax_device is not synchronous.
	 */
	if (!daxdev_mapping_supported(vma, dax_dev))
		return -EOPNOTSUPP;

	file_accessed(file);
	if (IS_DAX(file_inode(file))) {
		vma->vm_ops = &ext4_dax_vm_ops;
		vma->vm_flags |= VM_HUGEPAGE;
	} else {
		vma->vm_ops = &ext4_file_vm_ops;
	}
	return 0;
}

vm_fault_t ext4_filemap_fault(struct vm_fault *vmf)
{
	struct inode *inode = file_inode(vmf->vma->vm_file);
	vm_fault_t ret;

	down_read(&EXT4_I(inode)->i_mmap_sem);
	ret = filemap_fault(vmf);
	up_read(&EXT4_I(inode)->i_mmap_sem);

	return ret;
}
```
由此看来，file_operations::mmap 的作用:
1. file_accessed
2. 注册 vm_ops

## mount
问题:
1. mount 在路径查询到时候的作用是什么 ?
2. master 和 slave 在这里是个什么概念
3. kern_mount 的时候 : 那些根本没有路径的，应该不调用 graft_tree 吧
4. 根文件系统的 mount 是怎么回事

6. 忽然想到，在本来的文件系统上，一个文件夹，比如 /home/shen/core 下面本来就是存在文件的，
在这种情况下，该文件系统被 mount ， 描述一下这种情况


TODO
1. https://zhuanlan.zhihu.com/p/93592262 新版的 mount 看这里就可以了吧，现在大致理解了 mount 的作用，先看下一个吧!


| function    | explaination                                                                                                                                                                                             |
|-------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| lock_mount  | 后挂载的文件系统会在挂载到前一次挂载的文件系统的根dentry上。lock_mount这个函数的一部分逻辑就保证了在多文件系统挂载同路径的时候，让每个新挂载的文件系统都顺序的挂载（覆盖）上一次挂载实例的根dentry。[^5] |
| lokup_mount | 通过 parent fs 的 vfsmount 和 dentry 找到在该位置上的 mount 实例，lock_mount 需要调用从而找到多次在同一个位置 mount 的最终 mount 点                                                                      |



两种特殊的mount 情况:
1. 同一个设备可以 mount 到不同的位置
    1. 可以理解为多个进入到该文件系统可以多个入口
2. 多个设备可以 mount 到同一个目录中间
    1. 后面的隐藏前面的
    2. 多次 mount 需要多次 unmount 对应


loop device: The loop device driver transforms operations on the associated block device into file(system)operations, that's how the data/partitions end up in a file. [^3]
> 1. 问题是，loop device 的实现方法在哪里，不会是 deriver/block/loop 吧
> 2. dev/loop 的文件是做什么的 ?  `mkfs.xfs -f /dev/loop0` 的作用是啥 ?

理解一下mount系统调用:
实际上flags + data对应mount命令的所有-o选项。那怎么区分哪些属于flags哪些属于data呢？
在深入内核代码一探究竟之前（本篇不深入了）我们可以通俗的认为flags就是所有文件系统通用的挂载选项，由VFS层解析。data是每个文件系统特有的挂载选项，由文件系统自己解析。[^4]
> 其中还总结了，/mnt/mi/linux/tools/include/uapi/linux/mount.h MS 之类的宏和 mount -o 的对应关系


file_system_type + super block + mount的关系 : file_system_type 一个文件系统中间只有一个，每一个 partitions 被 mount 之后，都可以创建一个 superblock，当其中的
1. sda1同时挂在了/mnt/a和/mnt/x上，所以它有两个挂载实例对应同一个super_block. : 当一个设备被mount的了，就会存在一个 superblock，但是之后无论 mount 多少次，都是只有一个
2. 每mount一次就会存在一个 mount 实例



相关的数据结构:
```c
// 这个结构体强调的是自己的信息
struct vfsmount {
	struct dentry *mnt_root;	/* root of the mounted tree */
	struct super_block *mnt_sb;	/* pointer to superblock */
	int mnt_flags;
} __randomize_layout;

// 和其他人关系
// 1. namespace
// 2. mount
struct mount {
	struct hlist_node mnt_hash; // 全局表格
	struct mount *mnt_parent; // 爸爸是谁
	struct dentry *mnt_mountpoint; // 放在爸爸的什么位置
	struct vfsmount mnt;
	union {
		struct rcu_head mnt_rcu;
		struct llist_node mnt_llist;
	};
#ifdef CONFIG_SMP
	struct mnt_pcp __percpu *mnt_pcp;
#else
	int mnt_count;
	int mnt_writers;
#endif
	struct list_head mnt_mounts;	/* list of children, anchored here */
	struct list_head mnt_child;	/* and going through their mnt_child */
	struct list_head mnt_instance;	/* mount instance on sb->s_mounts */
	const char *mnt_devname;	/* Name of device e.g. /dev/dsk/hda1 */
	struct list_head mnt_list; /* 链接到进程namespace中已挂载文件系统中，表头为mnt_namespace的list域 */
	struct list_head mnt_expire;	/* link in fs-specific expiry list */
	struct list_head mnt_share;	/* circular list of shared mounts */
	struct list_head mnt_slave_list;/* list of slave mounts */
	struct list_head mnt_slave;	/* slave list entry */
	struct mount *mnt_master;	/* slave is on master->mnt_slave_list */
	struct mnt_namespace *mnt_ns;	/* containing namespace */
	struct mountpoint *mnt_mp;	/* where is it mounted */
	union {
		struct hlist_node mnt_mp_list;	/* list mounts with the same mountpoint */
		struct hlist_node mnt_umount;
	};
	struct list_head mnt_umounting; /* list entry for umount propagation */
#ifdef CONFIG_FSNOTIFY
	struct fsnotify_mark_connector __rcu *mnt_fsnotify_marks;
	__u32 mnt_fsnotify_mask;
#endif
	int mnt_id;			/* mount identifier */
	int mnt_group_id;		/* peer group identifier */
	int mnt_expiry_mark;		/* true if marked for expiry */
	struct hlist_head mnt_pins;
	struct hlist_head mnt_stuck_children;
} __randomize_layout;
```


老版本的记录:
1. register_filesystem => file_system_type::mount => 调用文件系统自定义mount函数，该函数只是 mount_nodev 的封装，作用是提供自己的 fill_super
2. fill_super 完成的基本任务 : 


按照全新的版本 : fs_context_operations 即可
1. 参数解析
2. 上线

```c
static const struct fs_context_operations hugetlbfs_fs_context_ops = {
	.free		= hugetlbfs_fs_context_free,
	.parse_param	= hugetlbfs_parse_param,
	.get_tree	= hugetlbfs_get_tree,
};

static int hugetlbfs_get_tree(struct fs_context *fc)
{
	int err = hugetlbfs_validate(fc);
	if (err)
		return err;
	return get_tree_nodev(fc, hugetlbfs_fill_super);
  // @todo 总结一下 get_tree_nodev 对于含有 dev 对称函数
}
```

新版本:
1. some important calling graph of mount :
```c
kern_mount
  vfs_kern_mount : 调用特定文件系统的回调函数，构建一个vfsmount
      fs_context_for_mount
        alloc_fs_context : fc->fs_type->init_fs_context or legacy_init_fs_context
  fc_mount
      vfs_get_tree
          fc->ops->get_tree : get_tree is init in the alloc_fs_context, legacy_init_fs_context is init get_tree with legacy_get_tree 
              A : legacy_get_tree : fc->fs_type->mount
              B : get_tree_nodev : it seems that kernel always call get_tree_nodev eg. get_tree_nodev(fc, shmem_fill_super) : 
                    vfs_get_super
                        sget_fc
                        fill_super
      vfs_create_mount : Note that this does not attach the mount to anything.
```

```c
do_mount
  do_new_mount
    fs_context_for_mount : init the context 
    vfs_get_tree
    do_new_mount_fc
      vfs_create_mount
      do_add_mount : 将 vfsmount 结构放到全局目录树中间
        graft_tree
```


2. some important function :
    1. fs_context_for_mount : init context
    2. vfs_get_tree :  sget and fill_super
    3. vfs_create_mount : init vfsmount

从do_mount的代码可以它主要就是：
- 将dir_name解析为path格式到内核
- 一路解析flags位表，将flags拆分位mnt_flags和sb_flags
- 根据flags中的标记，决定下面做哪一个mount操作。

do_add_mount函数主要做两件事：
1. lock_mount确定本次挂载要挂载到哪个父挂载实例parent的哪个挂载点mp上。
2. 把newmnt挂载到parent的mp下，完成newmnt到全局的安装。安装后的样子就像我们前文讲述的那样。

#### pivot_root
1. 为什么更改 root 的mount 位置存在这种诡异的需求啊
    1. 找到使用这个的软件


## superblock
super 作为一个文件系统实例的信息管理中心，很多动态信息依赖于 superlock.
    1. sync，fs-writeback
    2. remove/create inode / file

mount 的时候，首先需要加载 superblock

问题1 : kill super 到底是在做什么 ?
```c
void kill_anon_super(struct super_block *sb)
{
	dev_t dev = sb->s_dev;
	generic_shutdown_super(sb);
	free_anon_bdev(dev);
}

void kill_litter_super(struct super_block *sb)
{
	if (sb->s_root)
		d_genocide(sb->s_root);
	kill_anon_super(sb);
}

void kill_block_super(struct super_block *sb) // 提供基于block 的 fs, ext2 fat 之类的
{
	struct block_device *bdev = sb->s_bdev;
	fmode_t mode = sb->s_mode;

	bdev->bd_super = NULL;
	generic_shutdown_super(sb); // TODO 
	sync_blockdev(bdev);
	WARN_ON_ONCE(!(mode & FMODE_EXCL));
	blkdev_put(bdev, mode | FMODE_EXCL);
}
```
- kill_block_super(), which unmounts a file system on a block device
- kill_anon_super(), which unmounts a virtual file system (information is generated when requested)
- kill_litter_super(), which unmounts a file system that is not on a physical device (the information is kept in memory)
An example for a file system without disk support is the ramfs_mount() function in the ramfs file system:
> kill_anon_super 和 kill_litter_super 使用标准是什么，基于内存的和 generated when needed 的区别还是很清晰的，
> 但是检查了一下 reference ，感觉存在疑惑 !
> 其次 TODO 这些 kill_super 函数 和 super_operation::put_super 的关系是什么 ?

问题2 : 各种 mount 辅助函数，搞不清楚到底谁在使用新版本的接口
- mount_bdev(), which mounts a file system stored on a block device
- mount_single(), which mounts a file system that shares an instance between all mount operations
- mount_nodev(), which mounts a file system that is not on a physical device
- mount_pseudo(), a helper function for pseudo-file systems (sockfs, pipefs, generally file systems that can not be mounted)

#### super operations
> TODO 这些都是需要检查一下，总结一下的。
- alloc_inode: allocates an inode. Usually, this funcion allocates a struct <fsname>_inode_info structure and performs basic VFS inode initialization (using inode_init_once()); minix uses for allocation the kmem_cache_alloc() function that interacts with the SLAB subsystem. For each allocation, the cache construction is called, which in the case of minix is the init_once() function. Alternatively, kmalloc() can be used, in which case the inode_init_once() function should be called. The alloc_inode() function will be called by the new_inode() and iget_locked() functions.
- destroy_inode releases the memory occupied by inode
- write_inode : 对于一个 inode 的信息进行更新
- - evict_inode: removes any information about the inode with the number received in the i_ino field **from the disk and memory** (both the inode on the disk and the associated data blocks). This involves performing the following operations:

```c
static const struct super_operations myfs_sops = {
	.statfs = simple_statfs,
	.drop_inode = generic_delete_inode,
};
```




## dentry
存在如下的问题:
1. vfs inode 对应磁盘中间的一个 disk inode，dentry 是不是也是磁盘中间 dentry 的对应 ?
    1. 加载过程和写回过程 ?
2. 其中 d_op 的作用是什么 ?
3. dcache 和 dentry 的关系
    1. 有没有拿掉 dcache 的操作，这种选项
1. Everything is file.
    1. directory is file too ! Find the evidence :
        1. `struct dentry` for ext2 : contains filename + inode number
            1. ino -> inode table index -> fetch inode content
            2. inode content -> file
        2. how ext2 init `struct dentry` ?
        3. file whoes type is directory : a array of dentry !
        4. but dir and file is different : there are there are different file operation and inode operations for directory
            1. (file directory) (inode file)
2. how `struct dentry` are init ? walk_component => lookup_slow
3. if a directory contains thousands of files, so we will create thousands of struct dentry when open the directory ?


1. `d_make_root`: allocates the root dentry. It is generally used in the function that is called to read the superblock (fill_super), which must initialize the root directory. So the root inode is obtained from the superblock and is used as an argument to this function, to fill the s_root field from the struct super_block structure.
2. `d_add`: associates a dentry with an inode; the dentry received as a parameter in the calls discussed above signifies the entry (name, length) that needs to be created. This function will be used when creating/loading a new inode that does not have a dentry associated with it and has not yet been introduced to the hash table of inodes (at lookup); 将 新创建的 inode 和 其 dentry 关联起来。
3. `d_instantiate`: The lighter version of the previous call, in which the dentry was previously added in the hash table.

```c
struct dentry {
	/* RCU lookup touched fields */
	unsigned int d_flags;		/* protected by d_lock */
	seqcount_t d_seq;		/* per dentry seqlock */
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

	/* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */
	const struct dentry_operations *d_op;
	struct super_block *d_sb;	/* The root of the dentry tree */
	unsigned long d_time;		/* used by d_revalidate */
	void *d_fsdata;			/* fs-specific data */

	union {
		struct list_head d_lru;		/* LRU list */
		wait_queue_head_t *d_wait;	/* in-lookup ones only */
	};
	struct list_head d_child;	/* child of parent list */
	struct list_head d_subdirs;	/* our children */
	/*
	 * d_alias and d_rcu can share memory
	 */
	union {
		struct hlist_node d_alias;	/* inode alias list */
		struct hlist_bl_node d_in_lookup_hash;	/* only for in-lookup ones */
	 	struct rcu_head d_rcu;
	} d_u;
} __randomize_layout;
```

negative dentry [^6] :
1. dentry is a way of remembering the resolution of a given file or directory name without having to search through the filesystem to find it.
2. A negative dentry is a little different, though: it is a memory of a filesystem lookup that failed.


#### path

```c
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
} __randomize_layout;
```
> 原来，path 只是一个 dentry !


## helper 
1. `inode_init_owner` : 
2. get_next_ino : 调用者只一些虚拟的文件系统。
    1. 为什么虚拟文件需要 ino ，又不需要和磁盘打交道
```c
/*
 * Each cpu owns a range of LAST_INO_BATCH numbers.
 * 'shared_last_ino' is dirtied only once out of LAST_INO_BATCH allocations,
 * to renew the exhausted range.
 *
 * This does not significantly increase overflow rate because every CPU can
 * consume at most LAST_INO_BATCH-1 unused inode numbers. So there is
 * NR_CPUS*(LAST_INO_BATCH-1) wastage. At 4096 and 1024, this is ~0.1% of the
 * 2^32 range, and is a worst-case. Even a 50% wastage would only increase
 * overflow rate by 2x, which does not seem too significant.
 *
 * On a 32bit, non LFS stat() call, glibc will generate an EOVERFLOW
 * error if st_ino won't fit in target struct field. Use 32bit counter
 * here to attempt to avoid that.
 */
#define LAST_INO_BATCH 1024
static DEFINE_PER_CPU(unsigned int, last_ino);

unsigned int get_next_ino(void)
{
	unsigned int *p = &get_cpu_var(last_ino);
	unsigned int res = *p;

#ifdef CONFIG_SMP
	if (unlikely((res & (LAST_INO_BATCH-1)) == 0)) {
		static atomic_t shared_last_ino;
		int next = atomic_add_return(LAST_INO_BATCH, &shared_last_ino);

		res = next - LAST_INO_BATCH;
	}
#endif

	res++;
	/* get_next_ino should not provide a 0 inode number */
	if (unlikely(!res))
		res++;
	*p = res;
	put_cpu_var(last_ino);
	return res;
}
```

3. `d_make_root` : 通过 root inode 创建出来对应的 dentry，所有的inode 都是存在对应的 dentry 并且存放在 parent direcory file 中间，但是唯独 root 不行。

```c
struct dentry *d_make_root(struct inode *root_inode)
{
	struct dentry *res = NULL;

	if (root_inode) {
		res = d_alloc_anon(root_inode->i_sb);
		if (res) {
			res->d_flags |= DCACHE_RCUACCESS;
			d_instantiate(res, root_inode);
		} else {
			iput(root_inode);
		}
	}
	return res;
}
```

4. `inode_init_always` : 基本的初始化

5. `inode_init_once` : 所以，inode_init_always 和 inode_init_once 存在什么区别吗?
inode_init_once 看来作用更像是更加基本的初始化，在 minfs 中间之所以需要单独调用 init_once，
因为其使用的是通用的 kmalloc 机制。

```c
static int __init init_inodecache(void)
{
	ext4_inode_cachep = kmem_cache_create_usercopy("ext4_inode_cache",
				sizeof(struct ext4_inode_info), 0,
				(SLAB_RECLAIM_ACCOUNT|SLAB_MEM_SPREAD|
					SLAB_ACCOUNT),
				offsetof(struct ext4_inode_info, i_data),
				sizeof_field(struct ext4_inode_info, i_data),
				init_once); // 
	if (ext4_inode_cachep == NULL)
		return -ENOMEM;
	return 0;
}
```

## attr
1. 是不是还有一个扩展属性的一个东西 ?

2. 看了一下 ext2_setattr 和 ext2_getattr 的实现，只是提供 inode 的各种基本属性而已。


## open.c
vfs 提供的基础设施总结一下
1. lookup
2. open ? 为什么 open 需要单独分析

getname_kernel 和  getname_flags 有什么区别吗 ?

## fuse
https://github.com/libfuse/libfuse

利用 fuse 从而实现将 ntfs 挂载出来的，有意思啊!

附带一个 blog
https://blog.betacat.io/post/2020/08/how-to-mount-etcd-as-a-filesystem/

## block layer
似乎 block layer 被取消了，但是似乎 mq 又是存在的 ?
一个 block 被发送到被使用:
```python
b.attach_kprobe(event="blk_mq_start_request", fn_name="trace_start")
b.attach_kprobe(event="blk_account_io_completion", fn_name="trace_completion")
```


## initfs
[Documentation for ramfs, rootfs, initramfs.](https://lwn.net/Articles/156098/)

https://landley.net/writing/rootfs-intro.html
1. 因为需要 mount rootfs，在曾经的世界中间，给内核指定一个参数就可以了。
2. 如果是靠指定参数的方法，为什么还需要 initrd ?
3. 既然 initrd 还是可以使用的，为什么 qemu 不能运行 ?

Robert love 谈到 initramfs 的作用:
https://qr.ae/pNs0eS 
但是还是让人非常迷惑:
1. grub 如何让 ssd 中间的 initramfs 被加载到内存
2. nvme 模块如何被加载内存
3. kernel image 如何被加载到内存的

## overlay fs


## inode
感觉 inode.c 其实和 dcache.c 是对称的，inode 本身作为 cache 的，与需要加载，删除，初始化等操作

1. inode.c 中间存在好几个结尾数字为5的函数，表示什么含义啊 ?

其中的集大成者是 iget5_locked ?
```c
struct inode *iget5_locked(struct super_block *sb, unsigned long hashval,
		int (*test)(struct inode *, void *),
		int (*set)(struct inode *, void *), void *data)
{
	struct inode *inode = ilookup5(sb, hashval, test, data);

	if (!inode) {
		struct inode *new = alloc_inode(sb);

		if (new) {
			new->i_state = 0;
			inode = inode_insert5(new, hashval, test, set, data);
			if (unlikely(inode != new))
				destroy_inode(new);
		}
	}
	return inode;
}
```
真正的有意义的调用:

```c
struct block_device *bdget(dev_t dev)
{
	struct block_device *bdev;
	struct inode *inode;

	inode = iget5_locked(blockdev_superblock, hash(dev),
			bdev_test, bdev_set, &dev);

```

有意思
2. 注释，根本不能理解，inode 数量不够是啥意思啊
 * This is a generalized version of ilookup() for file systems where the
 * inode number is not sufficient for unique identification of an inode.

3. 两个版本的函数都是存在的 ilookup 和 ilookup5



* ***hash***
1. 也是存在 inode_hashtable 的，就像是 dentry_hashtable 一样
2. 之所以使用 hash 而不是 inode number，是为了防止多个文件系统，inode numebr 互相重复吧!

```c
static unsigned long hash(struct super_block *sb, unsigned long hashval)
{
	unsigned long tmp;

	tmp = (hashval * (unsigned long)sb) ^ (GOLDEN_RATIO_PRIME + hashval) /
			L1_CACHE_BYTES;
	tmp = tmp ^ ((tmp ^ GOLDEN_RATIO_PRIME) >> i_hash_shift);
	return tmp & i_hash_mask;
}
```



## dcache
问题
1. alias 是做什么的 ?
    1. d_move 的作用
2. d_ops 的作用是什么
3. negtive 怎么产生的 ?

* ***shrinker***

1. alloc_super 中间创建了一个完成 shrinker 的初始化 TODO 这个 shrinker 机制还有人用吗 ?
    1. super_cache_scan
    2. super_cache_count

2. `struct superpage` 中间存在两个函数 TODO 应该用于特殊内容的缓存的
```c
	long (*nr_cached_objects)(struct super_block *,
				  struct shrink_control *);
	long (*free_cached_objects)(struct super_block *,
				    struct shrink_control *);
```
3.  super_cache_scan 调用两个函数
    1. prune_dcache_sb 
    2. prune_icache_sb
```c
long prune_dcache_sb(struct super_block *sb, struct shrink_control *sc)
{
	LIST_HEAD(dispose);
	long freed;

	freed = list_lru_shrink_walk(&sb->s_dentry_lru, sc,
				     dentry_lru_isolate, &dispose);
	shrink_dentry_list(&dispose);
	return freed;
}

prune_icache_sb : TODO 这个是用于释放 inode 还是 inode 持有的文件 ? 还是当 inode 被打开之后就不释放 ?
```

4. 
list_lru_shrink_walk 似乎就是遍历一下列表，将可以清理的页面放出来
```c
static enum lru_status dentry_lru_isolate(struct list_head *item,
		struct list_lru_one *lru, spinlock_t *lru_lock, void *arg)
```
然后使用 prune_dcache_sb 紧接着调用 shrink_dentry_list ，将刚刚清理出来的内容真正的释放:

5. shrink 的源头还有 unmount 的时候 





分析一个 cache 基本方法:
1. 加入
2. 删除
3. 查询

There are a number of functions defined which permit a filesystem to manipulate dentries:
dget : open a new handle for an existing dentry (this just increments the usage count)
dput : close a handle for a dentry (decrements the usage count). If the usage count drops to 0, and the dentry is still in its parent’s hash, the “d_delete” method is called to check whether it should be cached. If it should not be cached, or if the dentry is not hashed, it is deleted. Otherwise cached dentries are put into an LRU list to be reclaimed on memory shortage.
d_drop : **this unhashes a dentry from its parents hash list.** A subsequent call to dput() will deallocate the dentry if its usage count drops to 0
d_delete : delete a dentry. If there are no other open references to the dentry then the dentry is turned into a negative dentry (the d_iput() method is called). **If there are other references, then d_drop() is called instead**
d_add : add a dentry to its parents hash list and then calls d_instantiate()
d_instantiate : add a dentry to the alias hash list for the inode and updates the “d_inode” member. The “i_count” member in the inode structure should be set/incremented. If the inode pointer is NULL, the dentry is called a “negative dentry”. This function is commonly called when an inode is created for an existing negative dentry
d_lookup : look up a dentry given its parent and path name component It looks up the child of that given name from the dcache hash table. If it is found, the reference count is incremented and the dentry is returned. The caller must use dput() to free the dentry when it finishes using it.

* ***分析: d_lookup***

d_lookup 相比 `__d_lookup` 多出了 rename 的 lock 问题，查询是在 parent 是否存在指定的 children
```c
static struct hlist_bl_head *dentry_hashtable __read_mostly;

static inline struct hlist_bl_head *d_hash(unsigned int hash)
{
	return dentry_hashtable + (hash >> d_hash_shift);
}
```

按照字符串的名字，找到位置，然后对比。问题是:
1. 似乎 dentry_hashtable 是一个全局的，如果文件系统中间存在几千个 readme.md 岂不是 GG
2. hlist 是怎么维护的呀，大小如何初始化呀

* ***d_add***

1. d_add 调用位置太少了，比较通用的调用位置在 libfs 中间，
2. 其核心执行的内容如下，也就是存在两个 hash，全局的，每个

		hlist_add_head(&dentry->d_u.d_alias, &inode->i_dentry);
3. 实际上，依托 hlist_bl_add_head_rcu d_hash 加入 dentry_hashtable


The “dcache” caches information about names in each filesystem to make them quickly available for lookup. 
Each entry (known as a “dentry”) contains three significant fields: 
1. a component name, 
2. *a pointer to a parent dentry*, 
3. and a pointer to the “inode” which contains further information about the object in that parent with the given name.
> 为啥需要包含 parent dentry ?


## lvm
https://opensource.com/business/16/9/linux-users-guide-lvm



## splice and pipe
内核似乎没有文档 : https://www.kernel.org/doc/html/latest/filesystems/splice.html

## IO model
利用 fio 总结一下常用的 IO 模型

## devpts
https://www.kernel.org/doc/html/latest/filesystems/devpts.html


## dup(merge)
1. 为什么会存在 dup 一个 fd 的需求啊
2. 多个 file 会指向 inode，因为对于文件的读写状态不同的
    1. 但是为什么会存在多个 file descriptor 指向同一个 fd 啊 ?

Duplicated file descriptors created by dup2() or fork() point to the same file object.
## IO buffer(merge)
1. tlpi 的 chapter 13 可以阅读一下

## notify
1. io_uring 或者 aio 或者 之类的机制可以替代 inotify 吗 ?
2. 可以搞一下的细节:
    1. fsnotify_group 是怎么回事
    2. 如何通知支持 inotify 和 dnotify 两个模块的


* ***如何实现告知 ?***
```c
fsnotify_modify : 在 open.c 中间可以查看到
/*
 * fsnotify_modify - file was modified
 */
static inline void fsnotify_modify(struct file *file) { fsnotify_file(file, FS_MODIFY); }

/**
 * This is the main call to fsnotify.  The VFS calls into hook specific functions
 * in linux/fsnotify.h.  Those functions then in turn call here.  Here will call
 * out to all of the registered fsnotify_group.  Those groups can then use the
 * notification event in whatever means they feel necessary.
 */
int fsnotify(struct inode *to_tell, __u32 mask, const void *data, int data_is,
	     const struct qstr *file_name, u32 cookie)
```

* ***如何让从 notifyFd 中间读出 event***

```c
/* inotify syscalls */
static int do_inotify_init(int flags)
{
	struct fsnotify_group *group;
	int ret;

	/* Check the IN_* constants for consistency.  */
	BUILD_BUG_ON(IN_CLOEXEC != O_CLOEXEC);
	BUILD_BUG_ON(IN_NONBLOCK != O_NONBLOCK);

	if (flags & ~(IN_CLOEXEC | IN_NONBLOCK))
		return -EINVAL;

	/* fsnotify_obtain_group took a reference to group, we put this when we kill the file in the end */
	group = inotify_new_group(inotify_max_queued_events);
	if (IS_ERR(group))
		return PTR_ERR(group);

	ret = anon_inode_getfd("inotify", &inotify_fops, group, // 这就是关键吧
				  O_RDONLY | flags);
	if (ret < 0)
		fsnotify_destroy_group(group);

	return ret;
}
```

notify 的工作，和 aio epoll 机制其实都是类似的，相比于 blocking io，每一个 IO 需要一个 thread 阻塞，而切换为 epoll，多个 IO 被阻塞到同一个 thread 而已，还是说其实 aio 是存在各种设计模式的 TODO

## attr

## direct IO
1. 到底 DIRECT_IO flags 的效果是什么
2. dax 机制是什么 ?

## block device
1. 也许读一下 ldd3 比较有用 ?
2. 现在的问题是，根本搞不清楚，为什么，为什么 char dev 是有效的

## char device

## tmpfs
cat /proc/mounts : we find the filesystem type of /dev is tmpfs, so what's the difference between fs between 

## exciting
http://www.betrfs.org/
https://wiki.archlinux.org/index.php/LVM : 卷 逻辑卷 到底是啥 ?

## benchmark
https://lwn.net/Articles/385081/
https://research.cs.wic.edu/wind/Publications/impressions-fast09.pdfs

#### fio
> TODO 将 fio 的类容整理到此处, 然后将那个文件夹删除

#### filebench
下面记录 [^12]:
I/O Microbenchmarking
Trace Capture/Reply
Model Based

FileBench is an application simulator

"Benchpoint" Run Generation Wrapper ??????
prof : 真的存在吗 ?

下面记录 [^11]:

There are four main entities in WML: fileset, process, thread, and flowp

A Filebenh run proceeds in two stages: fileset preallocation and
an actual workload executionc

By default, Filebench does not create any files in the file system and only allocates enough internal
file entries to accommodate all defined files.
> internal file entries 是 filebench 自己管理的吧 !

Every thread executes a loop of flowops. 
> 能不能 randomly choose a file

Filebench uses Virtual Fie Descriptors (VFDs) to refer to files in flowops.

When opening a file, Filebench first needs to pick afile from a
fileset. *By default this is done by iterating over all file entries in
a fileset.* To change this behavior one can use the index attribute that allows one to refer to a specific file in a fileset using
a unique index. In most real cases, instead of using a constant
number for the index, one should use custom variables described
in the following section 
> fd 自动顺序选择，是不是 fd 默认为 1 

The abilty to quickly define multiple processes and synchronization between them was one of the main requirements during
Filebench framework conception.
> Filebench 设计本来的目的是对于实际 workload 的模拟

https://github.com/filebench/filebench/issues/130
似乎有的需要添加上 `run time`

https://github.com/filebench/filebench/issues/112
```
echo 0 > /proc/sys/kernel/ranomize_va_spaced
```
解决 wait pid 的问题


**还存在的一些问题:**
1. 虽然 配置 中间显示只是使用一个 process 和 一个 thread，但是 htop 显示存在两个
      1. 使用 4 process 1 thread 的结构，结果
2. 感觉文件都是没有被创建过的样子
3. prealloc 的含义是什么 ?
4. 是不是只有循环这个操作，那么调整 read / write 之类的比例，岂不是直接 GG
When openinga file, Filebench first needs to pick a file from a
fileset. By default this is done by iterating over all file entries in
a fileset. To change this behavior one can use the index attribute that allows one to refer to a specific file in a fileset using
a unique *index*. In most real cases, instead of using a constant
number for the index, one should use custom variables described
in the following section. 
> index 从来没有体现过作用!


使用的记录:
1. read write 必须配对，但是可以都去掉.
2. 通过 debug 10 得到的日志，显示默认 fd = 1 的方法，如果

运行一下各个结果:
1. finishoncount : 似乎没有作用
2. fd=1 是用于一个 loop 中，而不是用于选择文件的

计划:
1. fileset 的构建
2. 这个支持太少了，有必要吗增加吗 ? open 是测试 vfs 的性能，还是测试

## nvme
multi-stream :
https://m.weixin.qq.com/s/iXIhqqP-j1zxMIjA31G4fwp
https:/www.samsung.com/us/labs/pdfs/2016-08-fms-multi-stream-v4.pdf/


## fallocate
本以为只是快速实现创建大文件，实际上可以进行很多有意思的操作，可以简单调查一下.

## union fs
docker


[^1]: [kernel doc : Overview of the Linux Virtual File System](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)
[^2]: [github : aio](https://github.com/littledan/linux-aio)
[^3]: [stackexchange : loop device](https://unix.stackexchange.com/questions/66075/when-mounting-when-should-i-use-a-loop-device)
[^4]: [zhihu : mount syscall](https://zhuanlan.zhihu.com/p/36268333)
[^5]: [zhihu : mount](https://zhuanlan.zhihu.com/p/76419740)
[^6]: [lwn : Dentry negativity](https://lwn.net/Articles/814535/)
[^7]: [io uring and eBPF](https://thenewstack.io/how-io_uring-and-ebpf-will-revolutionize-programming-in-linux/)
[^8]: [zhihu : io_uring introduction](https://zhuanlan.zhihu.com/p/62682475?utm_source=wechat_timeline)
[^9]: [blog : File locking in Linux](https://gavv.github.io/articles/file-locks/)
[^10]: [Efficient IO with io_uring](https://kernel.dk/io_uring.pdf)
[^11]: [usenix : Filebench a flexible framework for fs benchmark](https://www.usenix.org/system/files/login/articles/login_spring16_02_tarasov.pdf)
[^12]: [sun : Filebench tutorial](http://www.nfsv4bat.org/Documents/nasconf/2005/mcdougall.pdf)