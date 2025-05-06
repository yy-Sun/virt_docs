## block layer

block layer 的 qemu 存储子系统的重要概念。

### 一、`-drive`和`-blockdev`这两个参数的区别

`-drive`是QEMU早期用来定义块设备的参数，它会隐式地创建BlockBackend和BlockDriverState。而`-blockdev`是较新的参数，允许用户显式地配置BlockDriverState节点，从而更灵活地控制块设备的结构和拓扑。

#### 1 设计背景与架构

| **特性**     | **`-drive`**                                  | **`-blockdev`**                        |
| :----------- | :-------------------------------------------- | :------------------------------------- |
| **架构模型** | 传统模型（Legacy）                            | 现代模型（Blockdev）                   |
| **核心对象** | 隐式管理 `BlockBackend` 和 `BlockDriverState` | 显式定义 `BlockDriverState`（BDS）节点 |
| **目标**     | 简化配置，自动创建必要对象                    | 提供细粒度控制，支持复杂存储拓扑       |

#### 2 配置方式与语法
##### 2.1 `-drive`
作用：快速定义虚拟机可见的块设备（如磁盘），自动创建关联的 `BlockBackend` 和 `BlockDriverState`。
###### 语法示例：

```bash
qemu-system-x86_64 \
  -drive file=disk.qcow2,format=qcow2,id=mydisk,if=virtio
```
###### 参数说明：

- file：磁盘镜像路径。

- format：镜像格式（如 qcow2）。

- id：设备标识符（用于 QMP 命令操作）。

- if：接口类型（如 virtio、ide）。

###### 隐式行为：
- 自动创建 BlockBackend（关联虚拟机设备，如 virtio-blk）。
- 自动创建 BlockDriverState（管理磁盘镜像）。

##### 2.2 `-blockdev`

显式定义 `BlockDriverState` 节点，构建存储设备的底层拓扑（如镜像链、过滤器驱动）。

###### 语法示例：

```bash
qemu-system-x86_64 \
  -blockdev driver=file,node-name=file0,filename=disk.raw \
  -blockdev driver=qcow2,node-name=qcow0,file=file0
```

###### 参数说明：

- `driver`：驱动类型（如 `file`、`qcow2`）。
- `node-name`：自定义节点名称（必填，用于后续引用）。
- `file`：指定父节点（如 `file0` 是 `qcow0` 的底层存储）。

###### **显式行为**：

- 需手动定义所有节点（如文件层、格式层）。
- 需通过其他参数（如 `-device`）将节点关联到虚拟机设备。

####  3. 核心区别

| **区别点**         | **`-drive`**                                 | **`-blockdev`**                                              |
| :----------------- | :------------------------------------------- | :----------------------------------------------------------- |
| **对象管理**       | 隐式创建 `BlockBackend` + `BlockDriverState` | 显式定义 `BlockDriverState`，可选关联 `BlockBackend`         |
| **存储拓扑**       | 仅支持单层结构（如单个磁盘镜像）             | 支持多层嵌套（如 `file` → `qcow2` → 加密过滤器）             |
| **灵活性**         | 低（简单场景）                               | 高（支持复杂链式结构、动态修改拓扑）                         |
| **虚拟机设备关联** | 自动绑定到设备（如 `-drive if=virtio`）      | 需通过 `-device` 手动关联（如 `-device virtio-blk-pci,drive=qcow0`） |
| **版本兼容性**     | 所有版本                                     | QEMU 2.9+                                                    |

>? 推荐逐步将旧版 `-drive` 替换为 `-blockdev` + `-device`，以利用新特性。