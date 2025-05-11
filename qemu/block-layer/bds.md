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

