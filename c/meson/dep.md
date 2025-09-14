## Dependencies

https://mesonbuild.com/Dependencies.html

很少有应用程序是完全独立的，而是使用外部库和框架来完成工作。Meson 让查找和使用外部依赖项变得非常容易。以下是使用 `zlib` 压缩库的示例。

```bash
zdep = dependency('zlib', version : '>=1.2.8')
exe = executable('zlibprog', 'prog.c', dependencies : zdep)
```

首先，Meson 会被告知查找外部库 zlib，如果找不到则会报错。version 关键字是可选的，用于指定依赖项的版本要求。然后，系统会使用指定的依赖项构建可执行文件。请注意，用户无需手动处理编译器或链接器标志，也无需处理任何其他细节。

如果您有多个依赖项，请将它们作为数组传递：

```bash
executable('manydeps', 'file.c', dependencies : [dep1, dep2, dep3, dep4])
```

如果依赖项是可选的，您可以告诉 Meson 在找不到依赖项时不要出错，然后进行进一步的配置。

```bash
opt_dep = dependency('somedep', required : false)
if opt_dep.found()
  # Do something.
else
  # Do something else.
endif
```

无论实际依赖项是否找到，您都可以将 opt_dep 变量传递给目标构造函数。Meson 将忽略未找到的依赖项。

Meson 还允许获取 pkg-config 文件中定义的变量。这可以通过使用 `dep.get_pkgconfig_variable()`函数来实现。

```bash
zdep_prefix = zdep.get_pkgconfig_variable('prefix')
```

### 1、以通过多种方式找到依赖项中的任意变量

当您需要从依赖项中获取任意变量（该依赖项有多种查找方式，且您不想限制其类型）时，可以使用通用的 get_variable 方法。该方法目前支持基于 cmake、pkg-config 和 config-tool 的变量。

```bash
foo_dep = dependency('foo')
var = foo_dep.get_variable(cmake : 'CMAKE_VAR', pkgconfig : 'pkg-config-var', configtool : 'get-var', default_value : 'default')
```

### 2、依赖性检测方法

> 依赖项检测器适用于所有提供 pkg-config 文件的库。遗憾的是，有些软件包不提供 pkg-config 文件。Meson 对其中一些库提供了自动检测支持.

您可以使用关键字 `method` 来告知 `Meson` 在搜索依赖项时应使用的方法。默认值为 `auto`。其他方法包括 `pkg-config、config-tool、cmake、builtin、system、sysconfig、qmake、extraframework` 和 `dub`

