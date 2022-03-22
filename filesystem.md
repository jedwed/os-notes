<h1>File Systems</h1>

---

- [OS/161 Virtual File System (VFS)](#os161-virtual-file-system-vfs)

---

## OS/161 Virtual File System (VFS)
- Represents interface to a mounted filesystem
```c++
struct fs {
    int (*fs_sync)(struct fs *);                /* Force the filesystem to flush/save its content to disk */
    const char *(*fs_getvolname)(struct fs *);  /* Retrieve the volume name */
    struct vnode *(*fs_getroot)(struct fs *);   /* Retrieve the vnode assosciated with the root of the filesystem */
    int (*fs_unmount)(struct fs *);             /* Unmount the file system (Note: mount called vis function ptr passed to vfs_mount) */
    void *fs_data;                              /* Private file system specific data */
};

struct vnode {
    int vn_refcount;                /* Count the number of 'references' to this node */
    struct spinlock vn_countlock;   /* Look for mutual exclusive access to counts */
    struct fs *vn_fs;               /* Pointer to FS containing the node */
    void *vn_data;                  /* Pointer to FS specific vnode data (eg. in-memory copy of inode) */
    const struct vnode_ops *vn_ops; /* Array of pointers to functions operating on vnodes */
};

struct vnode_ops {
    unsigned long vop_magic; /* should always be VOP_MAGIC */

    int (*vop_eachopen) (struct vnode *object, int flags_from_open);
    int (*vop_reclaim) (struct vnode *vnode);

    int (*vop_read) (struct vnode *file, struct uio *uio);
    int (*vop_readlink) (struct vnode *link, struct uio *uio);
    int (*vop_getdirentry) (struct vnode *dir, struct uio *uio);
    int (*vop_write) (struct vnode *file, struct uio *uio);
    int (*vop_ioctl) (struct vnode *object, int op, userptr_t data);
    int (*vop_stat) (struct vnode *object, struct stat *statbuf);
    int (*vop_gettype) (struct vnode *object, int *result);
    int (*vop_isseekable) (struct vnode *object, off_t pos);
    int (*vop_fsync) (struct vnode *object);
    int (*vop_mmap) (struct vnode *file /* add stuff */);
    int (*vop_truncate) (struct vnode *file, off_t len);
    int (*vop_namefile) (struct vnode *file, struct uio *uio);

    int (*vop_creat) (struct vnode *dir, const char *name, int excl, struct vnode **result);
    int (*vop_symlink) (struct vnode *dir, const char *contents, const char *name);
    int (*vop_mkdir) (struct vnode *parentdir, const char *name);
    int (*vop_link) (struct vnode *dir, const char *name, struct vnode *file);
    int (*vop_remove) (struct vnode *dir, const char *name);
    int (*vop_rmdir) (struct vnode *dir, const char *name);
    int (*vop_rename) (struct vnode *vn1, const char *name1, struct vnode *vn2, const char *name2);
    int (*vop_lookup) (struct vnode *dir, char *pathname, struct vnode **result);
    int (*vop_lookparent) (struct vnode *dir, char *pathname, struct vnode **result, char *buf, size_t len);
};
```

- Note most operations are on vnodes. How do we operate on filenames?
  - **Higher level API on names that use the internal `VOP_*` functions**

```c++
int vfs_open(char *path, int openflags, mode_t mode, struct vnode **ret);
void vfs_close(struct vnode *vn);
int vfs_readlink(char *path, struct uio *data);
int vfs_symlink(const char *contents, char *path);
int vfs_mkdir(char *path);
int vfs_link(char *oldpath, char *newpath);
int vfs_remove(char *path);
int vfs_rmdir(char *path);
int vfs_rename(char *oldpath, char *newpath);
int vfs_chdir(char *path);
int vfs_getcwd(struct uio *buf);
```

- `vfs_open` returns the vnode for further operation
- int returns to indicate success

```c++
/*
* Function table for emufs
files.
*/
static const struct vnode_ops emufs_fileops = {
    VOP_MAGIC, /* mark this a
    valid vnode ops table */
    emufs_eachopen,
    emufs_reclaim,
    emufs_read,
    NOTDIR, /* readlink */
    NOTDIR, /* getdirentry */
    emufs_write,
    emufs_ioctl,
    emufs_stat,
    emufs_file_gettype,emufs_tryseek,
    emufs_fsync,
    UNIMP, /* mmap */
    emufs_truncate,
    NOTDIR, /* namefile */
    NOTDIR, /* creat */
    NOTDIR, /* symlink */
    NOTDIR, /* mkdir */
    NOTDIR, /* link */
    NOTDIR, /* remove */
    NOTDIR, /* rmdir */
    NOTDIR, /* rename */
    NOTDIR, /* lookup */
    NOTDIR, /* lookparent */
};
```