# nvidia-imex-shim

由节点代理在 Mokka 节点本地驱动目录里、紧挨真正的 IMEX daemon 放置的一个 `execve` 包装器。

上游 daemon 把 `nvidia-imex -c /imexd/imexd.cfg` 写死，没有任何命令行透传，因此 `--nogpu` 无法从外部注入。这个垫片会找到与自己同目录的 `nvidia-imex.real`，并在其后追加 `--nogpu`，同时保留调用方的全部参数、环境变量和 stdio。这次 `exec` 会直接替换垫片进程——没有残留的包装进程，信号也能直达 daemon。

一旦上游支持命令行透传即可移除。见 NVIDIA/k8s-test-infra#304。
