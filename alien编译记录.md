根据Alien项目的readme，运行需要如下几步：
- rust
- riscv64-linux-musl-gcc
- git submodule update --init --recursive
- make run

rust环境可以通过项目根目录下的rust-toolchain.toml由rustup自动配置。
riscv64-linux-musl-gcc乍一看不知道是什么，去网上搜了搜，结果被引到了一个github项目
[musl-riscv-toolchain](https://github.com/michaeljclark/musl-riscv-toolchain/)

这个项目可以编译出riscv64-linux-musl-gcc交叉编译器，编译方法很简单，看readme很容易理解。 \
但是在编译前需要下载一些东西，但是当时我的Ubuntu没有设置代理，所以下载总是失败。 \
在~/.bashrc中设置好代理后，重新打开终端，仍然有部分文件无法下载，一看是ftp的。 \
幸好项目中有提到可以手动下载这些文件，然后放到指定目录下。 \
于是我就手动下载好了这些文件，放到指定目录下，然后重新运行编译命令，终于成功编译出了riscv64-linux-musl-gcc交叉编译器。

但是rust那边出现了很奇怪的报错 
```
can't find crate for `core`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not be installed
  = help: consider downloading the target with `rustup target add riscv64gc-unknown-none-elf`
  = help: consider building the standard library from source with `cargo build -Zbuild-std`
```

但是根据rust-toolchain.toml，这个target应该是自动安装的啊。 \
我尝试手动运行`rustup target add riscv64gc-unknown-none-elf`，结果提示已经安装了。 

与陈林峰学长共同排查后，发现是因为下载过程中被打断了，导致安装不完整。 \
通过rustup重新安装这个target后，问题解决。 \
然后学长发现，我安装的musl工具链中并没有包含riscv64-linux-musl-cc这个可执行文件， \
学长通过调整使用riscv64-linux-musl-gcc解决了这个问题。 

之后陈林峰学长又展示了busybox的构建过程，然后成功运行了alien项目。 

过程中，学长提到使用的musl是一个预编译的musl工具链。在理解musl是一个c标准库实现后， \
我转去搜索musl，然后找到了https://musl.cc/#binaries，应该就是学长提到的预编译的musl工具链网站。 \
直接下载riscv64-linux-musl-cross的预编译工具链后，解压到某个目录下，然后添加PATH环境变量之后就可以使用riscv64-linux-musl-cc了