>? bds, 即 `BlockDriverState` 的简称

在QEMU中，`BlockDriverState（BDS）` 是块设备后端资源及其运行时状态的抽象，代表一个虚拟块设备实例的完整上下文，管理该设备的IO操作、配置、资源（如文件句柄）、以及可能的层次结构关系

## 一、BDS 关键字段
```clike
struct BlockDriverState {
    BlockDriver *drv;          // 驱动实现（如qcow2驱动）
    void *opaque;             // 驱动私有数据（如qcow2的元数据缓存）
    int64_t total_sectors;    // 设备总扇区数
    bool read_only;           // 是否只读
    BdrvChild *backing;       // 指向基础镜像的BdrvChild
    BdrvChild *file;          // 直接数据存储节点（如实际文件）
    QDict *options;           // 配置选项（缓存模式、discard设置等）
    // ... 其他字段（统计信息、加密上下文、子节点列表等）
};
```

## 二、BDS 的创建

`BDS` 提供的核心创建函数为 `bdrv_open()`

```clike
BlockDriverState * no_coroutine_fn
bdrv_open(const char *filename, const char *reference, QDict *options,
          int flags, Error **errp);
```

其对外提供的创建有两个接口：

```clike
qmp_blockdev_add();
drive_new();
```

其中 `drive_new()` 是传统`legacy`的创建 BDS的用法，其通过 `blk = blockdev_init(filename, bs_opts, errp)` 创建 `blk`, 匿名创建 `BDS`；

而 `qmp_blockdev_add()` 是通过  `bds_tree_init(QDict *bs_opts, Error **errp)` 创建 `BDS `，每个BDS包含自己的 `node-name`，即现代方式的 block layer 创建方式。此种创建方式下，`blk` 对象是在 `qmp_device_add() / qdev_device_add_from_qdict()` 中创建的，即通过 `set_drive` 创建 blk，此时 blk 是匿名的，即 `blk->name` 为 NULL

## 三、BDS 相关 API

### 1、全局链表 

```clike
/* Protected by BQL */
static QTAILQ_HEAD(, BlockDriverState) graph_bdrv_states =
    QTAILQ_HEAD_INITIALIZER(graph_bdrv_states);

/* Protected by BQL */
static QTAILQ_HEAD(, BlockDriverState) all_bdrv_states =
    QTAILQ_HEAD_INITIALIZER(all_bdrv_states);
```

通过 `qmp_blockdev_add()` 创建的具名 `node` 才会在 `graph_bdrv_states` 中。
而所有的 `bdrv` 对象都会在 `all_bdrv_states` 对象中。

>? 与传统 block 层相比，现代的 block layer 称 bs 为 `bdrv_node`, 而称传统的 bs 为 `bdrv_state`

### 2、 `bdrv_open_inherit()` 创建 `bs` 对象

`bdrv_open_inherit()` 是创建 bds 的主要方式，我们这里解析此函数的详细流程。

首先是 `qmp_blockdev_add(options)` 创建 bdrv_node 的流程，其最终调用 `bdrv_open_inherit` 时仅传递 options 和 flags

```clike
qmp_blockdev_add(options)
  bds_tree_init(options)
    bdrv_open(options, )
      bdrv_open_inherit(options, parse_filename:true)
```

此磁盘创建时 `blockdev` 的命令行如下

```bash
-blockdev '{"driver":"file","filename":"/workspace/Imgs/openEuler.qcow2","aio":"native","node-name":"libvirt-1-storage","auto-read-only":true,"discard":"unmap","cache":{"direct":true,"no-flush":false}}' \
-blockdev '{"node-name":"libvirt-1-format","read-only":false,"discard":"unmap","cache":{"direct":true,"no-flush":false},"driver":"qcow2","file":"libvirt-1-storage","backing":null}' \
```

其生成的 options : dict如下：

```json
{
    "cache.no-flush": false,
    "node-name": "libvirt-1-storage",
    "driver": "file",
    "filename": "/workspace/Imgs/openEuler.qcow2",
    "auto-read-only": true,
    "read-only": "off",
    "aio": "native",
    "cache.direct": true,
    "discard": "unmap"
}

{
    "cache.no-flush": false,
    "backing": null,
    "node-name": "libvirt-1-format",
    "driver": "qcow2",
    "read-only": false,
    "cache.direct": true,
    "discard": "unmap",
    "file": "libvirt-1-storage"
}
```

