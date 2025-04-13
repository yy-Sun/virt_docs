## configure

### 更新 submodules

`qemu` 的 `submodule` 默认配置的都是 `gitlab` 地址，国内访问 `gitlab` 较慢，一般都可以替换成同源的 `github`

- 替换 .gitmodules

将代码仓 url 中 `gitlab.com/qemu-project` 替换为 `github/qemu`

- 更新 `subprojects/` 下面的 `*.wrap` 文件

同样将代码仓 url 中将 `gitlab.com/qemu-project` 替换为 `github/qemu`

```bash
$ git diff subprojects/berkeley-softfloat-3.wrap
diff --git a/subprojects/berkeley-softfloat-3.wrap b/subprojects/berkeley-softfloat-3.wrap
index c3e356d42f..2e136e527d 100644
--- a/subprojects/berkeley-softfloat-3.wrap
+++ b/subprojects/berkeley-softfloat-3.wrap
@@ -1,5 +1,5 @@
 [wrap-git]
-url = https://gitlab.com/qemu-project/berkeley-softfloat-3.git
+url = https://github.com/qemu/berkeley-softfloat-3.git
 revision = b64af41c3276f97f0e181920400ee056b9c88037
 patch_directory = berkeley-softfloat-3
 depth = 1
```

### configrue 配置项目

`./configure --help` 查看配置文件

这里以 qemu 9.0 为例：

```bash
./configure  --target-list="x86_64-softmmu" \
  --enable-kvm \
  --disable-gcov \
  --disable-tsan \
  --with-coroutine=ucontext \
  --without-default-features \
  --enable-vmdk \
  --enable-linux-aio \
  --enable-linux-io-uring \
  --enable-debug
```

后续需要用到什么特性，再开启对应的特性。

> 如果需要调试，可以添加 --enable-debug， 这样会将优化设置为 -o0

## 编译

```bash
make -j
```

## 测试

可选项，一般懒得跑

```bash
make check
```

## 安装

```bash
make install
```