---
layout: post
title: "fd, fd-table, open fd-table (linux, fd)"
author: "melon"
date: 2024-06-18 21:59
categories: "2024"
tags:
  - linux
---

everything in linux can be seen as file, the fd is the generaic handler to all things.
file descriptor (fd) can be seen as a representation of resources in linux system.

the fd is used by the kernel as references to all kinds of resources in system,
and as hook point of the address from device memory through memory map operations.

<hr>

### # graphic model of the relationship among fd & fdt & ofdt

<img src="https://cdn.jsdelivr.net/gh/slothfull/cdn@main/image/fdt.pdf" width="750"/>

1 the first 3 fd stdin(0), stdout(1) and stderr(2) are special fd.
in this case, all of them point to a pseudo-terminal (/dev/pts/0).
they have no positions in open fd table due to their character device type.
thus, proc#l and proc#2 must be running under the terminal sessions.

2 normally, only open fd table entries got their own flag, however, some fd can
have per-process flags. till now there is only one such flag: close-on-exec (O_CLOEXEC).
in this case, the fd 9 of process 1 and fd 3 of process 2 are point to the same
system-wide open fd table entries.

3 even though the fd algorithm constantly reuses the fd and allocates them sequentially
from the lowest available, it doesn’t mean that fd number gaps cannot occur.
in this case, the fd 9 of the process 1 goes after fd 3.
one reason could be: the files used fd 4, 5, 6 and 7 are already closed.
another reason could be: an explicit duplication of a fd using dup2, dup3 or fcntl with F_DUPFD.
using these syscalls, we can specify the wanted fd number.

4 a process can have >=1 fd points to the same entry in open file descriptions.
system calls dup, dup2, dup3 and fcntl with F_DUPFD help with that.
in this case, fd 0 and fd 2 of the process 2 refer to the same pseudo terminal entry.

5 stdout fd might point to a regular file or pipe but not a terminal.
in this case, the stdout of the process 2 refers to a file on disk /tmp/out.txt.

6 fd from various processes can point to the same entry in system-wide open fd table.
this can be achieved by fork syscall to inherit fd from parent proc to its child.

7 in this case the file path is shown for similarity, but actually kernel uses inode,
minor and major number of a device.

8 running in the shell context, 0,1 and 2 fd pointed to a pseudo-terminal.

9 multiple open fd entries can be linked with the same physical file on disk.
the kernel allows us to open a file with different flags and at various offset positions.

<hr>

### # fd & fdt & ofdt structures in kernel
1 after linux abstract the tasks by thread, the fd is available inside thread's obj:

```text
struct task_struct {
    ...
    struct files_struct* files;
    ...
}
```

<p style="margin-bottom: 20px;"></p>

2 file descriptor table (per-process fdtable) can be found inside files_struct, it defines all the fd
managed by current kernel thread.

```text
struct files_struct {
	atomic_t              count;                         // read most part
	bool                  resize_in_progress;
	wait_queue_head_t     resize_wait;

	struct fdtable __rcu* fdt;                           // per process fd table, rcu: speed-up write to fdtable,
	struct fdtable        fdtab;                         // targeting for scenario as always read seldom write

	spinlock_t file_lock  ____cacheline_aligned_in_smp;  // written part on a separate cache line (smp)
	unsigned int          next_fd;
	unsigned long         close_on_exec_init[1];
	unsigned long         open_fds_init[1];
	unsigned long         full_fds_bits_init[1];
	struct file __rcu*    fd_array[NR_OPEN_DEFAULT];
};
```

ref: https://github.com/torvalds/linux/blob/master/include/linux/fdtable.h

<p style="margin-bottom: 20px;"></p>

3 open file table is a system-wide table maintaining all opened instances.
each entries in this table contains some extra file status info: file_offset, file_status\...

```text
// low-level design ideas:
// f_{lock,count,pos_lock} members can be highly contended and share the same cacheline.
// f_{lock,mode} are very frequently used together and so share the same cacheline as well.
// the read-mostly f_{path,inode,op} are kept on a separate cacheline.

struct file {
    union {
        struct callback_head      f_task_work;   // fput() uses this when close & free file
        struct llist_node         f_llist;       // fput() must use workqueue (most kernel threads)
        unsigned int              f_iocb_flags;
    };
                                                 // for security: keep f_ep, f_flags out the reach of irq context
    spinlock_t                    f_lock;
    fmode_t                       f_mode;
    atomic_long_t                 f_count;
    struct mutex                  f_pos_lock;
    loff_t                        f_pos;
    unsigned int                  f_flags;
    struct fown_struct            f_owner;
    const struct cred*            f_cred;
    struct file_ra_state          f_ra;
    struct path                   f_path;
    struct inode*                 f_inode;       // system-wide inode table, cached value
    const struct file_operations* f_op;

    u64                           f_version;
#ifdef CONFIG_SECURITY
    void*                         f_security;
#endif
    void*                         private_data;  // needed for tty drv or others

#ifdef CONFIG_EPOLL
    struct hlist_head*            f_ep;          // used by fs/eventpoll.c to link all the hooks to this file
#endif
    struct address_space*         f_mapping;
    errseq_t                      f_wb_err;
    errseq_t                      f_sb_err;      // for syncfs
} __randomize_layout
  __attribute__((aligned(4)));	                 // lest something weird decides that 2 is OK
```

