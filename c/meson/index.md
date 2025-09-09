Meson（The Meson Build System）是个项目构建系统，类似的构建系统有 `Makefile`、`CMake`、`automake` …。 Meson 是一个由 Python 实现的开源项目，其思想是，开发人员花费在构建调试上的每一秒都是浪费，同样等待构建过程直到真正开始编译都是不值得的。

因此，Meson 的设计目的是在用户友好的同时不损害性能，Meson 提供客户语言（custom language）作为主要工具，用户可以使用它完成项目构建的描述。客户语言的设计目标是简单（simplicity）、清晰（clarity）、简洁（conciseness），其中很多灵感来源于 Python 语言。

Meson 的另个一主要设计目的是为现代编程工具提供优秀的支持和最好的实现。这包括一些特性如：单元测试（unit testing）、代码覆盖率报告（code coverage reporting）、头文件预编译（precompiled headers）。用户不需要寻找第三方宏指令（third party macros）或编写 Shell 脚本来实现这些特性，Meson 可以开箱即用。Meson 相比 CMake 来说，不仅仅支持 C/C++，还支持多种编程语言。

> 如今，很多项目都由 CMake 转向到了 Meson，例如 `DPDK` 和 `Mapnik`。

### Ninja 的简介

项目开发中一般将 Meson 和 Ninja 配合使用，Meson 负责构建项目依赖关系，Ninja 负责编译代码。Ninja 是一个轻量的构建系统，主要关注构建的速度。它与其他构建系统的区别主要在于两个方面：一是 Ninja 被设计成需要一个输入文件的形式，这个输入文件则由高级别的构建系统生成；二是 Ninja 被设计成尽可能快速执行构建的工具。

### Meson 的特性

- 支持多种平台，包括 Linux、macOS、Windows、GCC、Clang、Visual Studio 等
- 支持多种编程语言，包括 C/C++、D、Fortran、Java、Rust
- 支持在一个非常可读和用户友好的非图灵完整 DSL 中构建定义
- 支持很多操作系统和裸机进行交叉编译
- 支持极快的完整和增量构建而优化，而不牺牲正确性
- 支持与发行版包一起工作的内置多平台依赖提供程序

### Meson 的依赖

Meson 是依赖 Python 与 Ninja 实现的，依赖的版本如下：

- Python (version 3.6 or newer)
- Ninja (version 1.8.2 or newer)

## Meson 安装

### Windows 平台

- a）在 [Meson GitHub Releases](https://github.com/mesonbuild/meson/releases) 网站下载 Windows 版的安装程序，如 `meson-0.60.3-64.msi`

- b）双击 `meson-0.60.3-64.msi` 安装程序，按默认选项直接安装 Meson

- c）在系统的 `开始菜单栏` 里，找到 Visual Studio 开发人员工具（

  ```sh
  Native Tools Command Prompt for VS xxxx
  ```

  ），双击运行后，在 CMD 窗口内执行以下命令查看 Meson 和 Ninja 的版本

  ```sh
  > meson --version
  0.60.3
  
  > ninja --version
  1.10.2
  ```

### Debian/Ubuntu

```sh
# apt install -y meson ninja-build
```

### Fedora/CentOS

```sh
# yum install -y meson ninja-build

# 或者

# dnf install -y meson ninja-build
```

### 通过 PyPi 安装

Meson 可以直接通过 [PyPi](https://pypi.python.org/pypi/meson) 安装，但必须确保使用的是 Python3 的 `pip`，安装命令如下：

```sh
# pip3 install meson ninja
```

或者使用标准的 Python 命令安装 Meson

```sh
# 安装meson
# python3 -m pip install meson

# 安装ninja
# python3 -m pip install ninja
```

## Meson 运行

注意

若使用的是 Windows 平台，则需要在 Visual Studio 开发人员工具（`Native Tools Command Prompt for VS xxxx`）里执行 Meson 的命令，这是因为 C/C++ 编译器只会在该工具上运行。

- 通过 Mesonn 初始化新的 C/C++ 项目，并使用 Meson 构建项目

```sh
# 创建一个新目录来保存项目文件
$ mkdir meson_project

# 进入项目目录
$ cd meson_project

# 使用Meson初始化并构建一个新的C/C++项目，会自动生成"meson.build"配置文件和C/C++源文件
$ meson init --name meson_project --build

# 项目构建完成后，默认的构建目录是build，可以直接运行构建生成的可执行文件
$ build/meson_project
```

- 当项目代码发生变更后，可以进入 `build` 目录重新构建代码

```sh
# 进入build目录
$ cd build

# 重新构建代码
$ meson compile
```

- Meson 项目的顶层目录结构如下

```sh
meson_project
├── build               # Meson的构建目录
├── meson.build         # Meson的配置文件
└── meson_project.c     # C/C++源文件
```

## Meson 指定编译参数

通过 `meson configure` 命令可以查看 Meson 内置的编译参数、默认值以及可选值

```sh
# 进入Meson项目的根目录
$ cd meson_project

# 查看Meson的编译参数
$ meson configure
```

Meson 项目可以通过 `meson_options.txt` 配置文件来增加项目特有的编译参数，如：

```text
option('tests', type: 'boolean', value: true,
	description: 'build unit tests')
option('use_hpet', type: 'boolean', value: false,
	description: 'use HPET timer in EAL')
```

Meson 还支持在生成项目编译配置时，通过 `-D` 指定编译参数

```sh
# 进入Meson项目的根目录
$ cd meson_project

# 指定编译参数，生成输出目录
$ meson build -Dprefix=/usr -Dtests=disabled

# 进入输出目录
$ cd build

# 编译代码
$ ninja -j8
```