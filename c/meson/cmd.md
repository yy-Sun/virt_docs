## Meson 命令

### 1、编译一个 meson 项目

大部分的 meson 项目可以通过以下方式进行编译

```bash
$ cd /path/to/source/root
$ meson setup build && cd build
$ meson compile
$ meson test
```

> 唯一需要注意的是，您需要创建一个单独的构建目录。Meson 不允许您在源代码树内构建源代码。所有构建工件都存储在构建目录中。这样，您可以同时拥有多个具有不同配置的构建树。而且生成的文件就不会被意外添加到版本控制中。
>
> 要在代码更改后重新编译，只需输入 meson compile 即可。构建命令始终相同。您可以对源代码和构建系统文件进行任意更改，Meson 会检测到这些更改并执行正确的操作。
>
> 如果您想构建优化的二进制文件，只需在运行 Meson 时使用参数 `--buildtype=debugoptimized` 即可。建议您为未优化的构建保留一个构建目录，为优化的构建保留另一个构建目录。要编译任何给定的配置，只需进入相应的构建目录并运行 meson compile 即可。
>
> Meson 会自动添加编译器标志以启用调试信息和编译器警告（例如 -g 和 -Wall）。这意味着用户无需处理它们，而可以专注于编码。

### 2、完全控制 meson 编译与安装参数

发行版打包人员通常希望完全控制所使用的构建标志。Meson 原生支持这种用例。构建和安装 Meson 项目所需的命令如下

```bash
$ cd /path/to/source/root
$ meson --prefix /usr --buildtype=plain builddir -Dc_args=... -Dcpp_args=... -Dc_link_args=... -Dcpp_link_args=...
$ meson compile -C builddir
$ meson test -C builddir
$ DESTDIR=/path/to/staging/root meson install -C builddir
```

命令行开关 `--buildtype=plain` 指示 Meson 不要在命令行中添加自己的标志。这使得打包者可以完全控制所使用的标志。

与其他构建系统非常相似。唯一的区别在于 `DESTDIR` 变量作为环境变量传递，而不是作为 meson install 的参数传递。

### 3、meson setup

这个命令是项目的**起点**。它的主要职责是执行项目的“配置”步骤，并生成相应的构建文件（如 Ninja 的 `build.ninja`）。

```bash
# 接收2个位置参数，以及其他可选参数
  builddir 
  sourcedir
```

**核心作用：**

- 创建一个构建目录（通常名为 `build` 或其他名字）。
- 分析项目根目录的 `meson.build` 文件。
- 根据系统环境、命令行参数等解析项目的依赖和选项。
- 最终在构建目录中生成构建系统文件（主要是 `build.ninja` 和 `meson-private` 中的内部状态文件）。

**何时使用？**

- **第一次构建项目时**：你从源码仓库拉取代码后，第一步就是运行 `meson setup`。
- **需要全新的构建配置时**：例如，你想从 `Release` 模式切换到 `Debug` 模式，或者彻底改变某个基础选项（如编译器），一个常见的做法是删除整个 `build` 目录，然后重新运行 `meson setup` 以确保从一个干净的状态开始。
- **创建多个不同配置的构建目录时**：例如，你可以有一个 `build-debug` 目录和一个 `build-release` 目录，分别用不同的 `meson setup` 命令初始化。

**常用选项示例：**

```bash
# 最基本用法，在当前目录创建名为 'build' 的构建目录
meson setup build

# 创建构建目录并指定安装前缀
meson setup build --prefix=/usr

# 创建构建目录并设置构建类型为调试模式
meson setup build --buildtype=debug

# 创建构建目录并传递项目选项（-D）
meson setup build -Doption1=value1 -Doption2=value2
```

### 4、meson configure

这个命令用于**交互**一个**已经通过 `meson setup` 初始化好的构建目录**。你无法在一个空的或非 Meson 构建目录中运行它。

**核心作用：**

- **查看**当前构建目录的所有配置选项。
- **修改**当前构建目录的配置选项。修改后，Meson 会自动重新生成构建文件（`build.ninja`）。

**何时使用？**

- 你想看看当前构建目录都设置了哪些选项。
- 你在开发过程中需要**调整某个选项**，而不想重新从头开始配置整个项目。这是最常见的用途, 因此 `meson configure` 的很多选项和 `meson setup` 保持一致。

**常用选项示例：**

```bash
# 进入构建目录后，查看所有当前配置的选项
cd build
meson configure

# 在不进入构建目录的情况下，查看其配置
meson configure build/

# 修改一个选项（例如，将安装前缀改为 /opt）
meson configure build/ --prefix=/opt

# 修改项目选项（-D）
meson configure build/ -Doption1=new_value

# 启用/禁用一个功能（假设有个选项叫 “docs”）
meson configure build/ -Ddocs=true
meson configure build/ -Ddocs=false
```