#### 2.1  storage 层的 `bdrv_open_inherit()`
```clike
第一个 bs, driver : file 的创建过程，其 options 如下，
其余参数如下：filename:(null), reference:(null), flags:0, parent:(nil), child_class:(nil), child_role:0, parse_filename:1
{
    "cache.no-flush": false,
    "node-name": "libvirt-1-storage",
    "driver": "file",
    "filename": "/workspace/Imgs/openEuler.qcow2",
    "auto-read-only": true,
    "read-only": "off",
    "aio": "native",
    "cache.direct": true,
    "discard": "unmap"
}

static BlockDriverState * no_coroutine_fn
bdrv_open_inherit(const char *filename, const char *reference, QDict *options,
                  int flags, BlockDriverState *parent,
                  const BdrvChildClass *child_class, BdrvChildRole child_role,
                  bool parse_filename, Error **errp)
{
    /* 1、创建 BS，最后返回的就是 此 */
    BlockDriverState *bs = bdrv_new();
    
    /* 2、filename 有时也是 options 的一部分，这种情况下 filename 是以 "json:" 为前缀，
     * 则将 json 内容 合并到 options 中，我们这里 filename 为 NULL, 因此这一步什么都不会做
     */
    if (parse_filename) {
        parse_json_protocol(options, &filename, &local_err);
        if (local_err) {
            goto fail;
        }
    }
	
    /* 3、复制 options 到 bs */
    bs->explicit_options = qdict_clone_shallow(options);
	
    /* 4、填充参数，这里filename, parse_filename,均为 false */
    ret = bdrv_fill_options(&options, filename, &flags, parse_filename,
                            &local_err);

    /* 5、设置 bs 的读写权限，因为我们这里 "read-only": "off"， 所以 flags 允许读写 */
    if (g_strcmp0(qdict_get_try_str(options, BDRV_OPT_READ_ONLY), "on") &&
        !qdict_get_try_bool(options, BDRV_OPT_READ_ONLY, false)) {
        flags |= (BDRV_O_RDWR | BDRV_O_ALLOW_RDWR);
    } else {
        flags &= ~BDRV_O_RDWR;
    }

    bs->open_flags = flags;
    bs->options = options;
    /* 6、浅拷贝 options，当前 options 中也没有嵌套 dict / list */
    options = qdict_clone_shallow(options);

    /* 7、获取 drv, 这里 drv 为 BlockDriver bdrv_file */
    drvname = qdict_get_try_str(options, "driver");
    if (drvname) {
        drv = bdrv_find_format(drvname);
        if (!drv) {
            error_setg(errp, "Unknown driver: '%s'", drvname);
            goto fail;
        }
    }

    assert(drvname || !(flags & BDRV_O_PROTOCOL));

    /* 8、如果 backing 字段存在，则设置  flags |= BDRV_O_NO_BACKING，并且从 options 中移除 backing  
     * 我们现在没有快照，因此暂不考虑
     */
    backing = qdict_get_try_str(options, "backing");
    if (qobject_to(QNull, qdict_get(options, "backing")) != NULL ||
        (backing && *backing == '\0'))
    {
        ...
    }

    /* 9、不是 BDRV_O_PROTOCOL 的情况，我们现在的 bds 是 file_bds
     * 因此 flags & BDRV_O_PROTOCOL 为 true,暂不用考虑，在 open format-layer{qcow2}时，就会走进去了 
     * flags 的更新是在 [4] bdrv_fill_options 中完成的 */
    if ((flags & BDRV_O_PROTOCOL) == 0) {
        ...
    }

    /* 10、在 open format layer 时才可能会执行，这里暂时不考虑 */
    bs->probed = !drv;
    if (!drv && file) {
        ...
    }

    /* 11、Open the image，主要就是解析 bs 层的一些参数，更新 drv 等 */
    ret = bdrv_open_common(bs, file, options, &local_err);


    /* 12、如果指定了 backing, 则为 bs 设置 bs->backing */
    if ((flags & BDRV_O_NO_BACKING) == 0) {
        ret = bdrv_open_backing_file(bs, options, "backing", &local_err);
        if (ret < 0) {
            goto close_and_fail;
        }
    }

    /* 13、bs->options 已经拷贝了一份，检查是否有多于的 option，Check if any unknown options were used */
    if (qdict_size(options) != 0) {
        const QDictEntry *entry = qdict_first(options);
        if (flags & BDRV_O_PROTOCOL) {
            error_setg(errp, "Block protocol '%s' doesn't support the option "
                       "'%s'", drv->format_name, entry->key);
        } else {
            error_setg(errp,
                       "Block format '%s' does not support the option '%s'",
                       drv->format_name, entry->key);
        }

        goto close_and_fail;
    }
	
    /* 14、遍历父节点的 bs，通知他们子节点的 bs 发生了变化，一般和 cdrom 光驱有关 */
    bdrv_parent_cb_change_media(bs, true);

    return bs;
```
上述为 `bdrv_open_inherit` 的主要流程，但是有两个函数没有仔细分析，下述进行剖析：