ref: https://github.com/torvalds/linux/blob/master/include/linux/fs.h

<p style="margin-bottom: 20px;"></p>

4 inode table is a system-wide table holding all real file-like resources in its entries.

```text
// note: keep mostly read-only and often accessed fields at the beginning of the struct inode,
// especially for the rcu path lookup and stat data.

struct inode {
    umode_t                               i_mode;
    unsigned short                        i_opflags;
    kuid_t                                i_uid;
    kgid_t                                i_gid;
    unsigned int                          i_flags;

#ifdef CONFIG_FS_POSIX_ACL
    struct posix_acl*                     i_acl;
    struct posix_acl*                     i_default_acl;
#endif

    const struct inode_operations*        i_op;
    struct super_block*                   i_sb;
    struct address_space*                 i_mapping;

#ifdef CONFIG_SECURITY
    void*                                 i_security;
#endif

    unsigned long                         i_ino;              // data for stat, not shown during path walking
    union {                                                   // for fs only read i_nlink, they should use api:
        const unsigned int                i_nlink;            // set/clear/inc/drop_nlink, inode_inc/dec_link_count
        unsigned int                      __i_nlink;
    };
    dev_t                                 i_rdev;
    loff_t                                i_size;
    struct timespec64                     __i_atime;
    struct timespec64                     __i_mtime;
    struct timespec64                     __i_ctime;          // use inode_*_ctime accessors!
    spinlock_t                            i_lock;             // i_blocks, i_bytes, maybe i_size
    unsigned short                        i_bytes;
    u8                                    i_blkbits;
    enum rw_hint                          i_write_hint;
    blkcnt_t                              i_blocks;

#ifdef __NEED_I_SIZE_ORDERED
    seqcount_t                            i_size_seqcount;
#endif
                                                              // misc
    unsigned long                         i_state;
    struct rw_semaphore	                  i_rwsem;

    unsigned long                         dirtied_when;       // jiffies of first dirtying
    unsigned long                         dirtied_time_when;

    struct hlist_node                     i_hash;
    struct list_head                      i_io_list;          // backing dev io list
#ifdef CONFIG_CGROUP_WRITEBACK
    struct bdi_writeback*                 i_wb;               // the associated cgroup wb

    int                                   i_wb_frn_winner;    // foreign inode detection, see wbc_detach_inode()
    u16                                   i_wb_frn_avg_time;
    u16                                   i_wb_frn_history;
#endif
    struct list_head                      i_lru;              // inode LRU list
    struct list_head                      i_sb_list;
    struct list_head                      i_wb_list;	      // backing dev writeback list
    union {
        struct hlist_head                 i_dentry;
        struct rcu_head	                  i_rcu;
    };
    atomic64_t                            i_version;
    atomic64_t                            i_sequence;         // see futex
    atomic_t                              i_count;
    atomic_t                              i_dio_count;
    atomic_t                              i_writecount;
#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
    atomic_t                              i_readcount;        // struct files open ro
#endif
    union {
        const struct file_operations*     i_fop;              // former ->i_op->default_file_ops */
        void (*free_inode)(struct inode*);
    };
    struct file_lock_context*             i_flctx;
    struct address_space                  i_data;
    struct list_head                      i_devices;
    union {
        struct pipe_inode_info*           i_pipe;
        struct cdev*                      i_cdev;
        char*                             i_link;
        unsigned                          i_dir_seq;
    };

    __u32                                 i_generation;

#ifdef CONFIG_FSNOTIFY
    __u32                                 i_fsnotify_mask;    // all events this inode cares about
    struct fsnotify_mark_connector __rcu* i_fsnotify_marks;
#endif

#ifdef CONFIG_FS_ENCRYPTION
    struct fscrypt_inode_info*            i_crypt_info;
#endif

#ifdef CONFIG_FS_VERITY
    struct fsverity_info*                 i_verity_info;
#endif

    void*                                 i_private;          // fs or device private pointer
} __randomize_layout;
```

ref: https://github.com/torvalds/linux/blob/master/include/linux/fs.h

<hr>

### # todo
ref: https://biriukov.dev/docs/fd-pipe-session-terminal/1-file-descriptor-and-open-file-description/
