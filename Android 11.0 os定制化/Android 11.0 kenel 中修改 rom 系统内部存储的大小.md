> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/baidu_41666295/article/details/131114588)

1. 前言
-----

 在 11.0 的系统 rom 产品开发定制中，在对一些产品硬件设备配置要求搞的需求方面，由于在产品后续订单中，有些产品是出口的，但是硬件方面已经定板，[时间比较](https://so.csdn.net/so/search?q=%E6%97%B6%E9%97%B4%E6%AF%94%E8%BE%83&spm=1001.2101.3001.7020)仓促，所以  
就需要软件方面在 rom 内部存储的大小方面作假，修改 rom 真实的大小容量，所以就需要在 kenel 驱动部分来修改这部分的值最好了，接下来分析下计算 rom 容量的  
相关代码，然后做出修改

2.kenel 中修改 rom 系统内部存储的大小的核心类
-----------------------------

```
    kernel\kernel4.14\fs\statfs.c
    kernel/kernel4.14/include/linux/statfs.h
```

3.kenel 中修改 rom 系统内部存储的大小的核心功能分析和实现  
3.1 statfs.h 中相关 rom 参数的分析
----------------------------------------------------------------

```
   /* SPDX-License-Identifier: GPL-2.0 */
    #ifndef _LINUX_STATFS_H
    #define _LINUX_STATFS_H
     
    #include <linux/types.h>
    #include <asm/statfs.h>
     
    struct kstatfs {
        long f_type;
        long f_bsize;
        u64 f_blocks;
        u64 f_bfree;
        u64 f_bavail;
        u64 f_files;
        u64 f_ffree;
        __kernel_fsid_t f_fsid;
        long f_namelen;
        long f_frsize;
        long f_flags;
        long f_spare[4];
    };
     
    #define ST_RDONLY    0x0001    /* mount read-only */
    #define ST_NOSUID    0x0002    /* ignore suid and sgid bits */
    #define ST_NODEV    0x0004    /* disallow access to device special files */
    #define ST_NOEXEC    0x0008    /* disallow program execution */
    #define ST_SYNCHRONOUS    0x0010    /* writes are synced at once */
    #define ST_VALID    0x0020    /* f_flags support is implemented */
    #define ST_MANDLOCK    0x0040    /* allow mandatory locks on an FS */
    /* 0x0080 used for ST_WRITE in glibc */
    /* 0x0100 used for ST_APPEND in glibc */
    /* 0x0200 used for ST_IMMUTABLE in glibc */
    #define ST_NOATIME    0x0400    /* do not update access times */
    #define ST_NODIRATIME    0x0800    /* do not update directory access times */
    #define ST_RELATIME    0x1000    /* update atime relative to mtime/ctime */
     
    #endif
```

在 statfs.h 中的上述源码中，可以看出在在表示内部存储的基本参数  
f_frsize 基本的文件系统块大小  
f_ffree 空闲文件（inode）数量。  
f_bavail 内部存储剩余文件系统块的大小  
f_blocks 系统文件系统块大小  
f_files 系统文件总数  
f_bsize 文件系统块大小  
f_ffree 空闲文件计数  
f_namemax 最大文件名长度  
通过上述几个内部存储参数的讲解，发现最核心的就是 f_blocks 系统内部存储的容量大小，和  
f_bavail 系统内部存储可用容量的大小

3.2 statfs.c 中关于设置 rom 容量大小的相关代码分析
----------------------------------

```
   // SPDX-License-Identifier: GPL-2.0
    #include <linux/syscalls.h>
    #include <linux/export.h>
    #include <linux/fs.h>
    #include <linux/file.h>
    #include <linux/mount.h>
    #include <linux/namei.h>
    #include <linux/statfs.h>
    #include <linux/security.h>
    #include <linux/uaccess.h>
    #include <linux/compat.h>
    #include "internal.h"
     
    static int flags_by_mnt(int mnt_flags)
    {
        int flags = 0;
     
        if (mnt_flags & MNT_READONLY)
            flags |= ST_RDONLY;
        if (mnt_flags & MNT_NOSUID)
            flags |= ST_NOSUID;
        if (mnt_flags & MNT_NODEV)
            flags |= ST_NODEV;
        if (mnt_flags & MNT_NOEXEC)
            flags |= ST_NOEXEC;
        if (mnt_flags & MNT_NOATIME)
            flags |= ST_NOATIME;
        if (mnt_flags & MNT_NODIRATIME)
            flags |= ST_NODIRATIME;
        if (mnt_flags & MNT_RELATIME)
            flags |= ST_RELATIME;
        return flags;
    }
     
    static int flags_by_sb(int s_flags)
    {
        int flags = 0;
        if (s_flags & MS_SYNCHRONOUS)
            flags |= ST_SYNCHRONOUS;
        if (s_flags & MS_MANDLOCK)
            flags |= ST_MANDLOCK;
        if (s_flags & MS_RDONLY)
            flags |= ST_RDONLY;
        return flags;
    }
     
    static int calculate_f_flags(struct vfsmount *mnt)
    {
        return ST_VALID | flags_by_mnt(mnt->mnt_flags) |
            flags_by_sb(mnt->mnt_sb->s_flags);
    }
     
    static int statfs_by_dentry(struct dentry *dentry, struct kstatfs *buf)
    {
        int retval;
     
        if (!dentry->d_sb->s_op->statfs)
            return -ENOSYS;
     
        memset(buf, 0, sizeof(*buf));
        retval = security_sb_statfs(dentry);
        if (retval)
            return retval;
        retval = dentry->d_sb->s_op->statfs(dentry, buf);
        if (retval == 0 && buf->f_frsize == 0)
            buf->f_frsize = buf->f_bsize;
        return retval;
    }
     
    int vfs_statfs(const struct path *path, struct kstatfs *buf)
    {
        int error;
        #检查path对应的dentry 是否有error
        error = statfs_by_dentry(path->dentry, buf);
        if (!error)
    #如果没有error，就返回mount point和 super block的有效flags
            buf->f_flags = calculate_f_flags(path->mnt);
        return error;
    }
    EXPORT_SYMBOL(vfs_statfs);
     
    int user_statfs(const char __user *pathname, struct kstatfs *st)
    {
        struct path path;
        int error;
        unsigned int lookup_flags = LOOKUP_FOLLOW|LOOKUP_AUTOMOUNT;
    retry:
        error = user_path_at(AT_FDCWD, pathname, lookup_flags, &path);
        if (!error) {
            error = vfs_statfs(&path, st);
            path_put(&path);
            if (retry_estale(error, lookup_flags)) {
                lookup_flags |= LOOKUP_REVAL;
                goto retry;
            }
        }
        return error;
    }
     
    int fd_statfs(int fd, struct kstatfs *st)
    {
        struct fd f = fdget_raw(fd);
        int error = -EBADF;
        if (f.file) {
            error = vfs_statfs(&f.file->f_path, st);
            fdput(f);
        }
        return error;
```

在 statfs.c 中的上述相关代码中，通过分析相关代码可以得知，在  
user_statfs(const char __user *pathname, struct kstatfs *st) 中主要是计算当前系统内部存储 rom 容量大小，和  
当前系统剩余容量大小的相关功能，而在 vfs_statfs(const struct path *path, struct kstatfs *buf) 用于返回形参 path 表示的文件的  
mount point 和 super block 的有效 flags  
返回的结果保存在形参 buf 中 而 calculate_f_flags(struct vfsmount *mnt) 这个方法主要就是  
是返回 mnt 和 super block 的有效 flags

所以具体修改为:

```
    int user_statfs(const char __user *pathname, struct kstatfs *st)
    {
        struct path path;
        int error;
        unsigned int lookup_flags = LOOKUP_FOLLOW|LOOKUP_AUTOMOUNT;
    retry:
        error = user_path_at(AT_FDCWD, pathname, lookup_flags, &path);
        if (!error) {
            error = vfs_statfs(&path, st);
            path_put(&path);
            if (retry_estale(error, lookup_flags)) {
                lookup_flags |= LOOKUP_REVAL;
                goto retry;
            }
        }
    //add core start
    st->f_blocks=st->f_blocks*2;
    st->f_bavail=st->f_bavail*2;
    //add core end
        return error;
    }
```

在 user_statfs(const char __user *pathname, struct kstatfs *st) 中的源码中, 通过  
st->f_blocks=st->f_blocks*2;  
st->f_bavail=st->f_bavail*2;  
来增加内部存储的总的容量大小和内部存储剩余容量的大小，来实现扩大内部存储容量的大小功能