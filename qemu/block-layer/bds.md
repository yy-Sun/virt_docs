>? bds, 即 `BlockDriverState` 的简称

在QEMU中，`BlockDriverState（BDS）` 是块设备后端资源及其运行时状态的抽象，代表一个虚拟块设备实例的完整上下文，管理该设备的IO操作、配置、资源（如文件句柄）、以及可能的层次结构关系

## 一、BDS 关键字段
```c
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

```c
BlockDriverState * no_coroutine_fn
bdrv_open(const char *filename, const char *reference, QDict *options,
          int flags, Error **errp);
```

其对外提供的创建有两个接口：

```c
qmp_blockdev_add();
drive_new();
```

其中 `drive_new()` 是传统`legacy`的创建 BDS的用法，其通过 `blk = blockdev_init(filename, bs_opts, errp)` 创建 `blk`, 匿名创建 `BDS`；

而 `qmp_blockdev_add()` 是通过  `bds_tree_init(QDict *bs_opts, Error **errp)` 创建 `BDS `，每个BDS包含自己的 `node-name`，即现代方式的 block layer 创建方式。此种创建方式下，`blk` 对象是在 `qmp_device_add() / qdev_device_add_from_qdict()` 中创建的，即通过 `set_drive` 创建 blk，此时 blk 是匿名的，即 `blk->name` 为 NULL

## 三、BDS 相关 API

### 1、`graph_bdrv_states` 全局链表 API

```c
/* Protected by BQL */
static QTAILQ_HEAD(, BlockDriverState) graph_bdrv_states =
    QTAILQ_HEAD_INITIALIZER(graph_bdrv_states);

```

具有 `node-name` 的 bs 链表，在 `all_bdrv_states ` 不一定会在 `graph_bdrv_states`，通过 `qmp_blockdev_add()` 创建的具名 `node` 才会在 `graph_bdrv_states` 中。

>? 与传统 block 层相比，现代的 block layer 称 bs 为 `bdrv_node`, 而称传统的 bs 为 `bdrv_state`

#### 1.1 `bdrv_open_inherit()`

`bdrv_open_inherit()` 是创建 bds 的主要方式，我们这里解析此函数的详细流程。

首先是 `qmp_blockdev_add(options)` 创建 bdrv_node 的流程，其最终调用 `bdrv_open_inherit` 时仅传递 options 和 flags

```c
qmp_blockdev_add(options)
  bds_tree_init(options)
    bdrv_open(options, BDRV_O_INACTIVE)
      bdrv_open_inherit(options, BDRV_O_INACTIVE)
    	bs = bdrv_new();
        bdrv_fill_options(options, BDRV_O_INACTIVE)
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