##### 2.1.1 `bdrv_fill_options`

`bdrv_fill_options` 看着好像是要填充很多 options 的样子，但其主要目的是解析 file 相关属性。

函数参数为 `filename:(null), flags:0, allow_parse_filename:1`

```clike
static int bdrv_fill_options(QDict **options, const char *filename,
                             int *flags, bool allow_parse_filename,
                             Error **errp)
{
    const char *drvname;
    bool protocol = *flags & BDRV_O_PROTOCOL;
    bool parse_filename = false;

    /* 1、判断当前 bs 是否为 protocol 层，也就是是否为 storage 层
     * 主要是根据 drv->protocol_name 来判断的，如果有，就是 protocol 层
     */
    drvname = qdict_get_try_str(*options, "driver");
    if (drvname) {
        drv = bdrv_find_format(drvname);
        protocol = drv->protocol_name;
    }
    if (protocol) {
        *flags |= BDRV_O_PROTOCOL;
    } else {
        *flags &= ~BDRV_O_PROTOCOL;
    }

    /* 2、更新 cache 模式
     * 主要是为 options 更新 cache 的默认值
     */
    update_options_from_flags(*options, *flags);

    filename = qdict_get_try_str(*options, "filename");

    if (!drvname && protocol) {
        /* 3、在 filename 中找到 drive 并填充到 options
         * 例如 /dev/vda 与 /Images/test.img 可以判断出前者时 host_device
         */
        if (filename) {
            drv = bdrv_find_protocol(filename, parse_filename, errp);
            if (!drv) {
                return -EINVAL;
            }

            drvname = drv->format_name;
            qdict_put_str(*options, "driver", drvname);
        } else {
            error_setg(errp, "Must specify either driver or file");
            return -EINVAL;
        }
    }


    /* 4、Driver-specific filename parsing 
     * 有些 drv 的 filename 有特殊的含义需要进行解析
     */
    if (drv && drv->bdrv_parse_filename && parse_filename) {
        drv->bdrv_parse_filename(filename, *options, &local_err);
        if (!drv->bdrv_needs_filename) {
            qdict_del(*options, "filename");
        }
    }

    return 0;
}
```

总结 `bdrv_fill_options` 主要做以下事情

- 1、解析是否是存储层，也就是 `drv->protocol_name` 是否存在，当前`protocol` 的 `drv` 为

  - `bdrv_blkdebug`, `bdrv_blkverify`
  - `bdrv_http` ,`bdrv_ftp`,  `bdrv_ssh`, `bdrv_rbd`
  - `bdrv_file`， `bdrv_host_device`， `bdrv_host_cdrom`
  - `bdrv_iscsi`, `bdrv_nbd`， `bdrv_nbd_tcp`， `bdrv_nbd_unix`， `bdrv_nfs`， `bdrv_nvme` 等等

- 2、解析文件 filename， 用户没有传入 drive，并且是一个 `protocol`时，可以根据 `.bdrv_probe_device` (探针) 与 filename 来判断 `driver`

  支持 `.bdrv_probe_device`  的有 `bdrv_host_device`, `cdrom_probe_device`， 也就是说这两种

##### 2.1.2 `bdrv_open_common`

打开 `bdrv bs` 的通用部分，这一部分中，应该要耗尽 `options` 中的所有 opt，另外在 `bdrv_open_common` 中 调用了 `bdrv_open_driver` 来实现 `drv` 的分配。

参数如下：bs:0x563d61e90010, file:(null)

