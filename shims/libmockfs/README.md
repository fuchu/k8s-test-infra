# libmockfs

`libmockfs.so` 是一个 `LD_PRELOAD` 垫片，把对 PCI sysfs 和内核模块的访问重定向到一个假的目录树。

可用它来测试那些寻找 PCI 设备或检查某个内核模块是否已加载的软件。这类软件于是既不需要真实的 PCI 拓扑，也不需要真实的模块集合。

一个垫片同时服务两类访问面。两者渲染在同一根目录下，重定向规则就是一张单一的前缀表。若再加一个垫片，就要重复这套 libc 拦截逻辑，只为新增两项。

根目录变量名固定为 `MOCK_PCI_ROOT`。NRI 插件会在它服务的每个工作负载容器里设置它，chart 文档也提到了它，因此它是一个「容器可见的契约」。改名是一项独立的改动。

## 工作原理

当 `MOCK_PCI_ROOT` 被设置时，以下位置的访问：

```text
/sys/bus/pci
/sys/bus/pci/devices
/sys/devices/pci*
/sys/module
/proc/modules
/proc/driver/nvidia
```

会在前面拼上 `MOCK_PCI_ROOT` 进行重写。

该垫片重定向 `/proc/modules` 和 `/proc/driver/nvidia`，而不是把它们挂载进去。runc 拒绝任何 `/proc` 内、不在其白名单中的 bind mount。因此 `lsmod`，以及 Slurm 的 `AutoDetect=nvidia` GPU 发现，都依赖这个垫片。

例如：

```text
/sys/bus/pci/devices/0000:07:00.0/config
```

配合：

```bash
MOCK_PCI_ROOT=/tmp/mock-pci
```

会变为：

```text
/tmp/mock-pci/sys/bus/pci/devices/0000:07:00.0/config
```

所有不相关的路径原样放行。

该库拦截常见的 libc 文件系统 API，包括：

* `open`、`open64`、`openat`、`openat64`
* 加固版的 `__open*_2` 变体
* `fopen`、`fopen64`
* `opendir`
* `stat`、`lstat`、`fstatat`、`statx`
* `access`、`faccessat`
* `readlink`、`readlinkat`

真正的 libc 实现通过 `dlsym(RTLD_NEXT, ...)` 调用。

## 用法

创建一个镜像预期 PCI sysfs 布局的假目录树：

```text
/tmp/mock-pci/
└── sys/
    └── bus/
        └── pci/
            └── devices/
                └── 0000:07:00.0/
                    ├── config
                    ├── resource
                    └── vendor
```

然后以预加载垫片的方式运行目标进程：

```bash
MOCK_PCI_ROOT=/tmp/mock-pci \
LD_PRELOAD=/path/to/libmockfs.so \
lspci
```

若 `MOCK_PCI_ROOT` 未设置，该库就是个空操作（no-op）。

## 作用范围

这是用户态的 libc 拦截。它不会修改、挂载或虚拟化真实的 `/sys` 文件系统，只影响以预加载方式启动的进程。