### 5、meson compile

**核心用途：编译源代码。**

`meson compile` 是 Meson 构建系统用于**触发实际编译过程**的命令。它会调用底层的构建工具（最常用的是 Ninja）来根据之前 `meson setup` 生成的构建指令（如 `build.ninja` 文件）编译源代码，生成目标文件、库和最终的可执行文件。

你可以把它理解为对传统 `Makefile` 系统中 `make` 命令的直接替代，或者是 CMake 中 `make` 或 `ninja` 命令的替代。

#### 常用选项和参数

| 选项/参数              | 说明                                                         | 示例                                    |
| :--------------------- | :----------------------------------------------------------- | :-------------------------------------- |
| **（无参数）**         | **编译所有默认目标**。这是最常用的形式，编译整个项目。       | `meson compile`                         |
| `-C <dir>`             | **指定构建目录**。不在构建目录内部时使用。                   | `meson compile -C build`                |
| `-j <N>`, `--jobs <N>` | **指定并行编译的作业数**。可以显著加快编译速度。如果不指定，Ninja 会自动选择一个合适的值（通常是 CPU 核心数）。 | `meson compile -j6` （使用6个线程编译） |
| `-v`, `--verbose`      | **详细输出**。显示完整的编译命令，便于调试。                 | `meson compile -v`                      |
| `--clean`              | **清理**。移除所有编译产生的文件（效果类似于 `ninja -t clean`）。 | `meson compile --clean`                 |
| `target`               | **编译指定的目标**。可以是一个可执行文件、一个库等。         |                                         |

#### 示例 1：编译整个项目（最常见用法）

**方法A：先进入构建目录**

bash

```bash
cd my_project/build
meson compile
# 或者使用更短的别名 `mcs` (在某些系统上别名可能不可用)
# mcs
```

**方法B：不进入目录，使用 `-C` 选项**

bash

```bash
cd my_project
meson compile -C build
```

#### 示例 2：并行编译以加快速度

使用你 CPU 的所有核心进行编译：

```bash
meson compile -j$(nproc)   # 在 Linux 上，nproc 命令可以获取核心数
meson compile -j$(sysctl -n hw.ncpu) # 在 macOS 上
```

或者直接指定一个数字：

```bash
meson compile -j8
```

#### 示例 3：只编译特定的目标（如某个测试程序或应用）

假设你的项目定义了一个名为 `unit_tests` 的目标：

```bash
meson compile -C build unit_tests
```

#### 示例 4：清理构建产物

如果你想删除所有编译出来的对象文件和二进制文件，但保留配置（以便下次可以快速重新编译）：

```bash
meson compile -C build --clean
```

#### 示例 5：查看详细的编译命令

当你需要排查为什么某个文件没有正确编译时，详细输出非常有用：

```bash
meson compile -C build -v
```

### 6、meson install

**核心用途：安装构建产物。**

`meson install` 命令负责将编译成功后生成的**目标文件（可执行程序）、库文件、头文件、配置文件、文档等所有资源**从构建目录**复制**到系统上的最终安装位置（例如 `/usr/local` 或你指定的任何前缀）。

你可以把它理解为传统 `Autotools` 系统中 `make install` 命令的直接等价物。

和 `meson compile` 一样，`meson install` 也必须在**已经通过 `meson setup` 初始化并成功编译 (`meson compile`)** 的构建目录中运行。

#### 基本语法

```bash
meson install [options]
```

#### 常用选项

| 选项           | 说明                                                         | 示例                                              |
| :------------- | :----------------------------------------------------------- | :------------------------------------------------ |
| **（无选项）** | **执行安装**。使用 `meson setup` 时配置的 `--prefix` 路径。  | `meson install`                                   |
| `-C <dir>`     | **指定构建目录**。这是最常用的选项之一，允许你在项目根目录直接操作。 | `meson install -C build`                          |
| `--no-rebuild` | **假设构建目录是最新的，直接安装，不重新编译**。如果你确定自上次编译后没有修改任何代码，可以使用此选项来跳过检查，加快安装速度。 | `meson install -C build --no-rebuild`             |
| `--dry-run`    | **模拟安装**。显示**将会**安装哪些文件以及安装到何处，但实际上并不执行任何复制操作。非常适合在真正安装前进行检查。 | `meson install -C build --dry-run`                |
| `--destdir`    | **指定安装目的地目录**。这是**打包软件时极其重要**的选项。它会在指定的 `--prefix` 路径前再加上 `DESTDIR` 路径。 | `meson install -C build --destdir /tmp/mypackage` |

