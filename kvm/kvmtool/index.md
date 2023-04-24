# 概述
Native Linux KVM tool
=====================

kvmtool is a lightweight tool for hosting KVM guests. As a pure virtualization
tool it only supports guests using the same architecture, though it supports
running 32-bit guests on those 64-bit architectures that allow this.

From the original announcement email:
-------------------------------------------------------
The goal of this tool is to provide a clean, from-scratch, lightweight
KVM host tool implementation that can boot Linux guest images (just a
hobby, won't be big and professional like QEMU) with no BIOS
dependencies and with only the minimal amount of legacy device
emulation.

It's great as a learning tool if you want to get your feet wet in
virtualization land: it's only 5 KLOC of clean C code that can already
boot a guest Linux image.

Right now it can boot a Linux image and provide you output via a serial
console, over the host terminal, i.e. you can use it to boot a guest
Linux image in a terminal or over ssh and log into the guest without
much guest or host side setup work needed.
--------------------------

> 总而言之，kvmtool 是基于 kvm 技术实现的虚拟化工具，他没有 qemu 那么专业（代码只有5k，更容易了解虚拟化的本质），但是
可以完整的启动一个虚拟机，可以供我们来学习 kvm 相关接口。

kvm 项目地址如下：
https://github.com/kvmtool/kvmtool