```clike
static int bdrv_open_common(BlockDriverState *bs, BlockBackend *file,
                            QDict *options, Error **errp)
{
    /* 1、不管这是 protocol 还是 format， bs->file 初始都是 null */   
    assert(bs->file == NULL);
	
    /* 2、将 options 中的选项 move 到 opts，且按照bdrv_runtime_opts来拷贝 */
    opts = qemu_opts_create(&bdrv_runtime_opts, NULL, 0, &error_abort);
    if (!qemu_opts_absorb_qdict(opts, options, errp)) {
        ret = -EINVAL;
        goto fail_opts;
    }
	
    /* 3、更新 open_flags */
    update_flags_from_options(&bs->open_flags, opts);

    driver_name = qemu_opt_get(opts, "driver");
    drv = bdrv_find_format(driver_name);
    assert(drv != NULL);
	
    /* 4、共享卷只能用于只读场景 */
    bs->force_share = qemu_opt_get_bool(opts, BDRV_OPT_FORCE_SHARE, false);

    if (bs->force_share && (bs->open_flags & BDRV_O_RDWR)) {
        error_setg(errp,
                   BDRV_OPT_FORCE_SHARE
                   "=on can only be used with read-only images");
        ret = -EINVAL;
        goto fail_opts;
    }
	
    /* 5、获取 filename */
    if (file != NULL) {
        filename = blk_bs(file)->filename;
    } else {
        filename = qdict_get_try_str(options, "filename");
    }
	
    /* 6、设置 read_only flag */
    ro = bdrv_is_read_only(bs);
  	
    /* 7、设置 COPY_ON_READ flag */
    /* bdrv_new() and bdrv_close() make it so */
    assert(qatomic_read(&bs->copy_on_read) == 0);
    if (bs->open_flags & BDRV_O_COPY_ON_READ) {
        if (!ro) {
            bdrv_enable_copy_on_read(bs);
        } else {
            error_setg(errp, "Can't use copy-on-read on read-only device");
            ret = -EINVAL;
            goto fail_opts;
        }
    }
	
    /* 8、支持空间回收，则为 open_flags 设置 BDRV_O_UNMAP */
    discard = qemu_opt_get(opts, BDRV_OPT_DISCARD);
    if (discard != NULL) {
        if (bdrv_parse_discard_flags(discard, &bs->open_flags) != 0) {
    }
	
    /* 9、 bs 设置 detect_zeroes */
    bs->detect_zeroes =
        bdrv_parse_detect_zeroes(opts, bs->open_flags, &local_err);

    if (filename != NULL) {
        pstrcpy(bs->filename, sizeof(bs->filename), filename);
    } else {
        bs->filename[0] = '\0';
    }
    pstrcpy(bs->exact_filename, sizeof(bs->exact_filename), bs->filename);

    /* 10、Open the image, either directly or using a protocol 
     * [1] 为 bs 更新 node_name，如果 options 中没有 node-name，则分配一个\
     * [2] 为 bs 设置 drv, 并申请 drv 的私有空间 bs->opaque(名字有点奇怪哦)，并调用 drv->bdrv_open
     * [3] 更新 bs->total_sectors
     * [4] 执行 drv->bdrv_drain_begin(bs)
     */
    open_flags = bdrv_open_flags(bs, bs->open_flags);
    node_name = qemu_opt_get(opts, "node-name");
    ret = bdrv_open_driver(bs, drv, node_name, options, open_flags, errp);
 
    qemu_opts_del(opts);
    return 0;
}
```

#### 2.2  format 层的 `bdrv_open_inherit()`

和 storage 不同， format 层的 bs 初始化时，需要用到 storage 的对象，且需要创建 child 节点来链接 bs.

>? bs 创建时是独立的，当需要链接时才会需要 child 对象，一个 bs 可以是多个节点的 child.

