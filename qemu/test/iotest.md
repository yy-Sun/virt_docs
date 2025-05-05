# iotests

位于`tests/qemu-iotests`目录下的iotests是QEMU用于测试块设备相关功能的核心测试框架。相较于`make check`测试，该框架属于更高层级的测试，其中99%的代码采用bash或Python脚本编写。测试通过"黄金输出对比"（golden output comparison）机制验证结果，测试用例均以数字编号命名。
## 一、简介
### 1.1 运行方法

1. **前置要求**
   确保QEMU已成功编译

2. **执行步骤**

   ```bash
   cd build/tests/qemu-iotests  # 进入构建目录下的测试文件夹
   ./check [参数]              # 运行测试框架
   ```

### 1.2 参数配置

- **默认配置**
  使用`raw`镜像格式 + `file`协议，自动跳过当前环境不支持的测试

- **指定镜像格式**

  ```bash
  ./check -qcow2  # 使用qcow2格式运行
  ```

- **指定传输协议**

  ```bash
  ./check -nbd    # 使用NBD协议运行
  ```

- **选择特定测试用例**

  ```bash
  ./check -qcow2 001 030 153  # 以qcow2格式运行指定编号测试
  ```

- **缓存模式测试**
  通过`-c`参数可指定缓存模式，有助于发现特定缓存模式下的缺陷：

  ```bash
  ./check -c writethrough  # 使用透写缓存模式
  ```

### 1.3 获取帮助

```
./check -h  # 查看完整的参数说明
```

## 二、编写新测试用例指南

当您对块设备层进行任何修改时，都应当考虑编写测试用例。通常推荐选择iotest作为测试方案。由于现有测试用例已覆盖大量场景，扩展现有测试往往比新建测试更高效。（注意：目前没有绝对可靠的方法能从数百个测试中快速定位相关用例，建议使用`git grep`进行检索）

### 2.1 文件结构规范

每个iotest测试用例通常包含两个文件：

- **可执行脚本**：生成标准输出(stdout)和错误输出(stderr)

- **预期输出文件**：采用相同数字编号

  ```
  055      # 测试脚本
  055.out  # 预期输出
  ```

特殊情况下需增加变体文件：

- 缓存模式差异：添加`.out.nocache`文件
- 镜像格式差异：添加格式后缀如`178.out.qcow2`和`178.out.raw`

### 2.2 测试脚本编写方案

虽然没有严格限制，但推荐通过复制修改现有用例来创建新测试。常用编写方式包括：

1. **Bash脚本**

   - 使用测试框架提供的环境变量

   - 引入`common.*`系列库函数

   - 示例结构：

     ```bash
     #!/usr/bin/env bash
     source ./common.rc
     TEST_IMAGE="null-co://"
     ...测试逻辑...
     ```

2. **Python unittest**

   - 继承`iotests.QMPTestCase`类

   - 调用`iotests.main()`方法

   - 特点：输出信息精简，调试难度较高

   - 示例框架：

     ```python
     import iotests
     class TestCase(iotests.QMPTestCase):
         def test_feature(self):
             ...测试方法...
     iotests.main()
     ```

3. **简易Python脚本**

   - 混合方案：调用iotests库但不继承TestCase

   - 不使用unittest执行框架

   - 示例：

     ```python
     from iotests import qemu_img_create
     ...直接编写测试逻辑...
     ```

### 2.3 语言选择建议

- **自由选择**：Bash和Python对QEMU调用的支持度相当
- **Python专项提示**：
  - 强烈建议兼容Python 3语法
  - 推荐使用伪块设备`null-co://`（无需镜像清理）
  - 避免使用系统级设备如`/dev/null`（可能触发文件锁冲突）
  - 必须使用时需添加`locking=off`选项

### 2.4 镜像管理最佳实践

- 优先使用框架提供的镜像管理工具

- 非必要情况不使用真实I/O设备

- 协议无关测试推荐伪驱动方案：


  ```python
  TEST_IMAGE = "null-co://"  # 零配置的虚拟块设备
  ```

## 三、调试测试用例

当测试用例执行失败时，可通过以下`check`脚本参数进行调试：

### 3.1 核心调试选项

1. **GDB调试模式 (`-gdb`)**

   - 为每个QEMU进程启动`gdbserver`

   - 监听配置通过`$GDB_OPTIONS`环境变量设置（默认：`localhost:12345`）

   - 连接示例：

     ```bash
     gdb -iex "target remote localhost:12345"
     ```

   - 注意：未使用`-gdb`选项时，`$GDB_OPTIONS`将被忽略

2. **Valgrind内存检测 (`-valgrind`)**

   - 自动附加valgrind实例到QEMU进程

   - 检测到问题时生成日志文件：
     `$TEST_DIR/<valgrind_pid>.valgrind`

   - 实际执行的命令结构：

     ```bash
     valgrind --log-file=$TEST_DIR/<valgrind_pid>.valgrind \
              --error-exitcode=99 $QEMU ...
     ```

3. **调试日志增强 (`-d`)**

   - 提升日志详细级别
   - 额外显示QMP命令与响应等调试信息

4. **实时输出模式 (`-p`)**

   - 将QEMU的stdout/stderr直接输出到测试控制台
   - 替代默认行为（保存到`$TEST_DIR/qemu-machine-<random_string>.log`）

### 3.2 使用示例组合

```bash
# 带内存检测的详细日志模式
./check -valgrind -d 055

# 启用GDB调试并打印实时输出
GDB_OPTIONS="localhost:5555" ./check -gdb -p 128
```

## 四、测试用例分组规范

测试用例可以通过以下两种方式定义所属分组：

### 4.1 源码注释定义法

在测试文件第二行（紧接shebang行之后）以`# group:`注释声明分组：

```python
#!/usr/bin/env python3
# group: auto quick  # 声明属于auto和quick分组
#
...测试代码...
```

### 4.2 本地分组文件定义法（仅限下游）

通过创建`tests/qemu-iotests/group.local`文件定义（该文件不应提交到上游）。文件格式示例：

```plaintext
# 公司流程专用分组
#
# ci    - 构建时运行的测试
# down  - 下游专用测试（不上传上游）
#
# 每行格式：
# 测试编号 分组1 [分组2]...

013 ci          # 将013号测试加入ci分组
210 disabled    # 禁用210号测试
215 disabled    # 禁用215号测试
our-ugly-workaround-test down ci  # 自定义测试加入多个分组
```

### 4.3 特殊分组说明

- **quick**
  测试应在数秒内完成执行
- **auto**
  该分组测试具有严格限制：
  - 必须能在`make check`中运行
  - 兼容所有QEMU二进制文件（包括非x86架构）
  - 适配所有编译配置（可选功能未编译时应跳过而非失败）
  - 至少支持qcow2镜像格式
  - 兼容各类主机文件系统和用户权限（如nobody/root账户）
  - 严格控制内存和磁盘占用（避免CI流水线失败）
- **disabled**
  标记为禁用的测试将被`check`脚本忽略