```c
static BlockDriverState * no_coroutine_fn
bdrv_open_inherit(const char *filename, const char *reference, QDict *options,
                  int flags, BlockDriverState *parent,
                  const BdrvChildClass *child_class, BdrvChildRole child_role,
                  bool parse_filename, Error **errp)
{
    int ret;
    BlockBackend *file = NULL;
    BlockDriverState *bs;
    BlockDriver *drv = NULL;
    BdrvChild *child;
    const char *drvname;
    const char *backing;
    Error *local_err = NULL;
    QDict *snapshot_options = NULL;
    int snapshot_flags = 0;

    assert(!child_class || !flags);
    assert(!child_class == !parent);
    GLOBAL_STATE_CODE();
    assert(!qemu_in_coroutine());

    /* TODO We'll eventually have to take a writer lock in this function */
    GRAPH_RDLOCK_GUARD_MAINLOOP();

    if (reference) {
        bool options_non_empty = options ? qdict_size(options) : false;
        qobject_unref(options);

        if (filename || options_non_empty) {
            error_setg(errp, "Cannot reference an existing block device with "
                       "additional options or a new filename");
            return NULL;
        }

        bs = bdrv_lookup_bs(reference, reference, errp);
        if (!bs) {
            return NULL;
        }

        bdrv_ref(bs);
        return bs;
    }

    bs = bdrv_new();

    /* NULL means an empty set of options */
    if (options == NULL) {
        options = qdict_new();
    }

    /* json: syntax counts as explicit options, as if in the QDict */
    if (parse_filename) {
        parse_json_protocol(options, &filename, &local_err);
        if (local_err) {
            goto fail;
        }
    }

    bs->explicit_options = qdict_clone_shallow(options);

    if (child_class) {
        bool parent_is_format;

        if (parent->drv) {
            parent_is_format = parent->drv->is_format;
        } else {
            /*
             * parent->drv is not set yet because this node is opened for
             * (potential) format probing.  That means that @parent is going
             * to be a format node.
             */
            parent_is_format = true;
        }

        bs->inherits_from = parent;
        child_class->inherit_options(child_role, parent_is_format,
                                     &flags, options,
                                     parent->open_flags, parent->options);
    }

    ret = bdrv_fill_options(&options, filename, &flags, parse_filename,
                            &local_err);
    if (ret < 0) {
        goto fail;
    }

    /*
     * Set the BDRV_O_RDWR and BDRV_O_ALLOW_RDWR flags.
     * Caution: getting a boolean member of @options requires care.
     * When @options come from -blockdev or blockdev_add, members are
     * typed according to the QAPI schema, but when they come from
     * -drive, they're all QString.
     */
    if (g_strcmp0(qdict_get_try_str(options, BDRV_OPT_READ_ONLY), "on") &&
        !qdict_get_try_bool(options, BDRV_OPT_READ_ONLY, false)) {
        flags |= (BDRV_O_RDWR | BDRV_O_ALLOW_RDWR);
    } else {
        flags &= ~BDRV_O_RDWR;
    }

    if (flags & BDRV_O_SNAPSHOT) {
        snapshot_options = qdict_new();
        bdrv_temp_snapshot_options(&snapshot_flags, snapshot_options,
                                   flags, options);
        /* Let bdrv_backing_options() override "read-only" */
        qdict_del(options, BDRV_OPT_READ_ONLY);
        bdrv_inherited_options(BDRV_CHILD_COW, true,
                               &flags, options, flags, options);
    }

    bs->open_flags = flags;
    bs->options = options;
    options = qdict_clone_shallow(options);

    /* Find the right image format driver */
    /* See cautionary note on accessing @options above */
    drvname = qdict_get_try_str(options, "driver");
    if (drvname) {
        drv = bdrv_find_format(drvname);
        if (!drv) {
            error_setg(errp, "Unknown driver: '%s'", drvname);
            goto fail;
        }
    }

    assert(drvname || !(flags & BDRV_O_PROTOCOL));

    /* See cautionary note on accessing @options above */
    backing = qdict_get_try_str(options, "backing");
    if (qobject_to(QNull, qdict_get(options, "backing")) != NULL ||
        (backing && *backing == '\0'))
    {
        if (backing) {
            warn_report("Use of \"backing\": \"\" is deprecated; "
                        "use \"backing\": null instead");
        }
        flags |= BDRV_O_NO_BACKING;
        qdict_del(bs->explicit_options, "backing");
        qdict_del(bs->options, "backing");
        qdict_del(options, "backing");
    }

    /* Open image file without format layer. This BlockBackend is only used for
     * probing, the block drivers will do their own bdrv_open_child() for the
     * same BDS, which is why we put the node name back into options. */
    if ((flags & BDRV_O_PROTOCOL) == 0) {
        BlockDriverState *file_bs;

        file_bs = bdrv_open_child_bs(filename, options, "file", bs,
                                     &child_of_bds, BDRV_CHILD_IMAGE,
                                     true, true, &local_err);
        if (local_err) {
            goto fail;
        }
        if (file_bs != NULL) {
            /* Not requesting BLK_PERM_CONSISTENT_READ because we're only
             * looking at the header to guess the image format. This works even
             * in cases where a guest would not see a consistent state. */
            AioContext *ctx = bdrv_get_aio_context(file_bs);
            file = blk_new(ctx, 0, BLK_PERM_ALL);
            blk_insert_bs(file, file_bs, &local_err);
            bdrv_unref(file_bs);

            if (local_err) {
                goto fail;
            }

            qdict_put_str(options, "file", bdrv_get_node_name(file_bs));
        }
    }

    /* Image format probing */
    bs->probed = !drv;
    if (!drv && file) {
        ret = find_image_format(file, filename, &drv, &local_err);
        if (ret < 0) {
            goto fail;
        }
        /*
         * This option update would logically belong in bdrv_fill_options(),
         * but we first need to open bs->file for the probing to work, while
         * opening bs->file already requires the (mostly) final set of options
         * so that cache mode etc. can be inherited.
         *
         * Adding the driver later is somewhat ugly, but it's not an option
         * that would ever be inherited, so it's correct. We just need to make
         * sure to update both bs->options (which has the full effective
         * options for bs) and options (which has file.* already removed).
         */
        qdict_put_str(bs->options, "driver", drv->format_name);
        qdict_put_str(options, "driver", drv->format_name);
    } else if (!drv) {
        error_setg(errp, "Must specify either driver or file");
        goto fail;
    }

    /* BDRV_O_PROTOCOL must be set iff a protocol BDS is about to be created */
    assert(!!(flags & BDRV_O_PROTOCOL) == !!drv->protocol_name);
    /* file must be NULL if a protocol BDS is about to be created
     * (the inverse results in an error message from bdrv_open_common()) */
    assert(!(flags & BDRV_O_PROTOCOL) || !file);

    /* Open the image */
    ret = bdrv_open_common(bs, file, options, &local_err);
    if (ret < 0) {
        goto fail;
    }

    if (file) {
        blk_unref(file);
        file = NULL;
    }

    /* If there is a backing file, use it */
    if ((flags & BDRV_O_NO_BACKING) == 0) {
        ret = bdrv_open_backing_file(bs, options, "backing", &local_err);
        if (ret < 0) {
            goto close_and_fail;
        }
    }

    /* Remove all children options and references
     * from bs->options and bs->explicit_options */
    QLIST_FOREACH(child, &bs->children, next) {
        char *child_key_dot;
        child_key_dot = g_strdup_printf("%s.", child->name);
        qdict_extract_subqdict(bs->explicit_options, NULL, child_key_dot);
        qdict_extract_subqdict(bs->options, NULL, child_key_dot);
        qdict_del(bs->explicit_options, child->name);
        qdict_del(bs->options, child->name);
        g_free(child_key_dot);
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

    bdrv_parent_cb_change_media(bs, true);

    qobject_unref(options);
    options = NULL;

    /* For snapshot=on, create a temporary qcow2 overlay. bs points to the
     * temporary snapshot afterwards. */
    if (snapshot_flags) {
        BlockDriverState *snapshot_bs;
        snapshot_bs = bdrv_append_temp_snapshot(bs, snapshot_flags,
                                                snapshot_options, &local_err);
        snapshot_options = NULL;
        if (local_err) {
            goto close_and_fail;
        }
        /* We are not going to return bs but the overlay on top of it
         * (snapshot_bs); thus, we have to drop the strong reference to bs
         * (which we obtained by calling bdrv_new()). bs will not be deleted,
         * though, because the overlay still has a reference to it. */
        bdrv_unref(bs);
        bs = snapshot_bs;
    }

    return bs;

fail:
    blk_unref(file);
    qobject_unref(snapshot_options);
    qobject_unref(bs->explicit_options);
    qobject_unref(bs->options);
    qobject_unref(options);
    bs->options = NULL;
    bs->explicit_options = NULL;
    bdrv_unref(bs);
    error_propagate(errp, local_err);
    return NULL;

close_and_fail:
    bdrv_unref(bs);
    qobject_unref(snapshot_options);
    qobject_unref(options);
    error_propagate(errp, local_err);
    return NULL;
}

```



### 2、`all_bdrv_states` 全局链表相关 API
```c
/* Protected by BQL */
static QTAILQ_HEAD(, BlockDriverState) all_bdrv_states =
    QTAILQ_HEAD_INITIALIZER(all_bdrv_states);
```

所有 bs 创建时（`bdrv_new()`）都会加入该链表