```c
/* bdrv_open_inherit 的参数如下 
 * filename:(null), reference:(null), flags:0, parent:(nil), child_class:(nil), child_role:0, parse_filename:1
 * {
 *   "cache.no-flush": false,
 *   "backing": null,
 *   "node-name": "libvirt-1-format",
 *   "driver": "qcow2",
 *   "read-only": false,
 *   "cache.direct": true,
 *   "discard": "unmap",
 *   "file": "libvirt-1-storage"
 * }
 */

static BlockDriverState * no_coroutine_fn
bdrv_open_inherit(const char *filename, const char *reference, QDict *options,
                  int flags, BlockDriverState *parent,
                  const BdrvChildClass *child_class, BdrvChildRole child_role,
                  bool parse_filename, Error **errp)
{
    /* 1、创建 bs */
    bs = bdrv_new();

    /* 2、填充 options, 这里填充后 BDRV_O_PROTOCOL 为 0 */
    ret = bdrv_fill_options(&options, filename, &flags, parse_filename,
    
    /* 3、没有快照，走不进去 */
    if (flags & BDRV_O_SNAPSHOT) {
    }

    bs->open_flags = flags;
    bs->options = options;
    options = qdict_clone_shallow(options);
	
    /* 有 backing, 但是为 NULL，因此不会走进去 */
    /* See cautionary note on accessing @options above */
    backing = qdict_get_try_str(options, "backing");
    if (qobject_to(QNull, qdict_get(options, "backing")) != NULL ||
        (backing && *backing == '\0'))
    {
    }

    /* 4、open 一个 file，这里会进入 */
    if ((flags & BDRV_O_PROTOCOL) == 0) {
        BlockDriverState *file_bs;
		/* 这里重新打开 file 对象，并创建了 child */
        file_bs = bdrv_open_child_bs(filename, options, "file", bs,
                                     &child_of_bds, BDRV_CHILD_IMAGE,
                                     true, true, &local_err);
        if (file_bs != NULL) {
            /* Not requesting BLK_PERM_CONSISTENT_READ because we're only
             * looking at the header to guess the image format. This works even
             * in cases where a guest would not see a consistent state. */
            AioContext *ctx = bdrv_get_aio_context(file_bs);
            file = blk_new(ctx, 0, BLK_PERM_ALL);
            blk_insert_bs(file, file_bs, &local_err);
            bdrv_unref(file_bs);
            /* 此时 options 中添加了 file: blk */
            qdict_put_str(options, "file", bdrv_get_node_name(file_bs));
        }
    }

    /* 5、为 bs 配置 qcow2 drv，但是这次 file 不为 NULL */
    ret = bdrv_open_common(bs, file, options, &local_err);

    if (file) {
        blk_unref(file);
        file = NULL;
    }

    /* 6、当前format没有快照，因此无需关注 */
    if ((flags & BDRV_O_NO_BACKING) == 0) {
        ret = bdrv_open_backing_file(bs, options, "backing", &local_err);
        if (ret < 0) {
            goto close_and_fail;
        }
    }

    /* Check if any unknown options were used */
    if (qdict_size(options) != 0) {
        const QDictEntry *entry = qdict_first(options);
        if (flags & BDRV_O_PROTOCOL) {
            error_setg(errp, "Block protocol '%s' doesn't support the option "
                       "'%s'", drv->format_name, entry->key);
        } else {
            error_setg(errp,
                       "Block format '%s' does not support the option '%s'",
                       drv->format_name, entry->key);
        }

        goto close_and_fail;
    }

    /*7、遍历父节点的 bs，通知他们子节点的 bs 发生了变化，一般和 cdrom 光驱有关 */
    bdrv_parent_cb_change_media(bs, true);

    return bs;
}
```

##### 2.2.1 `bdrv_open_child_bs`

在创建 format layer bs 时，再次打开 `file_bs`，并返回 file_bs

```c

/*
 * 顾名思义，为 bs 打开/创建 child_bs
 * @filename 文件名
 * @options 当前 bs(也就是 parent bs ) 的创建选项
 * @bdref_key child_bs 的 key，一般都是 "file"
 * @parent 当前 bs
 * @child_class child 的 class
 * @child_role child 的 role
 */
static BlockDriverState *
bdrv_open_child_bs(const char *filename, QDict *options, const char *bdref_key,
                   BlockDriverState *parent, const BdrvChildClass *child_class,
                   BdrvChildRole child_role, bool allow_none,
                   bool parse_filename, Error **errp)
{
    /* [1] 找到 child_bs 的 node_name */
    reference = qdict_get_try_str(options, bdref_key);
    
    /* [2] 为 child_bs 完善属性，继续调用 bdrv_open_inherit，可参考 2.4 reference 模式下的 bdrv_open_inherit
     * reference 为 true, 直接返回 reference 对应的 bs(也就是通过 node_name查找)
     */
    bs = bdrv_open_inherit(filename, reference, image_options, 0,
                           parent, child_class, child_role, parse_filename,
                           errp);
    return bs;
}
```

##### 2.2.2 为 format_bs 创建 child并设置 child

对应 format bs，最重要的一点就是配置 `bs->file: Child`， 对于 qcow2，其堆栈如下：

```c
bdrv_child_cb_attach()
bdrv_replace_child_noperm()
bdrv_attach_child_common()
bdrv_attach_child_noperm();
bdrv_attach_child();
bdrv_open_child_common();
bdrv_open_file_child()
qcow2_open()
bdrv_open_driver()
bdrv_open_common()
bdrv_open_inherit()
bdrv_open()
bds_tree_init()
qmp_blockdev_add()
```

对应 qcow2 , 其 file child 节点的 child_class 统一为 `child_of_bds`, child_role 如下

```c
role = parent->drv->is_filter ?
        (BDRV_CHILD_FILTERED | BDRV_CHILD_PRIMARY) : BDRV_CHILD_IMAGE;
```



#### 2.3  快照模式下的 `bdrv_open_inherit()`

#### 2.4 reference 模式下的 `bdrv_open_inherit`

