---
title: libvirt 编译与安装
date: 2024-04-02 22:24:27
tags: [libvirt]
categories:
- 虚拟化
- libvirt
---

可以参考 libvirt 的官方文档，这里以编译 libvirt 10.2.0 为例
https://libvirt.org/compiling.html

## configure

### 替换 .gitmodules

libvirt 的 .gitmodules使用的 gitlab, 国内访问 gitlab 较慢，可以替换成 github

修改 `.gitmodules`, 将 `url = https://gitlab.com/keycodemap/keycodemapdb.git` 替换为 `url = https://github.com/qemu/keycodemapdb.git`

### 安装依赖

```bash
yum install -y libxslt gnutls-devel libtirpc-devel
```

### 查看配置列表

```bash
meson configure
```

### configure

```bash
meson setup build -Dsystem=true \
  -Ddriver_qemu=enabled \
  -Dyajl=enabled \
  -Ddriver_remote=enabled \
  -Ddriver_vmware=disabled \
  -Ddocs=disabled \
  -Ddriver_lxc=enabled \
  -Ddriver_libvirtd=enabled \
  -Ddriver_vbox=disabled \
  -Dselinux=disabled
```

{% note primary %} 

构建时可能会报错相关依赖找不到，缺少什么安装相应的库即可

另外，有些是相互依赖的，例如开启了 `driver_qemu` 则必须同时开启 `driver_libvirtd`，否则会报错并提示。

{% endnote %}

![libvirt_compile_1](https://docs-1259157567.cos.ap-nanjing.myqcloud.com/docs/libvirt_compile_1.png)

## 编译

```bash
ninja -C build
```

## 安装

```bash
ninja -C build install
```

### 启动 libvirt

```bash
 ./libvirtd --listen
2024-03-30 20:47:09.812+0000: 204087: info : libvirt version: 10.2.0
2024-03-30 20:47:09.812+0000: 204087: info : hostname: 192.168.182.131
2024-03-30 20:47:09.812+0000: 204087: error : 
virNetTLSContextCheckCertFile:105 : Cannot read CA certificate '/etc/pki/CA/cacert.pem':
No such file or directory
```

{% note danger %} 

libvirt 默认开启 tls 加密连接，需要配置服务端证书，如果不想开启 tls，可以修改配置文件

{% endnote %}

- 修改配置文件 `/etc/libvirt/libvirtd.conf` ，将以下两个注释打开

```conf
listen_tls = 0

listen_tcp = 1
```
### 防火墙

建议把 防火墙也关闭了。

```bash
systemctl stop firewalld
systemctl disable firewalld
```

