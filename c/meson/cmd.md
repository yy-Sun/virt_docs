## Meson 命令

### 编译一个 meson 项目

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

### 完全控制 meson 编译与安装参数

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