#### 示例 1：安装到自定义前缀（最常见用法）

假设你在配置时已经将安装路径设置在了你家目录下的某个位置，现在要安装：

**方法A：先进入构建目录**

bash

```
# 首先，配置时设置前缀
meson setup build --prefix=/home/yourusername/.local

# 然后编译
meson compile -C build

# 最后安装
cd build
meson install
```

文件将被安装到 `/home/yourusername/.local/bin`, `/home/yourusername/.local/lib` 等目录。

**方法B：不进入目录，使用 `-C` 选项（推荐）**

```bash
meson install -C build
```

#### 示例 2：安装到系统目录（通常需要 root 权限）

如果你想将软件安装到系统级目录（如 `/usr` 或 `/usr/local`），通常需要使用 `sudo`：

```bash
# 配置（通常默认前缀就是 /usr/local）
meson setup build

# 编译
meson compile -C build

# 安装 - 使用 sudo 获取 root 权限
sudo meson install -C build
```

装到 `/usr/local/bin`, `/usr/local/lib` 等目录。

#### 示例 3：在安装前进行检查（ dry-run ）

这是一个非常好的习惯，可以避免意外覆盖系统文件。

```bash
meson install -C build --dry-run
```

输出会显示类似以下内容：

```text
Installing src/my_awesome_tool to /usr/local/bin
Installing libmyawesome.so.1.2.3 to /usr/local/lib
Installing include/my_awesome.h to /usr/local/include
...
# 它只是显示，并不会真的复制文件。
```

### 7、meson test

**核心用途：运行在 `meson.build` 文件中定义的所有测试用例。**

它的主要功能是：

1. **自动化测试**：自动执行项目中定义的各种测试（单元测试、集成测试、性能测试等）。
2. **结果报告**：收集测试结果，并清晰地报告哪些测试通过、哪些失败、哪些被跳过。
3. **测试调试**：提供丰富的选项来过滤、重复运行测试，并获取详细的输出，帮助开发者快速定位问题。
4. **集成CI/CD**：其退出代码（exit code）和结构化输出可以轻松集成到持续集成/持续部署（CI/CD）流水线中。

和 `meson compile` 一样，`meson test` 也必须在**已经通过 `meson setup` 初始化并成功编译 (`meson compile`)** 的构建目录中运行，因为测试用例本身也是需要编译的可执行文件。

#### 基本语法

```bash
meson test [options] [test_name(s)]
```

#### 常用选项和参数

| 选项/参数                  | 说明                                                         | 示例                                               |
| :------------------------- | :----------------------------------------------------------- | :------------------------------------------------- |
| **（无参数）**             | **运行所有测试**。这是最常用的形式。                         | `meson test`                                       |
| `-C <dir>`                 | **指定构建目录**。                                           | `meson test -C build`                              |
| `test_name`                | **运行一个或多个指定的测试**。可以使用测试名称（在 `meson.build` 中定义的）来过滤。 | `meson test my_unit_test` `meson test test1 test2` |
| `-v`, `--verbose`          | **详细输出**。显示每个测试的完整命令行及其输出。**调试测试失败时必用！** | `meson test -v`                                    |
| `--print-errorlogs`        | **打印失败测试的日志**。如果测试失败，将其输出（`stdout/stderr`）打印到控制台。通常与 `-v` 一起使用。 | `meson test --print-errorlogs`                     |
| `--gdb`                    | **在 GDB 调试器中运行失败的测试**。一旦有测试失败，自动在 GDB 中启动该测试程序。 | `meson test --gdb`                                 |
| `--debug`                  | **使用调试参数运行**。Meson 会将一些调试参数（如 `--verbose`）传递给测试程序本身。 | `meson test --debug`                               |
| `--repeat <N>`             | **重复运行测试 N 次**。常用于检查测试是否存在偶发性故障（Flaky Tests）。 | `meson test --repeat 10`                           |
| `--timeout-multiplier <N>` | **设置超时乘数**。将所有测试的超时时间乘以 N。对于在慢速机器上运行测试很有用。 | `meson test --timeout-multiplier 2`                |
| `--benchmark`              | **运行基准测试**。如果测试被标记为 `is_parallel: false`，则按顺序运行它们，以获得更稳定的基准结果。 | `meson test --benchmark`                           |
| `--suite <suite_name>`     | **运行指定测试套件（Suite）中的测试**。项目可以将测试分组到不同的套件中。 | `meson test --suite unit_tests`                    |
| `-t <N>`, `--timeout <N>`  | **设置默认超时时间（秒）**。覆盖 `meson.build` 中设置的默认超时。 |                                                    |



