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

### 1、全局链表 

```c
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

```c
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
```c
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

```c
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
        /* filename 可能是 file bs，这种情况下，driver 包括在 filename 中 */
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

    assert(drv || !protocol);

    /* Driver-specific filename parsing */
    if (drv && drv->bdrv_parse_filename && parse_filename) {
        drv->bdrv_parse_filename(filename, *options, &local_err);
        if (local_err) {
            error_propagate(errp, local_err);
            return -EINVAL;
        }

        if (!drv->bdrv_needs_filename) {
            qdict_del(*options, "filename");
        }
    }

    return 0;
}
```



##### 2.1.2 `bdrv_open_common`



#### 2.2  format 层的 `bdrv_open_inherit()`

#### 2.3  快照模式下的 `bdrv_open_inherit()`
